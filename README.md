# My notes about Rust, Raspberry Pi, Linux kernel and more...

Disclaimer: The following content are my personal notes and I cannot guarantee that
it will work for you. If you're going to use them, you're on your own.
I've tried it on ubuntu 22.04 LTS running as a VM (host: ubuntu 22.10)


### In a nutshell:

Step 1:

Collecting needed sources and setting up development environment

* Get rust-for-linux source      : Done
* Get stable linux tree          : Done
* Get busybox source             : Done
* Get raspberry pi kernel source : Done

Step 2:

Configuring and compiling the sources 

1. Configure and cross-compile rust-for-linux source for aarch64
2. Configure and cross-compile stable kernel (5.15.y) for aarch64
3. Configure and cross-compile busybox for aarch64
4. Configure and cross-compile raspberry pi kernel for aarch64

Step 3:

Use Qemu to emulate arm64 cpu and bootup the kernel in it,

1. boot and test rust-for-linux kernel
2. boot and test stable (5.15.y) kernel
3. boot and test raspberry pi kernel 

#### Let's dive into Step 1:


Clone the following projects: rust-for-linux, busybox, stable kernel tree, raspberry pi kernel<br>
<code>git clone https://github.com/Rust-for-Linux/linux.git</code><br>
<code>git clone https://github.com/mirrors/busybox.git</code><br>
<code>git clone https://github.com/gregkh/linux.git</code><br>
<code>git clone https://github.com/raspberrypi/linux.git</code><br>

Install required packages and setup a development environment
```
sudo apt update && sudo apt install flex bison build-essential \
 libncurses-dev libssl-dev libelf-dev clang lld llvm wget autoconf automake \
 gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu
 ```

Rust needs to be installed, download the <code>rustup</code> tool from the following<br>

<code>curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh</code><br>

Then perform the default installation and after that we need to:

a. install a specific version of the compiler & its toolchain<br>
b. install the bindgen (this will serve as interface between existing C code and Rust)<br>
c. install the rust source as it is needed by core kernel rust modules 'core' and 'alloc'<br>

Go to rust-for-linux tree and run the following commands:

```
rustup override set $(scripts/min-tool-version.sh rustc)
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen
rustup component add rust-src
```

#### The step 2 (configuring and compiling the sources)

* Configure and cross-compile rust-for-linux tree (this is essentially same for other two kernels)
    However, rust configuration is not present by default so we need to omit the "rust.config" from configuration option.

 ```
 make ARCH=arm64 LLVM=1 qemu-busybox-min.config rust.config
 ```

* Compile the kernel
```
make ARCH=arm64 LLVM=1 -j4
```
Note: -j4 is used by me as my vm was configured with 4 CPUs.


* Configure and cross-compile busybox
```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- menuconfig
```
* Enable static linking (menuconfig -> settings -> Build Options -> Build static library)

* Build busybox
```
make ARCH=arm64 CROSS_COMPILE=aarch64-linux-gnu- -j4
```

----------------------------------
