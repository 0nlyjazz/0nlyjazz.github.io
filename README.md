# Rust, Raspberry, Linux kernel and more...


Disclaimer: The following content is my personal notes and I cannot guarantee that
it will work for you. If you're going to use them, you're on your own

#### My Notes:

##### Compiling latest Rust enabled linux kernel for aarch64:

I've tried it on ubuntu 22.04 LTS running as a VM (host: ubuntu 22.10)

1. Clone the project: 
<code>git clone https://github.com/Rust-for-Linux/linux.git</code>

2. Install the required packages
<code>sudo apt update && sudo apt install flex bison build-essential libncurses-dev libssl-dev libelf-dev clang lld llvm</code>
