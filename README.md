# 1. Install PostmarketSO using pmbootstrap

```bash
pmbootstrap init
```
Select vendor & device codename

## User interface

I recommend gnome-mobile

## Systemd

Select (type) "always"

## Extra packages

> wireguard-tools,iptables

## Install

```bash
pmbootstrap install
```

## Flash

```bash
pmbootstrap flasher flash_rootfs

pmbootstrap flasher flash_kernel
```

ALWAYS USE `FASTBOOT REBOOT` to ensure all data is written successfully
```bash
fastboot reboot
```

# 2. Networking setup

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

### Test the connection!

```bash
resolvectl query google.com
# OR
ping google.com
```

# 3. Komodo setup (ARM, musl)

> Since pmOS is based on Alpine Linux, it uses the musl C-linker and many of the common binaries for linux distros don't work.

To get around this, we'll just compile Komodos `Periphery` ourselves.

## Install Rust

I chose the nightly channel but I think you don't have to.

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
```

## Build Komodo

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


## Set up Systemd service
/etc/systemd/system/periphery.service
```
[Unit]
Description=Agent to connect with Komodo Core

[Service]
Environment=HOME=/root
ExecStart=/usr/local/bin/periphery --config-path /etc/komodo/periphery.config.toml
Restart=on-failure
TimeoutStartSec=0

[Install]
WantedBy=default.target
```