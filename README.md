# Notes about cross-compilation on ARM64, Linux kernel, Raspberry Pi, Rust etc.

Disclaimer: The following content are my personal notes and I cannot guarantee that it will work for you. If you're going to use them, you're on your own. I've tried it on ubuntu 22.04 LTS running as a VM (host: ubuntu 22.10)


## Aim: provide rust support in one of the stable kernels out there.
I've been using one of the stable kernels (5.15.y) for some of my hobby projects that 
includes a raspberry pi. Lately, I've started learning Rust and the project 
"Rust for Linux" grabbed my attention particularly. Seriously, a second language in kernel?

C++ was considered but quickly removed as it didn't serve the purpose & when we all 
started to think that C and assembly will be the de-facto language of the kernel and *boom* ...
rust happened!

Thanks to Miguel and the fantastic "Rust for linux" commuinity, kernel 6.1 finally saw Rust being
mainlined and this got me thinking...

How about giving rust support to one of the older, stable kernel branches? (pardon my overusage of the word 'stable', I certainly mean 'long-term supported' kernels)

Meanwhile something else happened, my sole raspberry pi board went kaput and due to acute semiconductor shortage, a replacement will take quite sometime to arrive but hey! QEmu to the rescue :-)

Not only this will serve as a nice little weekend project but also, I can keep playing with the software and by the time the board arrives, my rustified kernel will be ready. 


## In a nutshell:


### Step 1: Collecting needed sources and setting up development environment

* Get rust-for-linux source      : Done
* Get stable linux tree          : Done
* Get busybox source             : Done
* Get raspberry pi kernel source : Done

### Step 2: Configuring and compiling the sources 

1. Configure and cross-compile rust-for-linux source for aarch64
2. Configure and cross-compile stable kernel (5.15.y) for aarch64
3. Configure and cross-compile busybox for aarch64
4. Configure and cross-compile raspberry pi kernel for aarch64
Note: I used kernel 5.15.y as it matches with my raspberry pi kernel version which I use.
If you've a different kernel running, steps will be same moreorless. 


### Step 3: Use Qemu to emulate arm64 cpu and bootup the kernel in it,

1. boot and test rust-for-linux kernel
2. boot and test stable (5.15.y) kernel
3. boot and test raspberry pi kernel 


#### Let's dive into Step 1 (Getting the required sources and setting up build environment)


* Clone the following projects: rust-for-linux, busybox, stable kernel tree, raspberry pi kernel<br>

Kernel trees
<code>git clone https://github.com/Rust-for-Linux/linux.git</code><br>
<code>git clone https://github.com/gregkh/linux.git</code><br>
<code>git clone https://github.com/raspberrypi/linux.git</code><br>
Userland
<code>git clone https://github.com/mirrors/busybox.git</code><br>


* Install required packages and setup a development environment
```
sudo apt update && sudo apt install flex bison build-essential \
 libncurses-dev libssl-dev libelf-dev clang lld llvm wget autoconf automake \
 gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu
 ```

* We need some extra packages to be installed if we are compiling raspberry pi kernel as a debian package.

```
sudo apt install dpkg-dev libssl-dev autotools-dev
```

* Rust needs to be installed, download the <code>rustup</code> tool from the following<br>

<code>curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh</code><br>

Then perform the default installation and after that we need to:

a. install a specific version of the compiler & its toolchain<br>
b. install the bindgen (this will serve as interface between existing C code and Rust)<br>
c. install the rust source as it is needed by core kernel rust modules 'core' and 'alloc'<br>

* Go to rust-for-linux tree and run the following commands:

```
rustup override set $(scripts/min-tool-version.sh rustc)
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen
rustup component add rust-src

```

#### Let's dive into Step 2 (configuring and compiling the sources)

* Configure and cross-compile rust-for-linux tree (this is essentially same for other two kernels)
    However, rust configuration is not present by default so we need to omit the "rust.config" from configuration option.

```
 make ARCH=arm64 LLVM=1 qemu-busybox-min.config rust.config

```

* Cross compile the rust-for-linux tree


```
make ARCH=arm64 LLVM=1 -j4

```
Note: -j4 is used by me as my vm was configured with 4 CPUs.

* Configure and cross-compile raspberry pi kernel

```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- bcm2711_defconfig

make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j4 \
    CXXFLAGS="-march=armv8-a+crc -mtune=cortex-a72" \
    CFLAGS="-march=armv8-a+crc -mtune=cortex-a72" \
    bindeb-pkg

```

Note: the bindeb-pkg requires extra packages to be installed, refer step #1 


* Configure and cross-compile busybox<br>

```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig

```
* Enable static linking (menuconfig -> settings -> Build Options -> Build static library)

* Build busybox
```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j4

```


### Now, we need to boot our freshly baked kernels (rust-for-linux, stable and raspberry pi) using the busybox as our userland.

Note: This is a "minimal" userland so there won't be a lot of "commonly expected" features to be present.
The intention is to establish a baseline system that works so that we can fallback to it incase we run into some issues.


A few things needs to be done for busybox before we can run the show:
1. Add procfs support. 
2. Add the inittab
3. Add /etc/init.d/rcS file

Now, let's see how we can do that. As we know already, busybox is going to be our "tiny" rootfs, 
so let's add some basic configuration to it.
In the busybox/_install directory: 

```
cp ../exammples/inittab .
mkdir -p etc/init.d
echo "mkdir -p /proc" > etc/init.d/rcS
echo "mount -t proc none /proc" >> etc/init.d/rcS"
chmod a+x etc/init.d/rcS

```

Note:
Delete following lines in etc/inittab
```
 # Start an "askfirst" shell on /dev/tty2-4                                                                       tty2::askfirst:-/bin/sh
 tty3::askfirst:-/bin/sh
 tty4::askfirst:-/bin/sh
 # /sbin/getty invocations for selected ttys
 tty4::respawn:/sbin/getty 38400 tty5
 tty5::respawn:/sbin/getty 38400 tty6
 ```

Pack the ramdisk by running the following command inside <code>_install</code> directory:

```
find . | cpio -H newc -o | gzip > ../ramdisk.img

```

## Important note:
* Busybox seems not viable for the ARM64 cross compilation environment
* Needed to switch to buildroot. 
* Buildroot can be downloaded from 
```
git clone https://git.buildroot.net/buildroot
```

## Build it 
```
make menuconfig

Target options
    Target Architecture - Aarch64 (little endian)
Toolchain type
    External toolchain - Linaro AArch64
System Configuration
[*] Enable root login with password
        ( ) Root password = set your password using this option
[*] Run a getty (login prompt) after boot  --->
    TTY port - ttyAMA0
Target packages
    [*]   Show packages that are also provided by busybox
    Networking applications
        [*] dhcpcd
        [*] iproute2
        [*] openssh
Filesystem images
    [*] ext2/3/4 root filesystem
        ext2/3/4 variant - ext3
        exact size in blocks - 6000000
    [*] tar the root filesystem

make
```

## QEMU kernel bootup command-line

```
qemu-system-aarch64 -machine virt -cpu cortex-a57 -nographic -smp 2 -append "console=ttyAMA0 root=/dev/vda oops=panic debug " -kernel WORK/github-forks/rust-4-linux/arch/arm64/boot/Image -hda WORK/buildroot/output/images/rootfs.ext2 -m 2048

CONFIG_DEBUG_INFO=y
CONFIG_CMDLINE="console=ttyAMA0"

```

Libclang version:

libclang-common-14-dev/jammy,now 1:14.0.0-1ubuntu1 amd64 [installed,automatic]
libclang-cpp14/jammy,now 1:14.0.0-1ubuntu1 amd64 [installed,automatic]
libclang1-14/jammy,now 1:14.0.0-1ubuntu1 amd64 [installed,automatic]






----------------------------------
# Rust specific details

* Rust : Setting up environment:
Presentation video: https://www.youtube.com/watch?v=tPs1uRqOnlk
Slides: https://events.linuxfoundation.org/wp-content/uploads/2022/10/Wedson-Almeida-Filho-9_29_22-webinar-slide-deck-LF-Session-2.pdf

* Rust: Writing kernel modules in Rust:
Presentation: https://www.youtube.com/watch?v=-l-8WrGHEGI
Slides: https://events.linuxfoundation.org/wp-content/uploads/2022/10/Wedson-Almeida-Filho-LF-Writing-Linux-Kernel-Modules-in-Rust.pdf

* Rust code documentation and tests:
Presentation: https://www.youtube.com/watch?v=J8yoUQKEY5g
Slides: https://events.linuxfoundation.org/wp-content/uploads/2022/04/2022-04-20-Linux-Foundation-LF-Live-Mentorship-Series-Rust-for-Linux-Code-Documentation-Tests.pdf

* Rust : Writing safe abstractions and drivers:
Presentation: https://www.youtube.com/watch?v=3VU0hfsbHdc
Slides: https://events.linuxfoundation.org/wp-content/uploads/2021/11/2021-11-11-Linux-Foundation-LF-Live_-Mentorship-Series-Rust-for-Linux_-Writing-Safe-Abstractions-Drivers.pdf


