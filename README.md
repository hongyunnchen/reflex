# ReFlex

ReFlex is a software-based system that provides remote access to Flash with performance nearly identical to local Flash access. ReFlex closely integrates network and storage processing in a dataplane  kernel to achieve low latency and high throughput at low resource requirements. ReFlex uses a novel I/O scheduler to enforce tail latency and throughput service-level objectives (SLOs) for multiple clients sharing a remote Flash device.

## Requirements

ReFlex requires a NVMe Flash device and a [network interface card supported by Intel DPDK](http://dpdk.org/doc/nics). We have tested ReFlex with the Intel 82599 10GbE NIC. The driver is provided in `dp/drivers/ixgbe.c`. We have tested ReFlex with the following NVMe SSDs: Samsung PM1725 and Intel P3600.  

ReFlex is an extension of the [IX dataplane operating system](https://github.com/ix-project/ix) and thus inherits the dependencies described  on the [IX requirements page](https://github.com/ix-project/ix/wiki/Requirements).  In particular, ReFlex currently relies on [Dune](http://dune.scs.stanford.edu)  to provide direct access to hardware (such as NIC and NVMe queues) from userspace. ReFlex uses Intel [DPDK](http://dpdk.org) for fast packet processing and Intel  [SPDK](http://www.spdk.io) for high performance access to NVMe Flash. Currently, ReFlex has been successfully tested on Ubuntu 16.04 LTS with kernel 4.4.0.

**Note:** ReFlex provides an efficient *dataplane* for remote access to Flash. To deploy ReFlex in a datacenter cluster, ReFlex should be combined with a control plane to manage Flash resources across machines and optimize the allocation of Flash IOPS and capacity.  

## Setup Instructions

There is currently no binary distribution of ReFlex. You will therefore need  to compile the project from source, as described below. 

1. Obtain ReFlex source code and fetch dependencies:

   ```
   git clone https://github.com/stanford-mast/reflex.git
   cd reflex
   ./deps/fetch-deps.sh
   ```

2. Install library dependencies: 

   ```
   sudo apt-get install libconfig-dev libnuma-dev libpciaccess-dev libaio-dev libevent-dev
   ```

3. Build the dependecies:

   ```
   sudo chmod +r /boot/System.map-`uname -r`
   make -sj64 -C deps/dune
   make -sj64 -C deps/pcidma
   make -sj64 -C deps/dpdk config T=x86_64-native-linuxapp-gcc
   make -sj64 -C deps/dpdk
   make -sj64 -C deps/spdk config.h
   make -sj64 -C deps/spdk/lib/nvme/ CONFIG_NVME_IMPL="../../../../inc/ix/spdk.h -I ../../../dpdk/build/include/ -I ../../../../inc/ -D__KERNEL__"
   make -sj64 -C deps/spdk/lib/util/ CONFIG_NVME_IMPL="../../../../inc/ix/spdk.h -I ../../../../inc/ -D__KERNEL__"
   ```

4. Build ReFlex:

   ```
   make -sj64
   ```
5. Set up the environment:

   ```
   cp ix.conf.sample ix.conf
    # modify at least host_addr, gateway_addr, devices, and nvme_devices
   sudo sh -c 'for i in /sys/devices/system/node/node*/hugepages/hugepages-2048kB/nr_hugepages; do echo 4096 > $i; done'
   sudo modprobe -r ixgbe
   sudo modprobe -r nvme
   sudo insmod deps/dune/kern/dune.ko
   sudo insmod deps/pcidma/pcidma.ko
   ```
   
6. (Optional) Run IX TCP echo server to check that your setup is correct:

   ```
   sudo ./dp/ix -- ./apps/echoserver 4
   ```

   Then, try from another Linux host:

   ```
   echo 123 | nc -vv <IP> <PORT>
   ```
   You should see the following output: 

   ```
   Connection to <IP> <PORT> port [tcp/*] succeeded!
   123
   ```

7. Precondition the SSD and derive the request cost model:

   It is necessary to precondition the SSD to acheive steady-state performance and reproducible results across runs. We recommend preconditioning the SSD with the following local Flash tests: write *sequentially* to the entire address space using 128KB requests, then write *randomly* to the device with 4KB requests until you see performance reach steady state. The random write test typically takes about 10 to 20 minutes, depending on the device model and capacity. 
   
   After preconditioning, you should derive the request cost model for your SSD:

   ```
   cp sample.devmodel nvme_devname.devmodel
    # update request costs and token limits to match your SSD performance 
    # see instructions in comments of file sample.devmodel
	# in ix.conf file, update nvme_device_model=nvme_devname.devmodel
   ```
   You may use any I/O load generation tool (e.g. [fio](https://github.com/axboe/fio)) for preconditioning and request calibration tests. Note that if you use a Linux-based tool, you will need to reload the nvme kernel module for these tests (remember to unload it before running the ReFlex server). 
  
   For your convenience, we provide an open-loop, local Flash load generator based on the SPDK perf example application [here](https://github.com/anakli/spdk_perf). We modified the SPDK perf example application to report read and write percentile latencies. We also made the load generator open-loop, so you can sweep throughput by specifying a target IOPS instead of queue depth. See setup instructions for ReFlex users in the repository's [README](https://github.com/anakli/spdk_perf/blob/master/README.md). 


## Running ReFlex 

### 1. Run the ReFlex server:

   ```
   sudo ./dp/ix -- ./apps/reflex_server
   ```

   ReFlex runs one dataplane thread per CPU core. If you want to run multiple ReFlex threads (to support higher throughput), set the `cpu` list in ix.conf and add `fdir` rules to steer traffic identified by {dest IP, src IP, dest port} to a particular core.

#### Registering service level objectives (SLOs) for ReFlex tenants:

* A *tenant* is a logical abstraction for accounting for and enforcing SLOs. ReFlex supports two types of tenants: latency-critical (LC) and best-effort (BE) tenants. 
* The current implementation of ReFlex requires tenant SLOs to be specified statically (before running ReFlex) in `pp_accept()` in `apps/reflex_server.c`. Each port ReFlex listens on can be associated with a separate SLO. The tenant should communicate with ReFlex using the destination port that corresponds to the appropriate SLO. This is a temporary implementation until there is proper client API support for a tenant to dynamically register SLOs with ReFlex. 

 > As future work, a more elegant approach would be to i) implement a ReFlex control plane that listens on a dedicated admin port, ii) provide a client API for a tenant to register with ReFlex on this admin port and specify its SLO, and iii) provide a response from the ReFlex control plane to the tenant, indicating which port the tenant should use to communicate with the ReFlex data plane.

### 2. Run a ReFlex client:

There are several options for clients:

1. Run a Linux-based client that opens TCP connections to ReFlex and sends read/write reqs to logical blocks. 

	* This client option avoids the performance overheads of the filesystem  and block layers in the client machine. However, the client is still subject to any latency or throughput inefficiencies of the networking layer in the  Linux operating system.

	On the client machine, setup the `mutilate` load generator. We provide a modified version of `mutilate` which supports the ReFlex block protocol.

	```
	git clone https://github.com/anakli/mutilate.git
	cd mutilate
	sudo apt-get install scons libevent-dev gengetopt libzmq-dev
	scons
	```

	To achieve the best performance with a Linux client, we recommend tuning NIC settings and disabling CPU and PCIe link power management on the client. 
	
	Network setup tuning (recommended):

	```
	 # enable jumbo frames
	sudo ifconfig ethX mtu 9000
	 # tune interrupt coalescing 
	sudo ethtool -C ethX rx-usecs 50
	 # disable LRO/GRO offloads 
	sudo ethtool -K ethX lro off
	sudo ethtool -K ethX gro off
	```

	Disable power mangement (recommended):

   	```
	## to disable CPU power management, run:
	#!/bin/bash
	for i in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do
    	echo performance > $i
	done

	## to disable PCIe link power management:
	sudo vi /etc/default/grub 
     # modify existing line:
     # GRUB_CMDLINE_LINUX_DEFAULT="quiet splash pcie_aspm=force processor.max_cstate=1 idle=poll"
   	sudo grub-mkconfig -o /boot/grub/grub.cfg
   	sudo reboot
   	sudo su
   	echo performance > /sys/module/pcie_aspm/parameters/policy
   	```

	Run Linux client:

	```
	./mutilate -s IP:PORT --noload -r NUM_LOGICAL_BLOCK_ADDRESSES -K 8 -V 4096 --binary
	```

	This command is for an unloaded latency test; it launches a single client thread issuing requests with queue depth 1. Sample expected output:
	
	```
    #type       avg     std     min     5th    10th    90th    95th    99th     max
	read      115.2     8.7   101.8   107.3   108.1   125.8   127.6   129.1   351.3
	update      0.0     0.0     0.0     0.0     0.0     0.0     0.0     0.0     0.0
	op_q        1.0     0.0     1.0     1.0     1.0     1.1     1.1     1.1     1.0

	Total QPS = 8662.4 (519747 / 60.0s)
	```	
	
	Follow instructions in [mutilate README](https://github.com/anakli/mutilate/blob/master/README.md) to launch multiple client threads across multiple agent machines for high-throughput tests.


2. Run an IX-based client that opens TCP connections to ReFlex and sends read/write requests to logical blocks.

	* This client option avoids networking and storage (filesystem and block layer) overhead on the client machine. It requires the client machine to run IX.
	
   Clone ReFlex source code on client machine and follow steps 1 to 6 in the setup instructions. Comment out `nvme_devices` in ix.conf. Run IX-based ReFlex client:

   ```
   sudo ./dp/ix -- ./apps/reflex_ix_client IP PORT SEQUENTIAL? NUM_THREADS REQ/s READ% SWEEP? REQ_SIZE PRECONDITION? 
    # example: sudo ./dp/ix -- ./apps/reflex_ix_client 198.168.40.1 1234 0 1 200000 100 1 4096 0 
   ```

   Sample output:

   ```
   RqIOPS:	 IOPS: 	 Avg:  	 10th: 	 20th: 	 30th: 	 40th: 	 50th: 	 60th: 	 70th: 	 80th: 	 90th: 	 95th: 	 99th: 	 max:  	 missed:
   1000   	 1000  	 98    	 92    	 93    	 94    	 94    	 95    	 96    	 110   	 111   	 112   	 113   	 117   	 149   	 0
   10000  	 10000 	 98    	 92    	 92    	 93    	 94    	 94    	 95    	 110   	 111   	 112   	 112   	 113   	 172   	 9
   50000  	 49999 	 100   	 92    	 93    	 94    	 95    	 95    	 96    	 111   	 111   	 113   	 114   	 134   	 218   	 682
   100000 	 99999 	 102   	 93    	 94    	 95    	 96    	 97    	 104   	 112   	 113   	 116   	 122   	 155   	 282   	 44385
   150000 	 150001	 105   	 94    	 95    	 96    	 98    	 100   	 109   	 113   	 115   	 120   	 132   	 168   	 357   	 303583
   200000 	 199997	 109   	 96    	 97    	 99    	 101   	 105   	 112   	 115   	 118   	 127   	 145   	 178   	 440   	 923940
   ```

   For high-throughput tests, increase the number of IX client threads (and CPU cores configured in client ix.conf). You may also want to run multiple ReFlex threads on the server.


3.  Run a legacy client application using the ReFlex remote block device driver.

	* This client option is provided to support legacy applications. ReFlex exposes a standard Linux remote block device interface to the client (which appears to the client as a local block device). The client can mount a filesystem on the block device. With this approach, the client is subject to overheads in the Linux filesystem, block storage layer and network stack.

    On the client, change into the reflex_nbd directory and type make. Be sure that the remote ReFlex server is running and that it is ping-able from the client. Load the reflex.ko module, type dmesg to check whether the driver was successfully loaded. To reproduce the fio results from the paper do the following.
    ```
    cd reflex_nbd
    make

	sudo modprobe ixgbe
	sudo modprobe nvme
	 # make sure you have started reflex_server on the server machine and can ping it
    
	sudo insmod reflex.ko
    sudo mkfs.ext4 /dev/reflex0
    sudo mkdir /mnt/reflex
    sudo mount /dev/reflex0 /mnt/reflex
	sudo chmod a+rw /mnt/reflex
    touch /mnt/reflex/fio
    sudo apt install fio
    for i in 1 2 4 8 16 32 ; do BLKSIZE=4k DEPTH=$i fio randread_remote.fio; done
    ```
## Reference

Please refer to the ReFlex [paper](https://web.stanford.edu/group/mast/cgi-bin/drupal/system/files/reflex_asplos17.pdf):

Ana Klimovic, Heiner Litz, Christos Kozyrakis
ReFlex: Remote Flash == Local Flash
in the 22nd International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS'22), 2017

## License and Copyright

ReFlex is free software. You can redistribute and/or modify the code under the terms of the BSD License. See [LICENSE](https://github.com/stanford-mast/reflex/blob/master/LICENSE). 

The original ReFlex code was written collaboratively by Ana Klimovic and Heiner Litz at Stanford University. Per Stanford University policy, the copyright of the original code remains with the Board of Trustees of Leland Stanford Junior University.

## Future work

We are actively working on a userspace-only implementation of ReFlex which does not use Dune. This implementation will simplify ReFlex deployment since the only requirements for running ReFlex will be a kernel and network/Flash hardware that support DPDK and SPDK.

The current implementation of ReFlex would also benefit from the following:

* Client API support for tenants to dynamically register SLOs. 
* Support for multiple NVMe device management by a single ReFlex server.
* Support for light weight network protocol such as UDP (currently use TCP/IP).

