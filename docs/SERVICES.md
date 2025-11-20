# Running Snort as a Service

For continuous monitoring, it is convenient to run Snort and its network interface preparation via **systemd**.  This guide explains the two unit files included in this project and shows how to enable them.

## 1. Preparing the network interface

Snort inspects every packet on a network interface.  To capture all traffic, the interface must operate in **promiscuous mode**, and offload features like Generic Receive Offload (GRO) and Large Receive Offload (LRO) should be disabled.  The `snort3-nic.service` unit provided here performs these tasks at boot time.

### `snort3-nic.service`

```
[Unit]
Description=Set Snort 3 NIC in promiscuous mode and Disable GRO, LRO on boot
After=network.target

[Service]
Type=oneshot
# Enable promiscuous mode on the monitoring interface and disable generic and
# large receive offload.  Using ExecStartPost ensures both commands run
# sequentially within the same one‑shot service.
ExecStart=/usr/sbin/ip link set dev ens33 promisc on
ExecStartPost=/usr/sbin/ethtool -K ens33 gro off lro off
TimeoutStartSec=0
RemainAfterExit=yes

[Install]
WantedBy=default.target
```

Replace `ens33` with the name of your monitoring interface if necessary.  Install this file under `/etc/systemd/system/` and enable it with:

```bash
sudo systemctl enable --now snort3-nic.service
```

The service runs once on boot and leaves the interface configured.

## 2. Launching Snort as a daemon

The `snort.service` unit starts Snort in daemon mode using the configuration file and directories set up earlier.  Key options include the snap length (`-s 65535`), ignoring bad checksums (`-k none`), dropping privileges and writing a PID file.

### `snort.service`

```
[Unit]
Description=Snort 3 IDS/IPS Daemon
After=network.target snort3-nic.service

[Service]
Type=simple
# Run Snort in daemon mode on interface ens33.  Adjust the interface name to match
# your monitoring NIC.  The extra options increase the snap length, ignore bad
# checksums, drop privileges to the 'snort' user and create a PID file.
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

Copy this file to `/etc/systemd/system/`, reload systemd and enable the service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now snort.service
```

## 3. Checking service status

After enabling both services, verify that they are active:

```bash
sudo systemctl status snort3-nic.service
sudo systemctl status snort.service
```

On a healthy installation, you should see output similar to the following, showing that Snort is running and the NIC service has completed successfully:

![Snort service status](../images/service_status.png)

If Snort fails to start, run `journalctl -u snort.service` to view detailed error messages and correct any configuration issues.
