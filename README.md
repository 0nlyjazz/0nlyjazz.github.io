# My notest about Rust, Raspberry, Linux kernel and more...

Disclaimer: The following content are my personal notes and I cannot guarantee that
it will work for you. If you're going to use them, you're on your own

##### Compiling latest Rust enabled linux kernel for aarch64:

I've tried it on ubuntu 22.04 LTS running as a VM (host: ubuntu 22.10)

1. Clone the project: 
<code>git clone https://github.com/Rust-for-Linux/linux.git</code>

2. Install the required packages

```
sudo apt update && sudo apt install flex bison build-essential \
 libncurses-dev libssl-dev libelf-dev clang lld llvm wget autoconf automake
 gcc-aarch64-linux-gnu binutils-aarch64-linux-gnu
 ```

3. Rust needs to be installed, download the <code>rustup</code> tool from the following<br>
<code>curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh</code><br>
Then perform the default installation and after that we need to:
a. install a specific version of the compiler & its toolchain 
b. install the bindgen (this will serve as interface between existing C code and Rust)
c. install the rust source as it is needed by core kernel rust modules 'core' and 'alloc'
```
rustup override set $(scripts/min-tool-version.sh rustc)
cargo install --locked --version $(scripts/min-tool-version.sh bindgen) bindgen
rustup component add rust-src
```

4. Make a minimal compilation configuration<br>
 <code>make ARCH=arm64 LLVM=1 qemu-busybox-min.config rust.config</code>

5. Compile the kernel<br>
 <code>make ARCH=arm64 LLVM=1 -j4</code><br>
Note: -j4 is used by me as my vm was configured with 4 CPUs.

6. To be updated ....