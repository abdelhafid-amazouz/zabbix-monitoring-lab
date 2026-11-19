# Setup guide

How to rebuild this lab. Follow the steps in order.

> ℹ️ This guide gives the method used in this lab. IPs, interface names and the network trigger threshold are examples: adapt them to your own network.

## 1. Create the 3 virtual machines

| VM | Role | IP |
|---|---|---|
| Zabbix Server | Zabbix appliance | `192.168.56.10` |
| Ubuntu Server | Monitored host | `192.168.56.11` |
| MX | Monitored host | `192.168.56.13` |

Give every VM a **Host-only** network adapter (network `192.168.56.0/24`).
The Zabbix server uses the official **Zabbix Appliance** (a ready-made VM image with the server, database and web interface already installed), version 7.x. Download it from https://www.zabbix.com/download_appliance and import it in VirtualBox. Ubuntu Server and MX are normal Linux VMs.

## 2. Set fixed IP addresses

Check each address with:

```bash
ip addr show <interface>
```

Interfaces used in the lab: `eth1` (Zabbix server), `enp0s8` (Ubuntu), `eth1` (MX).
The method depends on the OS. Examples:

**Ubuntu Server (netplan)**, file `/etc/netplan/50-cloud-init.yaml` (the file name can differ):

```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      dhcp4: false
      addresses: [192.168.56.11/24]
```

```bash
sudo netplan apply
```

**Zabbix Appliance (NetworkManager)**:

```bash
nmcli con mod eth1 ipv4.method manual ipv4.addresses 192.168.56.10/24
nmcli con up eth1
```

**MX**: same idea with the network manager of the desktop, or with `nmcli` using `192.168.56.13/24`.

Test that the machines see each other:

```bash
ping 192.168.56.10
```

## 3. Install Zabbix Agent 2 on the monitored machines

Install it from the official Zabbix download page for your OS and Zabbix version: https://www.zabbix.com/download

Example for Ubuntu (replace `<zabbix-version>` and `<ubuntu-release>` with the values shown on the download page, for example `7.4` and `24.04`):

```bash
wget https://repo.zabbix.com/zabbix/<zabbix-version>/release/ubuntu/pool/main/z/zabbix-release/zabbix-release_latest_<zabbix-version>+ubuntu<ubuntu-release>_all.deb
sudo dpkg -i zabbix-release_latest_<zabbix-version>+ubuntu<ubuntu-release>_all.deb
sudo apt update
sudo apt install zabbix-agent2
```

Then edit the agent configuration file (usually `/etc/zabbix/zabbix_agent2.conf`):

```ini
Server=192.168.56.10
ServerActive=192.168.56.10
Hostname=Ubuntu-server
```

Restart and enable the agent:

```bash
sudo systemctl restart zabbix-agent2
sudo systemctl enable zabbix-agent2
sudo systemctl status zabbix-agent2
```

Repeat on the MX machine with `Hostname=MX`.

## 4. Add the hosts in the Zabbix web interface

1. Open `http://192.168.56.10` in a browser and log in.
2. Go to **Data collection → Hosts → Create host**.
3. Set the host name, the template **Linux by Zabbix agent**, a host group, and an **Agent** interface with the host IP and port `10050`.
4. Wait a minute. The **ZBX** badge turns green when the agent answers.

## 5. Create the e-mail media type

1. **Alerts → Media types → Gmail** (or create a new *Email* type).
2. Set the e-mail provider to **Gmail** and enter the sending address.
3. Gmail usually needs an **app password** (or OAuth), not your normal password. Create an app password in your Google account and use that.
4. Set the message format (HTML or plain text) and save.

## 6. Attach the media to a user

1. **Users → Users → Admin → Media tab → Add**.
2. Type: Gmail. Send to: the address that receives alerts.
3. When active: `1-7,00:00-24:00` (24/7). Tick every severity you want.

## 7. Make sure an action sends the alerts

Go to **Alerts → Actions → Trigger actions** and check that an action is enabled and sends messages to the user.
Use the default action **Report problems to Zabbix administrators** (enable it if it is disabled), or create your own. Without an enabled action, no e-mail is sent.

## 8. Run the tests

### Test 1 — agent down

```bash
sudo systemctl stop zabbix-agent2
# wait about 3 minutes, check Monitoring → Problems and your mailbox
sudo systemctl start zabbix-agent2
```

### Test 2 — network traffic

Create a 500 MB file and copy it between two VMs:

```bash
dd if=/dev/zero of=testfile.bin bs=1M count=500
scp testfile.bin <username>@192.168.56.11:/tmp/
```

This test uses a **custom trigger**, shown as *(manual)* in Zabbix. Create it on the Ubuntu host: **Data collection → Hosts → Ubuntu-server → Triggers → Create trigger**.

- **Name:** `High network traffic on enp0s8`
- **Severity:** Average
- **Expression** (example, with the incoming traffic item of `enp0s8`):

```
avg(/Ubuntu-server/net.if.in["enp0s8"],1m)>100M
```

It means: raise a problem when the incoming traffic on `enp0s8` averages more than 100 Mbps over 1 minute. In this lab the `scp` ran at about 27 MB/s (around 218 Mbps), so it crossed the threshold. Adapt the value to your own network speed, and check the exact item key in **Latest data**.

### Test 3 — CPU overload

```bash
sudo apt install stress-ng
stress-ng --cpu 0 --timeout 360
```

`--cpu 0` uses all CPU cores. `--timeout 360` stops it after 6 minutes. The default trigger needs CPU above 90% for 5 minutes, so the test must run longer than that.

## Troubleshooting

| Symptom | Check |
|---|---|
| Host stays red / grey | Is the agent running? Is port `10050` reachable? Does `Server=` match the Zabbix IP? |
| No e-mail arrives | Gmail app password, media type test button, user media severities, spam folder |
| Wrong host name | `Hostname=` in the agent config must match the host name in Zabbix |
| Time out of sync warning | Sync the clock on the VMs (NTP) |
