# Rust, Raspberry, Linux kernel and more...


Disclaimer: The following content are my personal notes and I cannot guarantee that
it will work for you. If you're going to use them, you're on your own

#### My Notes:

##### Compiling latest Rust enabled linux kernel for aarch64:

I've tried it on ubuntu 22.04 LTS running as a VM (host: ubuntu 22.10)

1. Clone the project: 
<code>git clone https://github.com/Rust-for-Linux/linux.git</code>

2. Install the required packages

```
sudo apt update && sudo apt install flex bison build-essential \
 libncurses-dev libssl-dev libelf-dev clang lld llvm
 ``` 
 3. Make a minimal compilation configuration 
 <code> make ARCH=arm64 LLVM=1 qemu-busybox-min.config rust.config</cde>

 4. Compile the kernel 
 <code> make ARCH=arm64 LLVM=1 -j4</code>

-> -j4 is used by me as my vm was configured with 4 CPUs.
