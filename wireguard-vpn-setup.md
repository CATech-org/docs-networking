# WireGuard VPN Setup on DigitalOcean

A step-by-step guide to setting up a self-hosted WireGuard VPN hub on a DigitalOcean droplet running Ubuntu, enabling secure connectivity between all machines.

---

## 1. Rent a DigitalOcean Server

The DigitalOcean droplet acts as the central VPN hub. Since it only runs WireGuard, the cheapest available droplet is enough.

1. Sign in to [DigitalOcean](https://cloud.digitalocean.com/).
2. Click **Create** → **Droplets**.
3. Choose a region closest to your location for lower latency.
4. Select **Ubuntu** as the operating system (latest LTS recommended).
5. Pick the cheapest plan available (at the time of writing, this is the **$4/mo** tier with 1 GB RAM, 1 vCPU, and 20 GB SSD).
6. Leave all other options at their defaults. No additional volumes, monitoring, or add-ons are needed.
7. Add your SSH key for secure access (or create a root password if you prefer).
8. Click **Create Droplet** and wait for it to spin up.
9. Note the public IP address assigned to your droplet. You will need it throughout this guide.

---

## 2. Configure the DigitalOcean Cloud Firewall

DigitalOcean provides a cloud-level firewall that filters traffic before it reaches your droplet. This is separate from any software firewall (like UFW) running on the server itself.

1. In the DigitalOcean dashboard, go to **Networking** → **Firewalls** → **Create Firewall**.
2. Give it a descriptive name (e.g. `VPN-fw`).

### Inbound Rules

Only allow SSH and the WireGuard ports:

|Type|Protocol|Port|Source|
|---|---|---|---|
|SSH|TCP|22|All IPv4, All IPv6|
|Custom|UDP|51820|All IPv4, All IPv6|
|Custom|UDP|59873|All IPv4, All IPv6|

> **Note:** Port 51820 is the default WireGuard port. Port 59873 is used as a secondary WireGuard listener in this setup. Everything else is blocked inbound by default.

### Outbound Rules

Leave outbound open to allow the server to function normally (DNS resolution, package updates, time sync, etc.). The default outbound rules are fine:

|Type|Protocol|Port|Destination|
|---|---|---|---|
|ICMP|ICMP|All|All IPv4, All IPv6|
|All TCP|TCP|All|All IPv4, All IPv6|
|All UDP|UDP|All|All IPv4, All IPv6|

> **Note:** Firewall security comes primarily from inbound restrictions. Keeping outbound open is standard practice.

3. Under **Apply to Droplets**, select your VPN droplet.
4. Click **Create Firewall**.

---

## 3. Install WireGuard on the Server

We use [angristan/wireguard-install](https://github.com/angristan/wireguard-install), a community script with 11k+ GitHub stars, instead of manually configuring WireGuard. The script automates key generation, interface configuration, and systemd service creation.

1. SSH into your DigitalOcean droplet:

```bash
ssh root@<YOUR_SERVER_IP>
```

2. Download and run the installer script:

```bash
curl -O https://raw.githubusercontent.com/angristan/wireguard-install/master/wireguard-install.sh
chmod +x wireguard-install.sh
./wireguard-install.sh
```

3. The script interactively asks questions (server IP, port, DNS, client name, etc.). Follow the prompts. The defaults work for most setups.

4. Once complete, the script has:

    - Installed WireGuard (kernel module and tools)
    - Generated server and client key pairs
    - Created the server configuration at `/etc/wireguard/wg0.conf`
    - Started and enabled a systemd service for WireGuard
    - Generated a client configuration file (displayed as a QR code and saved to disk)

5. To add more clients later, run the script again:

```bash
./wireguard-install.sh
```

It will detect the existing installation and offer to add or remove clients.

---

## 4. Connect Client Devices

For each device you want on the VPN, run the installer script again to generate a new client configuration file. Then import that configuration into the WireGuard client app on each device.

Generate a separate client config for each device. This way you can revoke access per device if needed.

### Mobile (iOS / Android)

1. Install the **WireGuard** app from the App Store or Google Play.
2. When the script generates a client config, it also displays a **QR code** in the terminal.
3. In the WireGuard app, tap **Add a tunnel** → **Create from QR code** and scan it.
4. Toggle the tunnel on. You are connected.

### macOS / Windows / Linux

1. Install the WireGuard client from [wireguard.com/install](https://www.wireguard.com/install/).
2. Copy the generated `.conf` file from the server to your machine (e.g. via `scp`):

```bash
scp root@<YOUR_SERVER_IP>:~/wg0-client-<name>.conf ~/
```

3. Open the WireGuard client, click **Import tunnel(s) from file**, and select the `.conf` file.
4. Click **Activate**. You are connected.

> **Note:** Ubuntu Desktop does not include an SSH server by default. Only `openssh-client` is pre-installed. If you want to make your desktop machine accessible over the VPN via SSH, install the server manually:
> 
> ```bash
> sudo apt install openssh-server
> sudo systemctl enable ssh
> sudo systemctl start ssh
> ```
> 
> On Ubuntu, the SSH service name is `ssh`, not `sshd`.

---

## References

- [Ubuntu Server Docs — WireGuard VPN](https://ubuntu.com/server/docs/explanation/intro-to/wireguard-vpn/)
- [Ubuntu Server Docs — OpenSSH Server](https://ubuntu.com/server/docs/how-to/security/openssh-server/)
- [angristan/wireguard-install](https://github.com/angristan/wireguard-install)
