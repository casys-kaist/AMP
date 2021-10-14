# Experiment Environment Setup
## 1. Setup `sudo` without Password
```
$ sudo update-alternatives --config editor # choose vim
$ sudo visudo
<your id> ALL=(ALL) NOPASSWD:ALL
```

## 2. Update the Host Machine's Linux Kernel
* Please enable `CONFIG_TRANSPARENT_HUGEPAGE_ALWAYS`.
```
$ sudo apt install -y build-essential libncurses5-dev libssl-dev bc bison flex libelf-dev
$ cd linux-v4.15-vanilla
$ cp /boot/<current config> .config
$ yes "" | make oldconfig
$ sudo make all -j$(nproc) && sudo make modules_install -j$(nproc) && sudo make install -j$(nproc)
$ sudo vim /etc/default/grub
$ sudo update-grub
$ sudo reboot now
```
## 3. Install QEMU
```
$ cd qemu
$ sudo sed -Ei 's/^# deb-src /deb-src /' /etc/apt/sources.list
$ sudo apt -y update && sudo apt -y build-dep qemu
$ ./configure --target-list=x86_64-softmmu
$ sudo make install -j$(nproc)
```
## 4. Create a Guest Virtual Machine
```
$ wget https://mirror.kakao.com/ubuntu-releases/bionic/ubuntu-18.04.6-desktop-amd64.iso
$ qemu-img create ubuntu.img 100G
$ qemu-system-x86_64 \
  -enable-kvm \
  -drive file=ubuntu.img,index=0,media=disk,format=raw \
  -netdev user,id=hostnet0,hostfwd=tcp::2222-:22 \
  -device virtio-net-pci,netdev=hostnet0,id=net0,bus=pci.0,addr=0x3 \
  -m 2048 \
  -boot d -cdrom ./ubuntu-18.04.5-live-server-amd64.iso \
  -vnc :0
```
## 5. Update the Guest Machine's Linux Kernel
```
# apt install -y build-essential libncurses5-dev libssl-dev bc bison flex libelf-dev
# git clone git@github.com:casys-kaist/linux-v4.15-page-migration.git
# cd linux-v4.15-page-migration
# cp /boot/<current config> .config
# yes "" | make oldconfig
# vim .config
```

* Follow the configurations below to enable kernel debugging.
```
CONFIG_RANDOMIZE_BASE=n
CONFIG_FRAME_POINTER=y
CONFIG_KGDB=y
CONFIG_KGDB_SERIAL_CONSOLE=y
CONFIG_KGDB_KDB=y
CONFIG_KDB_KEYBOARD=y
```

* Run `check-config.sh` to make sure that the kernel configuration supports docker.

* Update the GRUB options to enable kernel debugging (`console`) and to increase the dmesg buffer size (`log_buf_len`).
```
GRUB_CMDLINE_LINUX="console=tty0 console=ttyS0,115200n8 log_buf_len=536870912"
```

```
# make all -j$(nproc) && make modules_install -j$(nproc) && make install -j$(nproc)
# vim /etc/default/grub
# update-grub
# reboot now
```
## 6. Setup Benchmarks
* Install Docker in the guest VM.
```
# apt -y update
# apt -y install \
    apt-transport-https \
    ca-certificates \
    curl \
    software-properties-common
# curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
# apt-key fingerprint 0EBFCD88
# add-apt-repository \
   "deb [arch=amd64] https://download.docker.com/linux/ubuntu \
   $(lsb_release -cs) \
   stable"
# apt -y update
# apt -y install docker-ce
```
* After that, build docker images by running `build.sh` in each directory.

## 7. Install the Experiment Library
* Install the experiment library that helps running workloads in a guest VM.
```
$ cd AMP-experiment-library
$ sudo python3 setup.py install
```

## 8. Run Experiments
* Run a QEMU virtual machine.
```
$ cd runqemu
$ python3 runqemu.py -drive_image_path ~/qemu-virtual-machines/ubuntu.img -num_nodes 2 -memory_size 32 -num_cores 10
```
* Emulate slow memory.
```
$ cd slow-memory-emulation
$ ./calibrate.sh 300
$ ./emulate.sh 0x3821c
```
* Run experiments.
```
$ cd AMP-experiment-scripts
$ cd run-workloads
$ python3 graph500.py -result_dir /home/tkheo/experiment-results 
```
