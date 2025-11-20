# Snort 3 Configuration Guide

This guide explains how to prepare the Snort configuration files, set protected networks and rule sets, and create the necessary directories for rules and logs.

Snort 3 uses a **Lua** configuration file (`snort.lua`) located under `/usr/local/etc/snort` by default.  This file loads defaults, defines network variables and configures modules.  A sample configuration is provided in `config/snort.lua` within this repository.

## 1. Define network variables

At the top of your `snort.lua` file, set the networks you want to monitor and protect.  `HOME_NET` should encompass the IP ranges under your control, and `EXTERNAL_NET` is defined relative to `HOME_NET`.  For example, if your lab uses the `192.168.153.0/24` subnet:

```lua
HOME_NET = '192.168.153.0/24'

-- external addresses are any not in HOME_NET
EXTERNAL_NET = '!$HOME_NET'

-- load default variables and settings
require('snort_defaults_pkg')
include 'snort_defaults.lua'
```

Defining these variables early ensures that subsequent modules and rules can reference them.

## 2. Configure rule sources

Snort 3 can load rules from multiple files.  Create directories to hold rules, SO rules and lists, then add them to the configuration.  For example:

```bash
sudo mkdir -p /usr/local/etc/rules /usr/local/etc/so_rules /usr/local/etc/lists
sudo touch /usr/local/etc/rules/local.rules
sudo mkdir -p /var/log/snort
sudo useradd -r -s /usr/sbin/nologin -M -c SNORT_IDS snort
sudo chown -R snort:snort /var/log/snort
sudo chmod -R 5775 /var/log/snort
```

In the Lua config, specify the rule files you want to include:

```lua
ips =
{
    mode = 'tap',          -- run in IDS mode; use 'inline' for IPS
    variables = default_variables,
    rules = [[
        -- include community and local rules
        include /usr/local/etc/rules/snort3-community-rules/snort3-community.rules
        include /usr/local/etc/rules/local.rules
    ]]
}

-- enable fast alert output to a file
alert_fast = {
    file = true,
    limit = 0
}

-- direct alert_fast output to /var/log/snort
output = {
    alert_fast = { file = '/var/log/snort/alert_fast.txt' }
}
```

The above snippet tells Snort to run in **tap** mode, load both the community rules and a custom `local.rules` file, and use the **alert_fast** output module to write alerts to `/var/log/snort/alert_fast.txt`.

## 3. Add local rules

Create a simple rule in `local.rules` to verify that Snort can trigger alerts:

```snort
alert icmp any any -> any any (
    msg:"ICMP traffic detected";
    sid:10000001;
)
```

Save this file in `/usr/local/etc/rules/local.rules`.  You can add additional signatures or import rule sets from the Snort community as needed.

## 4. Loading the configuration

With `snort.lua` and your rule files in place, run Snort manually to validate the configuration:

```bash
sudo /usr/local/bin/snort -c /usr/local/etc/snort/snort.lua -T
```

The `-T` flag performs a test run and prints module and rule loading information without actually processing network traffic.  Once the configuration loads successfully, proceed to [deploy Snort as a service](SERVICES.md).
