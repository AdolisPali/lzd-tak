# OpenTAKServer

## Install

```bash
bash ubuntu_installer.sh
```
Run as a normal sudo user, not root.

## Issue new certificate

```bash
su - audriuspal -c '
  source ~/.opentakserver_venv/bin/activate
  cd ~/.opentakserver_venv/lib/python3.*/site-packages/opentakserver
  flask ots issue-certificate <username>
'
```
`<username>` must already exist as an OTS user. Output: `~/ots/ca/certs/<username>/<username>.p12` (password `atakatak`).

## Logs

```bash
tail -f ~/ots/logs/opentakserver.log
```

## Ports

| Port | Use |
|---|---|
| 443 | Web UI |
| 8443 | Marti API (client cert required) |
| 8446 | Certificate enrollment |
| 8089 | CoT streaming (SSL) |
| 8088 | CoT streaming (plain TCP) |
| 8883 | RabbitMQ (SSL) |
