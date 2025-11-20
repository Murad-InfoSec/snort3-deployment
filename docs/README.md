# Snort 3 Deployment Documentation

This folder contains detailed documentation for building and operating the **Snort 3** Intrusion Detection System (IDS) on Ubuntu 22.04.  Each guide focuses on a different aspect of the project, from compiling the software to configuring rules, deploying the system as a service and validating that alerts are generated correctly.

## Available guides

| Guide | Description |
|------|-------------|
| **[INSTALL.md](INSTALL.md)** | Step‑by‑step instructions for installing all prerequisites, compiling the Data Acquisition (DAQ) library and building Snort 3 from source. |
| **[CONFIG.md](CONFIG.md)** | An overview of the Lua‑based Snort configuration, including how to set your protected networks, enable rule sets and prepare directories. |
| **[SERVICES.md](SERVICES.md)** | Documentation of the systemd unit files used to place a network interface in promiscuous mode and run Snort as a daemon.  Includes explanations of each option and a sample `systemctl status` output. |
| **[TESTING.md](TESTING.md)** | Guidance for generating traffic to trigger alerts, interpreting the alert log and verifying that Snort sees traffic from another host on the same subnet. |

Refer to these guides in order when reproducing the project, or consult a specific section when you need more details on a particular component.
