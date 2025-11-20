# Installation and Build Guide

This document describes how to prepare an Ubuntu 22.04 host for **Snort 3** and compile everything from source.  The steps here closely follow official guidance while adapting paths to this project.

## 1. Update the system and install dependencies

Snort 3 relies on a large set of development libraries and tools.  Begin with a clean installation of Ubuntu 22.04, update existing packages and install the required dependencies:

```bash
sudo apt update && sudo apt dist-upgrade -y

sudo apt install -y build-essential libpcap-dev libpcre3-dev libnet1-dev \
  zlib1g-dev luajit hwloc libdnet-dev libdumbnet-dev bison flex liblzma-dev \
  openssl libssl-dev pkg-config libhwloc-dev cmake cpputest libsqlite3-dev \
  uuid-dev libcmocka-dev libnetfilter-queue-dev libmnl-dev autotools-dev \
  libluajit-5.1-dev libunwind-dev libfl-dev
```

These packages provide the compiler, header files and runtime libraries needed to build the Data Acquisition (DAQ) library and Snort 3.  They correspond to the dependencies recommended by multiple deployment guides.

## 2. (Optional) Install gperftools

For high‑traffic deployments, you can improve memory allocation performance by linking Snort against the thread‑caching malloc library provided by **gperftools**.  This step is optional but recommended on busy sensors.

```bash
cd ~/snort_src
wget https://github.com/gperftools/gperftools/releases/download/gperftools-2.10/gperftools-2.10.tar.gz
tar -xvf gperftools-2.10.tar.gz
cd gperftools-2.10
./configure
make
sudo make install
sudo ldconfig
```

Use the `--enable-tcmalloc` option when configuring Snort to link against gperftools.

## 3. Build and install the DAQ library

Snort 3 uses the Data Acquisition (DAQ) library to abstract packet capture from different sources.  It is not provided by the Ubuntu repositories, so you must clone and compile it:

```bash
cd ~/snort_src
git clone https://github.com/snort3/libdaq.git
cd libdaq
./bootstrap
./configure
make
sudo make install
sudo ldconfig
```

The `ldconfig` command updates the dynamic linker cache so that Snort can find the newly installed library.

## 4. Download and build Snort 3

Now download the Snort 3 source tree from GitHub and build it using CMake.  Adjust the `--prefix` to control where files are installed; `/usr/local` is used here.

```bash
cd ~/snort_src
wget https://github.com/snort3/snort3/archive/refs/heads/master.zip -O snort3.zip
unzip snort3.zip
cd snort3-master

# Configure the build.  Use --enable-tcmalloc if you installed gperftools.
./configure_cmake.sh --prefix=/usr/local --enable-tcmalloc

cd build
make -j$(nproc)
sudo make install
sudo ldconfig
```

Use `snort -V` after installation to verify that the binary is in your `PATH` and reports the expected version.

At this point Snort and its libraries are installed under `/usr/local`, but no configuration or services have been set up yet.  Continue with the [configuration guide](CONFIG.md) to tailor Snort to your environment.
