# Building ArchLinux for BrainyPi 

This guide is to build ArchLinuxARM for BrainyPi. It will be a minimal system with i3-WM. This is still in development. Issues or suggestions are welcome.      

Linux OS built from this guide are not :

1. Fully feature rich 
2. Might be missing key features 
3. Might have some issues
4. Some features might not be fully tested. 

Please take a look at [Known Issues](#9-known-issues).

## 1. Requirements

1.  Ubuntu 20.04 Laptop/PC or virtual machine. (These steps have been tried and tested on Ubuntu 20.04)
2.  Minimum 10 GB of free disk space.

## 2. Overview of steps

1.  [Prepare system for building Linux](#3-prepare-system-for-building-linux) 
1.  [Download source code](#4-download-source-code)
1.  [Compile Uboot, Kernel and Ubuntu](#5-compile-uboot-kernel-and-ubuntu)
    1.  [Compile Uboot](#5i-compile-uboot) 
    1.  [Compile Kernel](#5ii-compile-kernel)
    1.  [Building ArchLinux rfs](#5iii-build-ubuntu)
		1. Minimal RFS
		1. Installing and configuring i3-WM
1.  [Generate ArchLinux image](#6-generate-ubuntu-image)
1.  [Flashing ArchLinux image to BrainyPi](#7-flashing-ubuntu-image-to-brainypi)

## 3. Prepare system for building Linux 

1.	Install tools for building.

	```sh
	sudo apt-get update
	sudo apt-get install git
	sudo apt-get install gcc-aarch64-linux-gnu device-tree-compiler libncurses5 libncurses5-dev build-essential libssl-dev mtools flex bison
	sudo apt-get install gawk wget git-core diffstat unzip texinfo gcc-multilib build-essential chrpath socat cpio python python3 python3-pip python3-pexpect xz-utils debianutils iputils-ping libsdl1.2-dev xterm sshpass curl git subversion g++ zlib1g-dev build-essential git python rsync man-db libncurses5-dev gawk gettext unzip file libssl-dev wget bc
	sudo apt-get install bc python dosfstools qemu-user-static
	```
1.	Install toolchain.

	```sh
	wget https://releases.linaro.org/components/toolchain/binaries/7.3-2018.05/aarch64-linux-gnu/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz
	sudo tar xvf gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu.tar.xz  -C /usr/local/
	export ARCH=arm64
	export CROSS_COMPILE=/usr/local/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-
	export PATH=/usr/local/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu/bin:$PATH
	```
1.	Check if toolchain is the default choice:

	```sh
	which aarch64-linux-gnu-gcc
	/usr/local/gcc-linaro-7.3.1-2018.05-x86_64_aarch64-linux-gnu/bin/aarch64-linux-gnu-gcc
	```
	
## 4. Download source code 

```sh
https://github.com/YenruoY/brainypi-linux-build-archlinux.git
cd brainypi-linux-build-archlinux
git submodule update --init --recursive
```

## 5. Compile Uboot, Kernel and Ubuntu

### 5.1 Compile Uboot

1. Compile u-boot

	```sh
	cd u-boot 
	make rk3399-brainypi_defconfig
	make -j$(nproc)
	cd ../
	./build/mk-uboot.sh brainypi
	```
1.	The compiled artifacts will be copied to out/u-boot folder

	```sh
	ls out/u-boot/
	idbloader.img  rk3399_loader_v1.20.119.bin  spi  trust.img  uboot.img
	```

### 5.2 Compile Kernel

1.	Compile kernel without any changes to configuration.

	```sh
	./build/mk-kernel.sh brainypi
	```

1.  Compile kernel with changes to configuration 

    1.  Load default configuration

        ```sh
		cd kernel
        make rockchip_linux_defconfig
        ```
    1.  Make changes to configuration

        ```sh
        make menuconfig 
        ```
    1.	Save configuration and copy as default configuration

        ```sh
        make savedefconfig
        cp defconfig arch/arm64/configs/rockchip_linux_defconfig
        cd ../
        ```

    1.  Build kernel
 
    	```sh
        ./build/mk-kernel.sh brainypi
        ```
        
1.	The compiled artifacts will be copied to folder `out/kernel`

	```sh
	ls out/kernel/
	Image  rk3399-brainypi.dtb 
	```

### 5.3 Building ArchLinux

1.  Download ArchLinux base root filesystem.

		wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
    
1.	Extract base root filesystem

	```sh
	mkdir rfs
	sudo tar xzvf ./ArchLinuxARM-aarch64-latest.tar.gz -C ./rfs
	```

1.	Copy the kernel modules to root filesystem

	```sh
	sudo find out/rootfs/ -name build | xargs rm -rf
	sudo find out/rootfs/ -name source | xargs rm -rf
	sudo cp -arv out/rootfs/lib/* ./rfs/usr/lib/*
	```

1.	Pre-chroot configuration 

	```sh
	sudo cp -av /usr/bin/qemu-aarch64-static ./rfs/usr/bin
	sudo rm ./rfs/etc/resolv.conf
	sudo cp -av /etc/resolv.conf ./rfs/etc/resolv.conf	
	```

	Edit the file `./rfs/etc/pacman.conf` by commenting the keyword **CheckSpace** by adding `#` in front of the keyword.

        $ sudo vim ./rfs/etc/pacman.conf

1. Mounting chroot

	```sh
	sudo ./build/ch-mount.sh -m ./rfs/
	```

1.	Setup user password
 
	```sh
	useradd -G wheel -m -s /bin/bash pi
	echo pi:brainy | chpasswd
	```

1.	Set hostname 

	```sh
	echo brainypi > /etc/hostname
	echo "127.0.0.1	   localhost" > /etc/hosts
	echo "127.0.1.1	   brainypi" >> /etc/hosts
	```

1.	Setting up the package manager

	```sh
	pacman-key --init
    pacman-key --populate
	pacman -Sy archlinux-keyring
	pacman -Syu
	```

1.	Generate locale

	```
	nano /etc/locale.gen
	```

	Un-comment the required locale, save it using `Ctrl+x` then `y` and then execute the below command
	
        $ locale-gen

1.	Install minimal packages

	```sh
	pacman -S sudo net-tools ethtool udev wireless_tools iputils resolvconf wget wpa_supplicant nano networkmanager openssh rxvt-unicode ttf-dejavu
	```

1.	Install and configuring i3-WM 

	1. Installing the required packages

		```sh
		pacman -Sy i3 xorg lightdm thunar lightdm-webkit2-greeter dmenu i3status i3lock ttf-dejavu
		```
		At this stage any other preferred packages can be install here.

	1. Configuring the greeter
		
		```sh
		nano /etc/lightdm/lightdm.conf
		```
	
		Add the following line after `[Seat:*]`

		```sh
		[Seat:*]
		greeter-session=lightdm-webkit2-greeter
		```
		Enable the greeter

		```sh
		systemctl enable lightdm
		```

	1. Configure session to start i3 on login

		```sh
		echo “exec i3” > /home/pi/.xsession
		chmod +x /home/pi/.xsession
		```

	1. Install `polkit` for system wide privileges and an authentication agent `lxsession`.

		```sh
		pacman -S polkit lxsession
		```

		Then add `lxsession` to `.xsession` file

		```sh
		echo "lxsession &" >> /home/pi/.xession
		```
	
1.	Exit chroot
 
	```sh
	exit
	sudo ./build/ch-mount.sh -u ./rfs/
	```

1.	Package the rootfs 

	```sh
	sudo mv ./rfs ./binary
	sudo tar -czvf ./rfs.tar.gz ./binary
	```

## 6. Generate ArchLinux Image 

1.	Generate image for brainypi

	```sh
	./build/mk-image.sh -c rk3399 -b brainypi -t system -r ./rfs.tar.gz
	```

1.	This will combine u-boot, kernel and root filesystem into one image at location `out/system.img`

1.	`system.img` can be flashed into EMMC or into SD card using Etcher.

## 7. Flashing image to BrainyPi

1.  See the Flashing guides to flash ubuntu on BrainyPi
    1.  [Flash to Internal Storage (EMMC)](./Flashing_on_EMMC.md)
    1.  [Flash to SDcard](./Flashing_on_SDcard.md)

## 9. Known Issues
1. Images built without GUI do not boot up to console. This is because of missing TTY config in default kernel configuration (rockchip_linux_defconfig).
	
	- To enable it, enable `USB port drivers` in the menuconfig.

2. Bluetooth does not work. This is because the bluetooth userspace drivers are missing. 
3. Docker does not work. This is because of missing kernel configuration for docker in default kernel configuration (rockchip_linux_defconfig).
4. Audio does not work.