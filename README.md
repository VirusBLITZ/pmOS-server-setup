# PostmarketOS + WireGuard + Docker + Komodo Setup Guide

## 1. Install PostmarketSO using pmbootstrap

```bash
pmbootstrap init
```

Select vendor & device codename

### User interface

I recommend gnome-mobile

### Systemd

Select (type) "always"

### Extra packages

> wireguard-tools,iptables

### Install

```bash
pmbootstrap install
```

### Flash

```bash
pmbootstrap flasher flash_rootfs

pmbootstrap flasher flash_kernel
```

ALWAYS USE `FASTBOOT REBOOT` to ensure all data is written successfully

```bash
fastboot reboot
```

## 2. Networking setup

copy the wireguard profile onto the server.

```bash
user@op6-server ~> sudo cat /etc/wireguard/op6.conf

[Interface]
Address = 10.7.0.2/24
DNS = 1.0.0.1
PrivateKey = QJ8ltkhGqsfV5iUg/uvs1TVFo7G6RnF3vYq7JYMsRVs=

# Force systemd-resolved to route ALL DNS queries through wg0
PostUp = resolvectl domain %i "~."
PostUp = resolvectl default-route %i true

[Peer]
PublicKey = vDYDTimVuR0kFY2lE6j2JziUJ40NlULp4PZRwSwVdnY=
PresharedKey = FbfBTj1dbJfyE7Ew/RRrJC6b85LgmIOX42w1XRB780M=
AllowedIPs = 0.0.0.0/0, ::/0
Endpoint = 172.16.42.2:51820
PersistentKeepalive = 25
```

Make sure systemd is the DNS resolver

```bash
sudo systemctl enable --now systemd-resolved

sudo ln -sf /run/systemd/resolve/stub-resolv.conf /etc/resolv.conf
```

Connect the profile on start-up

op6 = profile name in /etc/wireguard/op6.conf

```bash
sudo systemctl enable wg-quick@op6
sudo systemctl start wg-quick@op6
```

### Make sure the date is correct (for encryption & DNSSEC)

```bash
sudo date -s "2026-03-31 20:15:00"
```

### Test the connection

```bash
resolvectl query google.com
# OR
ping google.com
```

## 3. Docker Install

```bash
sudo apk add docker docker-cli-compose
```

### Fix Systemd service file

On Alpine (PostmarketOS), the `containerd` service is located at `/usr/bin/containerd` but the service file gets created with `/usr/local/bin/containerd` or similar.

Edit the service file and change the `ExecStart` path to `/usr/bin/containerd`:

```bash
sudo nano /usr/lib/systemd/system/containerd.service
```

### Enable and start the service

```bash
sudo systemctl enable --now containerd
sudo systemctl enable --now docker
```

If your containers aren't able to access the network, reboot the device and try again.

### Network test

Test IP Routing:

```bash
docker run --rm alpine ping -c 3 8.8.8.8
```

Test DNS:

```bash
docker run --rm alpine ping -c 3 google.com
```

## 4. Komodo setup (ARM, musl)

> Complete [3. Docker Install] first!

Since pmOS is based on Alpine Linux, it uses the musl C-linker and many of the common binaries for linux distros don't work.

To get around this, we'll just compile Komodos `Periphery` ourselves.

### Install Rust

I chose the nightly channel but I think you don't have to.

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

### Build Komodo

also install gcc etc

```bash
sudo apk add build-base pkgconf openssl openssl-dev openssl-libs-static
```

```bash
git clone https://github.com/moghtech/komodo
```

In the repo, run

```bash
cargo build -p komodo_periphery --release
```

to build only the periphery binary.

Then, copy the built binary to /usr/local/bin

```bash
sudo cp target/release/periphery /usr/local/bin
```

### Set up Systemd service

/etc/systemd/system/periphery.service

```systemd
[Unit]
Description=Agent to connect with Komodo Core

[Service]
Environment=HOME=/root
ExecStart=/bin/sh -lc "/usr/local/bin/periphery --config-path /etc/komodo/periphery.config.toml"
Restart=on-failure
TimeoutStartSec=0

[Install]
WantedBy=default.target
```
