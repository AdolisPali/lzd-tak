# OpenTAKServer on WSL2 (Ubuntu 24.04)

Self-hosted TAK server for ATAK/iTAK/WinTAK, installed via the official Ubuntu installer script.

## What's here

- `OpenTAKServer/` — git clone of the upstream source (**reference only**, not what's actually running — see below)
- `ubuntu_installer_run.sh` — the official installer script (originally from `https://i.opentakserver.io/ubuntu_installer`), lightly edited to skip the interactive ZeroTier/Mumble prompts so it can run unattended. **Kept around for setting up additional machines** — see below.

## Where the actual server lives

The installer does **not** build from the git clone — it installs the pre-built package from PyPI. The real install is:

- `~/.opentakserver_venv` — Python venv with `opentakserver` installed via pip
- `~/ots/` — data directory:
  - `config.yml` — main config (DB creds, ports, feature flags)
  - `ca/` — the certificate authority (see Certificates below)
  - `logs/opentakserver.log` — combined log for all OTS services
  - `mediamtx/` — video streaming server + recordings
  - `icons.sqlite`, `uploads/`

Installed and run as the **`audriuspal`** system user (the installer refuses to run as root by design). Requires `postgresql`, `rabbitmq-server`, `nginx` — all installed as system packages.

## Services (systemd)

| Service | Purpose |
|---|---|
| `opentakserver` | Main Flask app / web UI / REST API |
| `cot_parser` | Parses and routes CoT (Cursor-on-Target) messages via RabbitMQ |
| `eud_handler` | Plain TCP CoT listener, port 8088 |
| `eud_handler_ssl` | SSL/mutual-TLS CoT listener, port 8089 (what ATAK/WinTAK/iTAK connect to) |
| `mediamtx` | Video streaming (RTSP/RTMP/HLS/WebRTC) |

Check status: `systemctl status opentakserver cot_parser eud_handler eud_handler_ssl mediamtx nginx rabbitmq-server postgresql`

## Ports

| Port | Protocol | Purpose |
|---|---|---|
| 443 | HTTPS | Web UI (live map, admin) |
| 8443 | HTTPS + client cert required | Marti REST API (Data Sync, Mission API) |
| 8446 | HTTPS | Certificate auto-enrollment (username/password → signed client cert) |
| 8089 | SSL | CoT streaming — **this is the port ATAK/WinTAK/iTAK connect to** |
| 8088 | TCP (plaintext) | CoT streaming, unencrypted |
| 8080 | HTTP | Marti API, unencrypted (disabled by default — don't enable unless needed) |
| 8883 | SSL | RabbitMQ (MQTT-based clients) |
| 8322 / 1936 | SSL | RTSPS / RTMPS proxy for mediamtx |

## Login

Web UI: `https://localhost/`

Default admin account: **`administrator`** / **`password`**

⚠️ This is a well-known default and the server listens on all interfaces — **change it** via the web UI account settings before exposing this beyond localhost.

## Certificates

Root CA and all issued certs live under `~/ots/ca/`:

```
ca/
├── ca.pem                    # CA root cert (public, share for trust)
├── truststore-root.p12       # CA cert in PKCS12 form (password: atakatak) — needed by WinTAK/ATAK, plain .pem is NOT accepted for their CA truststore field
├── ca-do-not-share.key       # CA private key — keep this on the server only
└── certs/
    ├── opentakserver/        # server's own TLS cert (nginx, mediamtx, rabbitmq, eud_handler)
    ├── wintak/                # client cert issued for the WinTAK connection
    └── simulator/             # client cert generated for a movement-simulator script (incomplete — no matching OTS user created yet)
```

Copies for Windows are in `C:\Users\AudriusPal\Downloads\ots_certs\` (`ca.pem`, `truststore-root.p12`, `wintak.p12`).

### Issuing a new client certificate

**Important:** the cert's Common Name (CN) must exactly match an existing OTS username, or the server will silently drop the connection after the TLS handshake (this bit us once — see Known Issues).

```bash
# 1. Create the user first (via web UI: Users → Add User), e.g. "john"
# 2. Issue a cert with a matching CN:
su - audriuspal -c '
  source ~/.opentakserver_venv/bin/activate
  cd ~/.opentakserver_venv/lib/python3.*/site-packages/opentakserver
  flask ots issue-certificate john
'
# Produces ~/ots/ca/certs/john/john.p12 (import password: atakatak)
```

Alternative — **enrollment flow** (no CLI needed): create the user in the web UI, then in WinTAK/ATAK/iTAK use "Quick Connect"/certificate enrollment against port `8446` with that username/password. The app fetches a correctly-matched cert automatically.

## Connecting WinTAK (same Windows PC as this WSL instance)

WSL2 forwards `localhost` automatically, but **the server cert's SAN only covers the hostname `opentakserver`**, not `localhost` — WinTAK does strict hostname verification and will silently disconnect if the names don't match.

1. Add to Windows hosts file (`C:\Windows\System32\drivers\etc\hosts`, edit as Administrator):
   ```
   127.0.0.1 opentakserver
   ```
2. In WinTAK, add a server:
   - Host address: `opentakserver`
   - Port: `8089`, protocol: `SSL`
   - Secure Server API Port: `8443`
   - Unsecure Server API Port: `8080`
   - Certificate Enrollment API Port: `8446`
   - Install Certificate Authority → `truststore-root.p12` (password `atakatak`) — **not** `ca.pem`, WinTAK rejects plain PEM here
   - Install Client Certificate → `wintak.p12` (password `atakatak`)
3. Connect.

## Known issues hit during setup

- `cot_parser.service` can race RabbitMQ's plugin reconfiguration at the end of the install and exit immediately after starting. Fix: `systemctl restart cot_parser`.
- `flask ots issue-certificate <cn>` throws `RuntimeError: Working outside of request context` at the very end — this is a harmless bug in a secondary "build a convenience zip" step; the actual cert files are generated successfully before it hits that error.
- Every disconnect (clean or not) logs a cosmetic `pika.exceptions.ChannelWrongStateError` from `EudHandler.close_connection()` — a known messy code path in OTS itself, not a sign of a real problem.
- WinTAK client certs must have a CN matching an existing OTS username — certs issued via the raw CLI command are **not** automatically tied to a user account.

## Setting up another computer

Repeat these steps on the new machine. Assumes Ubuntu 24.04 (native or WSL2) with a normal (non-root) sudo-capable user already set up.

### 1. Get the files

```bash
git clone https://github.com/brian7704/OpenTAKServer.git
```
This is only for reference/source browsing — the actual install comes from PyPI via the script below, not from this clone.

Copy `ubuntu_installer_run.sh` from this machine to the new one (e.g. `scp`, a USB drive, or just re-download the original with `curl https://i.opentakserver.io/ubuntu_installer -Ls -o ubuntu_installer_run.sh` and re-apply the two edits described below if you want it interactive-safe).

### 2. Run the installer

**Must run as a normal user, not root** — the script refuses to run as root by design (OTS services run as whatever user launches this).

- **Interactive terminal available** (you're physically at the machine or in a normal SSH session): just run the original script directly, answering prompts as they appear:
  ```bash
  curl https://i.opentakserver.io/ubuntu_installer -Ls | bash -
  ```
- **No interactive terminal** (e.g. running through an automation tool with no TTY, like this session was): use `ubuntu_installer_run.sh` instead, which hardcodes "no" to the ZeroTier and Mumble prompts. You'll also need sudo to work without a password prompt for the duration of the run — either run it from a shell where you've already primed `sudo -v` and know you won't hit a timeout, or temporarily grant NOPASSWD sudo and revoke it right after:
  ```bash
  echo "yourusername ALL=(ALL) NOPASSWD: ALL" | sudo tee /etc/sudoers.d/90-ots-temp-install
  sudo chmod 0440 /etc/sudoers.d/90-ots-temp-install
  bash ubuntu_installer_run.sh
  sudo rm -f /etc/sudoers.d/90-ots-temp-install   # revoke immediately after
  ```

The script installs postgresql-postgis, rabbitmq-server, nginx, creates the venv, sets up the database, generates a fresh CA, configures nginx/systemd/RabbitMQ, and starts everything. Takes several minutes (mostly `apt upgrade`).

### 3. Verify it worked

```bash
systemctl is-active opentakserver cot_parser eud_handler eud_handler_ssl mediamtx nginx rabbitmq-server postgresql
```
If `cot_parser` shows `inactive`, it likely hit the startup race described in Known Issues — just `sudo systemctl restart cot_parser`.

```bash
curl -sk -o /dev/null -w "%{http_code}\n" https://localhost/   # expect 200
```

### 4. Change the default password

Log into `https://localhost/` with `administrator`/`password` and change it immediately — same default on every install.

### 5. Set up certificates for clients

Each new machine gets its **own CA** (generated fresh by `flask ots create-ca` during install), so certs from one server will NOT work against another — you must issue new client certs per-server. Follow the "Issuing a new client certificate" section above: create the OTS user first, then run `flask ots issue-certificate <username>` as the service user.

Copy the resulting files to wherever the TAK client (WinTAK/ATAK/iTAK) will import them from:
- `~/ots/ca/truststore-root.p12` (CA trust, password `atakatak`)
- `~/ots/ca/certs/<username>/<username>.p12` (client cert, password `atakatak`)

### 6. Client connection setup

If the TAK client is on the same machine (e.g. WinTAK on the Windows host of a WSL2 install), remember the server cert's SAN only covers the hostname `opentakserver`, not `localhost` — add `127.0.0.1 opentakserver` to that machine's hosts file and connect to `opentakserver`, not `localhost`. See "Connecting WinTAK" above for the full port/cert breakdown.

If the client is on a **different** machine/LAN and this is a WSL2 install, WSL2's default NAT network isn't reachable from outside the host — you'll need either mirrored networking mode (`.wslconfig` → `networkingMode=mirrored`, Windows 11 22H2+) or a `netsh interface portproxy` rule forwarding the relevant ports (8089, 8443, 8446, 443) from the Windows host's LAN IP into the WSL2 VM.

## Useful commands

```bash
# Tail combined log
tail -f ~/ots/logs/opentakserver.log

# Restart everything
sudo systemctl restart opentakserver cot_parser eud_handler eud_handler_ssl mediamtx nginx

# Activate the venv for flask ots CLI commands (must run as audriuspal)
su - audriuspal -c 'source ~/.opentakserver_venv/bin/activate && cd ~/.opentakserver_venv/lib/python3.*/site-packages/opentakserver && flask ots --help'
```
