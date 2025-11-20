https://medium.com/@MuradInfoSec/building-a-modern-ids-with-snort-3-on-ubuntu-3c7cf9a5aa9d

# Building a Modern IDS with Snort 3 on Ubuntu

Securing network traffic is a critical component of any defence‑in‑depth strategy. While firewalls and endpoint tools help, a **network intrusion detection system (IDS)** provides visibility into malicious activity that traverses your LAN. Snort has long been the open‑source IDS of choice; the latest release, **Snort 3**, redesigns the engine for performance and usability improvements【911547961791720†L38-L44】. This article walks through building and configuring Snort 3 on an Ubuntu 22.04 host, exploring the advantages of the new engine and demonstrating how to generate and inspect alerts.

## Why Snort and Why Version 3?

Snort is an open‑source intrusion detection and prevention system capable of real‑time packet analysis and logging. It combines signature, protocol and anomaly‑based techniques to detect denial‑of‑service, port scans, buffer overflows and other attacks【951449535655284†L310-L337】. Millions of downloads and an active community of ~400,000 registered users make Snort the most widely deployed IDS/IPS solution【951449535655284†L310-L315】. Snort’s longevity comes from its accuracy, adaptability and quick response; a global community continuously improves the engine, and Cisco’s Talos team ships updated signatures every hour【951449535655284†L415-L429】.

Snort 3 introduces significant improvements over the 2.x branch. The engine has been rewritten in C++, resulting in a modular code base that is easier to maintain【911547961791720†L52-L56】. Multi‑threading and shared memory allow Snort to process packets in parallel for faster start‑up and better scalability【911547961791720†L59-L61】. The new engine also exposes a richer plugin interface via LuaJIT, enabling custom modules and simplifying rule syntax【911547961791720†L63-L73】. A comparison table shows that Snort 3 supports any number of packet threads, allows nested policy bindings and introduces more flexible detection thresholds【911547961791720†L75-L96】. These enhancements make Snort 3 well suited for high‑speed networks and future development.

## Preparing the Environment

### Installing prerequisites

Begin on a freshly installed Ubuntu 22.04 host. Update the package list and install the build toolchain along with libraries Snort depends on:

```bash
apt update && apt dist-upgrade -y
apt install -y build-essential libpcap-dev libpcre3-dev libnet1-dev \
    zlib1g-dev luajit hwloc libdnet-dev libdumbnet-dev bison flex liblzma-dev \
    openssl libssl-dev pkg-config libhwloc-dev cmake cpputest libsqlite3-dev \
    uuid-dev libcmocka-dev libnetfilter-queue-dev libmnl-dev autotools-dev \
    libluajit-5.1-dev libunwind-dev libfl-dev
```

This extensive list mirrors the dependencies recommended by official guides【26014408151962†L247-L259】 and ensures that the Snort build process will succeed.

### Optional: gperftools

Snort can use the thread‑caching malloc library for improved memory allocation performance. Building gperftools is optional but recommended for busy systems. Download and compile it from source if desired.

## Building the DAQ library and Snort 3

Snort relies on the **Data Acquisition (DAQ) library** to abstract packet capture. The `libdaq` code is not provided by Ubuntu and must be compiled. Clone the repository and build it:

```bash
cd ~/snort_src
git clone https://github.com/snort3/libdaq.git
cd libdaq
./bootstrap
./configure
make
make install
```

Next, obtain the Snort 3 source code from GitHub, configure it with CMake and compile:

```bash
cd ~/snort_src
wget https://github.com/snort3/snort3/archive/refs/heads/master.zip -O snort3.zip
unzip snort3.zip
cd snort3-master
./configure_cmake.sh --prefix=/usr/local --enable-tcmalloc
cd build
make
make install
ldconfig
```

The `--enable-tcmalloc` option links Snort against gperftools if you built it. After installation, verify the version with `snort -V` to confirm that the binary is available under `/usr/local/bin`.

## Configuring Snort

### Setting network variables and directories

Snort uses a Lua configuration file. Copy or symlink `snort_defaults.lua` to `snort_defaults_pkg.lua` in `/usr/local/etc/snort` so that the base configuration can be loaded without specifying an absolute path. Then edit `snort.lua` and set the networks you want to protect:

```lua
HOME_NET = '192.168.153.0/24'
EXTERNAL_NET = '!$HOME_NET'
```

Create directories for rules and logs and ensure they are writable by the Snort user:

```bash
mkdir -p /usr/local/etc/rules /usr/local/etc/so_rules /usr/local/etc/lists
touch /usr/local/etc/rules/local.rules
mkdir -p /var/log/snort
useradd -r -s /usr/sbin/nologin -M -c SNORT_IDS snort
chown -R snort:snort /var/log/snort
chmod -R 5775 /var/log/snort
```

Add a basic rule to `local.rules` to verify that Snort can trigger alerts:

```
alert icmp any any -> any any ( msg:"ICMP traffic detected"; sid:10000001; metadata:policy security-ips alert; )
```

### Customising the Lua configuration

The provided `config/snort.lua` (see project files) loads the `snort_defaults` package, defines `HOME_NET`/`EXTERNAL_NET` and includes both the community ruleset and the local rules. It also enables the `alert_fast` output plugin, directing alerts to a file under `/var/log/snort`. Running Snort in _tap_ mode (`ips = { mode = 'tap', … }`) ensures that it does not modify traffic; if you plan to run Snort inline as an IPS, change the mode to `inline` and configure appropriate policies.

## Running Snort as a service

### Preparing the network interface

Network interfaces used for monitoring should run in **promiscuous mode** and have offloading disabled to ensure every packet is inspected. A simple `systemd` unit (`snort3-nic.service`) accomplishes this:

```ini
[Unit]
Description=Set Snort 3 NIC in promiscuous mode and disable GRO, LRO on boot
After=network.target

[Service]
Type=oneshot
ExecStart=/usr/sbin/ip link set dev ens33 promisc on
ExecStartPost=/usr/sbin/ethtool -K ens33 gro off lro off
TimeoutStartSec=0
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

Install this file under `/etc/systemd/system/` and enable it with `systemctl enable --now snort3-nic.service`. On boot, the interface `ens33` will remain in monitoring mode.

### Creating the Snort service

Running Snort continuously as a daemon is best handled via `systemd`. A robust `snort.service` might look like this:

```ini
[Unit]
Description=Snort 3 IDS/IPS Daemon
After=network.target snort3-nic.service

[Service]
Type=simple
ExecStart=/usr/local/bin/snort \
  -c /usr/local/etc/snort/snort.lua \
  -i ens33 \
  -A alert_fast \
  -s 65535 \
  -k none \
  -l /var/log/snort \
  -D \
  -u snort \
  -g snort \
  --create-pidfile
Restart=always

[Install]
WantedBy=multi-user.target
```

Notably, the service passes `-s 65535` to avoid truncating packets and `-k none` to ignore invalid checksums. It drops privileges to the `snort` user and group and writes a PID file so that systemd can track the process. Enable the service with `systemctl enable --now snort.service`.

### Verifying that Snort is running

Check the status of both services to ensure they are active. The screenshot below shows successful activation on our test machine:

![Service status output for snort.service and snort3-nic.service]({{file:file-9nvTVPgpMtH8qsQ4UL3Bfg}})

## Testing and generating alerts

With Snort running, it’s time to generate traffic. Use a second machine on the same network (as shown below) or simply open another terminal to run a test. Running `ip a` on the second host confirms that it’s on the `192.168.153.0/24` subnet:

![IP address of the test machine on the same subnet]({{file:file-491SELV1exdkXZbPm6fNXu}})

To trigger the sample rule, send an ICMP echo request or visit [testmyids.com](https://testmyids.com), which returns a web page designed to trigger IDS signatures. Execute the following:

```bash
curl http://testmyids.com
```

Within seconds Snort writes an entry to `alert_fast.txt`. Use `tail -f /var/log/snort/alert_fast.txt` to watch alerts stream in. An example of the alert output is captured here:

![Alerts generated after visiting testmyids.com]({{file:file-1rd6e17F5VaWMTdby4Kdcq}})

Another terminal view shows additional alerts and commands run during testing. Note that Snort generates `INDICATOR‑COMPROMISE id check returned root` alerts when the IDS rule triggers, highlighting suspicious traffic【26014408151962†L240-L259】:

![Additional alert logs and command history]({{file:file-4oMRcRCWKbpjsJJyiSYMyr}})

## Lessons learned

Deploying Snort 3 from source may seem intimidating, but the process becomes straightforward with a methodical approach. The new engine delivers improved performance and modularity, thanks to its multi‑threaded design and plugin architecture【911547961791720†L59-L67】. Building from source ensures that you get the latest updates from the constantly evolving Snort community, and running it as a `systemd` service guarantees resilience.

Snort remains a powerful tool for defenders. Its combination of signature, protocol and anomaly detection techniques allows it to identify a wide range of threats【951449535655284†L310-L337】. By integrating Snort into your lab or production network you gain real‑time visibility into malicious traffic and can build automated responses around its alerts. Snort’s open‑source nature means there is no licensing cost, and its extensible rule language lets you adapt it to your environment.

## Conclusion

This project demonstrates how to build a modern IDS using Snort 3 on Ubuntu. The steps—installing dependencies, compiling the DAQ library, building Snort, configuring the Lua file, tuning the network interface and creating robust `systemd` units—result in a clean and repeatable deployment. Once the service is running, it’s easy to add community or custom rules and begin monitoring traffic. Snort’s redesign makes it scalable, adaptable and faster than ever【911547961791720†L59-L73】. Whether you’re a security enthusiast or a professional seeking to strengthen your defences, Snort 3 is a powerful addition to your toolkit.
