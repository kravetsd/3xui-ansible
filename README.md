# 3x-ui Ansible Playbook

Automates the full setup of a [3x-ui](https://github.com/MHSanaei/3x-ui) VPN server on any fresh Linux VM.  
After running, you get a working VLESS Reality inbound and a ready-to-use connection string.

## What it does

1. Installs Docker on the target VM
2. Deploys 3x-ui via Docker Compose with all config on the host filesystem
3. Generates a unique VLESS UUID, Reality X25519 keypair, and shortId on the control node
4. Writes the VLESS Reality inbound directly into the 3x-ui SQLite database
5. Restarts the container to apply the config
6. Prints a `vless://` connection string and a full `xray-client-config.json`

## Requirements

**Control node** (the machine you run Ansible from):

```bash
pip install ansible cryptography
```

**Target VM:**
- Ubuntu/Debian (uses `apt`)
- Root or sudo access
- Ports `8443/tcp` open in firewall (or whatever `vpn_port` you set)

## Quick start

```bash
git clone https://github.com/youruser/3xui-ansible.git
cd 3xui-ansible

# Create your inventory
cp inventory.example.ini inventory.ini
```

Edit `inventory.ini`:

```ini
[vpn_servers]
myserver ansible_host=1.2.3.4 ansible_user=root ansible_password=secret
```

Run the playbook:

```bash
ansible-playbook playbook.yml -i inventory.ini
```

## Variables

All variables are defined in `vars.yml` and can be overridden per host in `inventory.ini` or with `-e` on the command line.

### VPN inbound

| Variable            | Default          | Description                                   |
|---------------------|------------------|-----------------------------------------------|
| `install_dir`       | `/opt/3xui`      | Directory on the target VM for all 3x-ui data |
| `vpn_port`          | `8443`           | Port the VLESS inbound listens on             |
| `vless_sni`         | `www.google.com` | SNI domain for Reality camouflage             |
| `vless_fingerprint` | `chrome`         | TLS fingerprint for Reality                   |
| `client_email`      | `client01`       | Label for the VLESS client in the 3x-ui panel |

### Panel

| Variable          | Default  | Description                                                        |
|-------------------|----------|--------------------------------------------------------------------|
| `panel_port`      | `2053`   | Port the 3x-ui web panel listens on                               |
| `panel_user`      | `admin`  | Admin username for the panel                                       |
| `panel_password`  | `admin`  | Admin password for the panel — **change this**                     |
| `panel_path`      | `/`      | URL path prefix for the panel, e.g. `/mypanel/` hides it from scanners |
| `panel_domain`    | `""`     | Domain for the panel (leave empty for HTTP/IP access)             |
| `panel_cert_file` | `""`     | Absolute path to SSL certificate on the VM (e.g. `/root/cert/fullchain.pem`) |
| `panel_key_file`  | `""`     | Absolute path to SSL private key on the VM (e.g. `/root/cert/privkey.pem`)   |

### Panel HTTPS setup

To enable HTTPS on the panel, copy your certs to the VM, then set the cert variables:

```bash
ansible-playbook playbook.yml -i inventory.ini \
  -e "panel_domain=vpn.example.com \
      panel_cert_file=/root/cert/fullchain.pem \
      panel_key_file=/root/cert/privkey.pem"
```

Or edit them directly in `vars.yml` before running.

### Override example

```bash
ansible-playbook playbook.yml -i inventory.ini \
  -e "vpn_port=8443 panel_user=myadmin panel_password=str0ngpass panel_path=/secret/"
```

## Output

At the end of the playbook run you will see two debug messages:

**1. VLESS connection string** — import directly into any VLESS client (v2rayN, Hiddify, Streisand, etc.) or add manually:

```
vless://UUID@1.2.3.4:8443?type=tcp&encryption=none&security=reality&pbk=PUBLIC_KEY&fp=chrome&sni=www.google.com&sid=SHORT_ID&spx=%2F&flow=xtls-rprx-vision#myserver
```

**2. Subscription URL** — paste into any client that supports 3x-ui subscriptions (Hiddify, v2rayNG, etc.). The URL serves all your inbounds for this client and updates automatically when you add more:

```
http://1.2.3.4:2053/sub/CLIENT_SUB_ID
```

If you configured a domain and SSL cert, it will use `https://` automatically.

**3. xray-core client config** — save as `xray-client-config.json` and run with `xray -c xray-client-config.json`. Exposes a SOCKS5 proxy on `127.0.0.1:1080`.

### Using xray-core on a client machine with no GUI

If the client machine has no direct internet access (only SSH), install xray-core manually and use the printed config:

```bash
# Download xray-core (adjust version/arch as needed)
wget https://github.com/XTLS/Xray-core/releases/latest/download/Xray-linux-64.zip
unzip Xray-linux-64.zip -d /opt/xray

# Save the printed config
nano /opt/xray/xray-client-config.json

# Run
/opt/xray/xray -c /opt/xray/xray-client-config.json
```

To run as a systemd service:

```bash
cat > /etc/systemd/system/xray-client.service <<EOF
[Unit]
Description=xray-core client
After=network.target

[Service]
ExecStart=/opt/xray/xray -c /opt/xray/xray-client-config.json
Restart=always

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now xray-client
```

The SOCKS5 proxy on `127.0.0.1:1080` does not support HTTP(S) directly. If you need HTTP proxy support (e.g. for `curl`, `npm`, or `claude`), install privoxy:

```bash
apt install privoxy

# Add to /etc/privoxy/config:
echo "forward-socks5 / 127.0.0.1:1080 ." >> /etc/privoxy/config
systemctl restart privoxy

# Then export in ~/.zshrc or ~/.bashrc:
export HTTP_PROXY=http://127.0.0.1:8118
export HTTPS_PROXY=http://127.0.0.1:8118
```

## Directory layout on target VM

```
/opt/3xui/
├── docker-compose.yml
├── db/
│   └── x-ui.db        # SQLite DB — source of truth for all inbound/client config
├── cert/              # SSL certs (optional, for panel HTTPS)
└── xray/              # xray-core binary (bind-mounted into container)
```

The `xray/config.json` inside that directory is auto-generated by 3x-ui from the database on every restart — do not edit it directly.

## Re-running the playbook

The playbook is **not fully idempotent** — re-running it generates new crypto (UUID, keypair, shortId) and replaces the existing inbound. This is intentional: each run provisions a fresh server.

If you want to reprovision the same server without changing credentials, pass the existing values explicitly:

```bash
ansible-playbook playbook.yml -i inventory.ini \
  -e "vless_uuid=YOUR_UUID reality_private_key=PRIV reality_public_key=PUB reality_short_id=SID"
```

## Troubleshooting

**Docker Compose command not found**  
Older Docker installs use `docker-compose` (v1). This playbook uses `docker compose` (v2, bundled with modern Docker). Re-run after a fresh Docker install from `get.docker.com` — it includes Compose v2.

**x-ui.db not created after 60s**  
Check container logs: `docker logs 3x-ui`. The image is pulled from `ghcr.io` — ensure the target VM can reach the internet.

**Port already in use**  
If port `8443` is taken, set a different `vpn_port`. Avoid `443` if Telegram MTProto proxy is running on the same machine.

**Connection test**  
After the playbook finishes and xray-core client is running on the client machine:

```bash
curl --proxy socks5://127.0.0.1:1080 https://ifconfig.me
# Should return the VPN server's IP, not your real IP
```
