
```

 ______                _
(_____ \              | |
 _____) )__  ____   __| |
|  ____/ _ \|  _ \ / _  |
| |   | |_| | | | ( (_| |
|_|    \___/|_| |_|\____|  - Compute Express Link (CXL) based Memory Pooling Systems

```
---
---
---


# README for CXL-emulation and Experiments HowTo #

### Cite our Pond paper (ASPLOS '23): 

>> Pond: CXL-Based Memory Pooling Systems for Cloud Platforms

**The preprint of the paper can be found [here](https://huaicheng.github.io/p/asplos23-pond.pdf).**

```
@InProceedings{pond.asplos23,
  author = {Huaicheng Li and Daniel S. Berger and Lisa Hsu and Daniel Ernst and Pantea Zardoshti and Stanko Novakovic and Monish Shah and Samir Rajadnya and Scott Lee and Ishwar Agarwal and Mark D. Hill and Marcus Fontoura and Ricardo Bianchini},
  title = "{Pond: CXL-Based Memory Pooling Systems for Cloud Platforms}",
  booktitle = {Proceedings of the 28th ACM International Conference on Architectural Support for Programming Languages and Operating Systems (ASPLOS)},
  address = {Vancouver, BC Canada},
  month = {March},
  year = {2023}
}

```

### What is this repo about?

This repo open-sources our approach for CXL simulation and evaluations (mainly
some scripts). It is not a full-fledged open-source version of Pond design.


### CXL Emulation on regular 2-socket (2S) server systems ###

We mainly emulate the following two characteristics of Compute Express Link
(CXL) attached DRAM:

- Latency: ~150ns
- No local CPU which can directly accesses it, i.e., CXL-memory treated as a
  "computeless/cpuless" node

The CXL latency is similar to the latency of one NUMA hop on modern 2S systems,
thus, we simulate the CXL memory using the memory on the remote NUMA node and
disables all the cores on the remote NUMA node to simulate the "computeless"
node behavior.

In this repo, the CXL-simulation is mainly done via a few scripts, check
``cxl-global.sh`` and the ``run-xxx.sh`` under each workload folder (e.g.,
``cpu2017``, ``gapbs``, etc.).

These scripts dynamically adjust the system configuration to simulate a certain
percentage of CXL memory in the system, and run workloads against such
scenarios.


### Configuring Local/CXL-DRAM Splits ###

One major setup we are benchmarking is to adjust the perentage of CXL-memory
being used to run a certain workload and observe the performance impacts
(compared to pure local DRAM "ideal" cases). For example, the common ratios the
scripts include "100/0" (the 100% local DRAM case, no CXL), "95/5", "90/10"
(90% local DRAM + 10% CXL), "85/15", "80/20", "75/25", "50/50", "25/75", etc. 

To provision correct amount of local/CXL memory for the above split ratios, we
need to profile the peak memory usage of the target workload. This is usually
done via monitoring tools such as ``pidstat`` (the ``RSS`` field reported in
the memory usage).


### A Simple HowTo using SPEC CPU 2017 ###

Under folder ``cpu2017``, ``run-cpu2017.sh`` is the main entry to run a series
of experiments under various split configurations. The script reads the
profiled information in a workload input file (e.g., ``wi.txt``), where the
first column is the workload name and the second column is the peak memory
consumption of the workload. Based on these, ``run-cpu2017.sh`` will iterate
over a series of predefined split-ratios and run the experiments one by one.
The scripts writes the logs and outputs to ``rst`` folder.

One could co-run profiling utilities such as emon or Intel Vtune
together with the workload to collect architecture-level metrics for
performance analysis. Make sure Intel Vtune is installed first before running
the script.

##### Ubuntu VM on Ubuntu Server with qemu

- SSH into your Ubuntu server and also make sure that you have installed qemu on your local machine

- First, install QEMU (see https://wiki.qemu.org/Hosts/Linux for other options):
	  ```bash
      sudo apt-get install qemu-system-x86
      # I have build qemu-7.2.0 using source code
      # Some prerequisite packets are required (details can be found https://wiki.qemu.org/Hosts/Linux here)
      sudo apt-get install git libglib2.0-dev libfdt-dev libpixman-1-dev zlib1g-dev ninja-build
      sudo apt-get install libibverbs-dev libjpeg8-dev libncurses5-dev libnuma-dev libaio-dev libslirp-dev libaio1
      # To download and build QEMU 7.2.0:
      wget https://download.qemu.org/qemu-7.2.0.tar.xz
      tar xvJf qemu-7.2.0.tar.xz
      cd qemu-7.2.0
      cd build
      ../configure --prefix=/opt/qemu-7
      # I want to install it under /opt/qemu-7 so it doesn’t interfere with any existing or future QEMU installation.
      # Alternatively, don't run 'make install' and source the binaries and libraries from the build directory.
      make -j all
      make install
    ```
  
- Configure Host Networking
	- Confirm IP forwarding is enabled for IPv4 and/or IPv6 on the host (0=Disabled, 1=Enabled):
	
	  ```bash
	  $ sudo cat /proc/sys/net/ipv4/ip_forward
	  1
	  $ sudo cat /proc/sys/net/ipv6/conf/default/forwarding
	  1
	  # If necessary, activate forwarding temporarily until the next reboot:
	  $ sudo echo 1 > /proc/sys/net/ipv4/ip_forward
	  
	  $ sudo echo 1 > /proc/sys/net/ipv6/conf/all/forwarding
	  ```
	
	- For a permanent setup create the following file:
		```bash
        $ sudo vim /etc/sysctl.d/50-enable-forwarding.conf

        # local customizations
        #
        # enable forwarding for dual stack
        net.ipv4.ip_forwarding=1 net.ipv6.conf.all.forwarding=1
        ```
  
- Download an Ubuntu ISO file (or any other linux):
	```bash
	mkdir -p ~/images/
    cd ~/images
	wget curl -LO https://releases.ubuntu.com/22.04/ubuntu-22.04.2-live-server-amd64.iso
	# create the qcow2 file that will serve as the hard drive for the VM:
	qemu-img create -f qcow2 pondcxl.qcow2 40G

- Next, run this command to boot the VM (I put this in a script ./images/live-server-setup.sh):
	```bash
	  sudo /opt/qemu-7/bin/qemu-system-x86_64 -name PondVM \
    -machine type=pc,accel=kvm,mem-merge=off -enable-kvm \
    -cpu host -smp cpus=8 -m 16384M \
    -object memory-backend-ram,size=12288M,policy=bind,host-nodes=0,id=ram-node0,prealloc=on,prealloc-threads=8 \
    -numa node,nodeid=0,cpus=0-7,memdev=ram-node0 \
    -object memory-backend-ram,size=4096M,policy=bind,host-nodes=1,id=ram-node1,prealloc=on,prealloc-threads=8 \
    -numa node,nodeid=1,memdev=ram-node1 \
    -device virtio-scsi-pci,id=scsi0 \
    -device scsi-hd,drive=hd0 \
    -drive file=./images/pondcxl.qcow2,if=none,aio=native,cache=none,format=qcow2,id=hd0 \
    -net user,hostfwd=tcp::8080-:22 \
    -net nic,model=virtio \
    -device virtio-net,netdev=network0 \
    -netdev tap,id=network0,ifname=tap0,script=no,downscript=no \
    -drive file=./images/ubuntu-22.04.2-live-server-amd64.iso,media=cdrom
	```
	
- After guest OS is installed, boot it with by removing -drive line from above command

- If the OS is installed into pondcxl.qcow2, you should be able to enter the guest OS. Inside the VM, edit /etc/default/grub, make sure the following options are set([more details](https://help.ubuntu.com/community/SerialConsoleHowto)).
	```bash
    GRUB_CMDLINE_LINUX="ip=dhcp console=ttyS0,115200 console=tty console=ttyS0"
    GRUB_TERMINAL=serial
    GRUB_SERIAL_COMMAND="serial --unit=0 --speed=115200 --word=8 --parity=no --stop=1"
  ```
  
- Still in the VM, update the grub
	```bash
	sudo update-grub
	```
	
- Configuring grub
	- Edit /boot/grub/menu.lst:
		```bash
		sudo nano /boot/grub/menu.lst
		```
	- Add the following lines to the top of the file:
		```bash
		# Enable console output via the serial port. unit 0 is /dev/ttyS0, unit 1 is /dev/ttyS1...
    serial --unit=0 --speed=115200 --word=8 --parity=no --stop=1
    terminal --timeout=15 serial console
    ```
	- When you next reboot, the output from grub will go to the normal console unless input is received from the serial port. Whichever receives input first becomes the default console. This gives you the best of both worlds.
	- Poweroff the VM
        ```bash
        sudo shutdown -h now
        ```
- Now you're ready to Run PondVM. If you stick to a Desktop version guest OS, please remove "-nographic" command option from the running script before running PondVM.

- Login to Pond VM

    - If you correctly setup the aforementioned configurations, you should be able to see **text-based** VM login in the same terminal where you issue the running scripts. Run below script to boot up PondVM (script can be found in ./images/qemu-bootup-l75.sh)

        ```bash
        sudo /opt/qemu-7/bin/qemu-system-x86_64 -name PondVM \
        -machine type=pc,accel=kvm,mem-merge=off -enable-kvm \
        -cpu host -smp cpus=8 -m 16384M \
        -object memory-backend-ram,size=12288M,policy=bind,host-nodes=0,id=ram-node0,prealloc=on,prealloc-threads=8 \
        -numa node,nodeid=0,cpus=0-7,memdev=ram-node0 \
        -object memory-backend-ram,size=4096M,policy=bind,host-nodes=1,id=ram-node1,prealloc=on,prealloc-threads=8 \
        -numa node,nodeid=1,memdev=ram-node1 \
        -device virtio-scsi-pci,id=scsi0 \
        -device scsi-hd,drive=hd0 \
        -drive file=./images/pondcxl.qcow2,if=none,aio=native,cache=none,format=qcow2,id=hd0 \
        -net user,hostfwd=tcp::8080-:22 \
        -net nic,model=virtio \
        -device virtio-net,netdev=network0 \
        -netdev tap,id=network0,ifname=tap0,script=no,downscript=no \
        -nographic
        ```
	- more conveniently, PondVM running script has mapped host port 8080 to guest VM port 22, thus, after you install and run openssh-server inside the VM, you can also ssh into the VM via below command line. (Please run it from your host server (not inside VM))
        ```bash
        ssh -p 8080 $user@localhost
        ```
	- How to setup password-less login into VM from server(https://help.ubuntu.com/community/SSH/OpenSSH/Keys)
        ```bash
        ssh-keygen -t rsa -b 4096
        ssh-copy-id -p 8080 $user@localhost
        ssh -p 8080 $user@localhost
        ```