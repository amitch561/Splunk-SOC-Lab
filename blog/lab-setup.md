# Day 1 - Splunk Enterprise Setup and Log Forwarding

## Objective

Get Splunk Enterprise up and running and configure a Windows VM to send log data to it.

## Environment

I have a virtual lab on Proxmox with OPNsense managing VLANs. I set up a SOC-Lab VLAN for this project to keep the endpoint separate from my security tools.

- Ubuntu Server (Security Tools VLAN) - Splunk Enterprise 10.2.3
- Windows 11 VM (SOC-Lab VLAN) - endpoint with Sysmon and the Splunk Universal Forwarder
- OPNsense - firewall rules between VLANs

I put the Windows endpoint and Splunk on different VLANs on purpose. If someone compromises the endpoint network, they shouldn't be able to reach the SIEM directly.

## Steps

### 1. Reinstalled Splunk Enterprise

My old Splunk install had an expired license and a bunch of leftover apps from previous projects. Easier to just start fresh.

- Removed the old install with `sudo rm -rf /opt/splunk`
- Installed the new .deb package with `sudo dpkg -i`
- Started Splunk with `sudo /opt/splunk/bin/splunk start --accept-license`
- Set up a systemd service so it starts on boot with `sudo /opt/splunk/bin/splunk enable boot-start -systemd-managed 1 -user root`

### 2. Configured Splunk to Receive Data

Went to Settings > Forwarding and Receiving > Configure Receiving in the Splunk UI and added port 9997. This is where the forwarder sends data.

### 3. Firewall Rules

The endpoint and Splunk are on different VLANs so I had to add rules in OPNsense:

- SOC-Lab VLAN to Splunk on TCP 9997 (log data)
- SOC-Lab VLAN to Splunk on TCP 8089 (management)
- SOC-Lab VLAN to DNS on TCP/UDP 53

### 4. Installed Sysmon

Default Windows logging doesn't capture enough for security work. Sysmon adds things like full command line logging, network connections, and file creation events.

- Got Sysmon from Microsoft Sysinternals
- Grabbed the config file from [olafhartong/sysmon-modular](https://github.com/olafhartong/sysmon-modular)
- Installed it with `.\Sysmon64.exe -accepteula -i sysmonconfig.xml`

### 5. Configured the Universal Forwarder

The forwarder was already installed and pointed at my Splunk server on 9997 but it wasn't actually collecting any Windows logs. By default it only monitors its own internal logs which isn't useful.

I had to create an inputs.conf file at `C:\Program Files\SplunkUniversalForwarder\etc\system\local\inputs.conf`:

```
[WinEventLog://Security]
disabled = 0
index = main

[WinEventLog://System]
disabled = 0
index = main

[WinEventLog://Application]
disabled = 0
index = main

[WinEventLog://Microsoft-Windows-Sysmon/Operational]
disabled = 0
index = main
renderXml = true
```

This tells the forwarder to collect Security, System, Application, and Sysmon logs and send them to the main index.

The channel names match what you see in Windows Event Viewer. The full syntax reference for inputs.conf lives on the Splunk server at `/opt/splunk/etc/system/README/inputs.conf.spec`.

## Challenges

The systemd service defaulted to running Splunk as a "splunk" user but all the files were owned by root. Got a wall of permission denied errors. Had to redo the boot-start command with `-user root`.

Notepad saved my inputs.conf as inputs.conf.txt without telling me. Windows was hiding the extension so it looked right in File Explorer but the forwarder couldn't find it. Ended up recreating it through PowerShell.

Even after all that, Splunk still showed zero events. Turned out I needed to restart the forwarder service as Administrator for the config changes to take effect.

## Result

Searched `index=*` in Splunk and events started showing up. Security logs, System logs, and Sysmon data all coming in from the Windows endpoint.

*(See screenshot: Splunk Events Flowing)*

## Takeaways

The forwarder doesn't send anything useful out of the box. You have to configure inputs.conf to tell it what to collect. I didn't know that going in.

This was all infrastructure work, not really what a SOC analyst does day to day. But knowing how data actually gets into the SIEM is useful. If logs stop showing up you need to know where to look - the forwarder config, the receiving port, the firewall rules.
