# Deploying the ViewDesk hbbr relay

How to build the forked `hbbr` (which includes the ViewDesk VPN raw-UDP relay on
`21117/udp`) and deploy it to the production relay host. Read this before any hbbr
update — the two "busy" errors below will bite otherwise.

## Server layout

- **Host:** `vpn-server` — public `194.154.58.6`, LAN `192.168.100.197`. SSH as
  `root@192.168.100.197`.
- **Stack:** `hbbs` + `hbbr` + `coturn` via `docker`, restart policy `unless-stopped`.
- **hbbr container:** name `hbbr`, image tagged `rustdesk/rustdesk-server:latest`
  (but the *running binary* is our fork), **host networking** — so host `net.core.*`
  sysctls apply directly to the relay's sockets.
- **The binary is a bind-mount**, not baked into the image:
  - host `/opt/rustdesk/hbbr` → container `/usr/bin/hbbr`
  - host `/root/rustdesk/data` → container `/root`

  So you deploy by replacing the **host** file `/opt/rustdesk/hbbr`, not with
  `docker cp`.

## 1. Build (static musl)

From this repo (Windows dev box with Docker). Remotes: `fork` =
`tlalos/rustdesk-server` (ours — push here), `origin` = upstream `rustdesk/rustdesk-server`
(**never push there**).

```bash
docker run --rm --user root -v "$PWD":/home/rust/src \
  messense/rust-musl-cross:x86_64-musl \
  bash -c "apt-get update && apt-get install -y pkg-config libssl-dev perl make && \
           cargo build --release --bin hbbr --target x86_64-unknown-linux-musl"
# -> target/x86_64-unknown-linux-musl/release/hbbr  (ELF x86-64, static-pie, stripped)
```

(`libssl-dev` + vendored OpenSSL are needed because `openssl-sys` is pulled in
transitively; the musl cross image has no musl libssl to link against.)

## 2. Raise host UDP buffers (once)

The relay requests 16 MB `SO_RCVBUF`/`SO_SNDBUF`, but the kernel clamps to
`net.core.rmem_max`/`wmem_max`. Without this the buffers stay at ~208 KB and an
RDP/bulk burst overflows them and drops datagrams for both paired peers.

```bash
sysctl -w net.core.rmem_max=16777216 net.core.wmem_max=16777216
printf 'net.core.rmem_max=16777216\nnet.core.wmem_max=16777216\n' \
  > /etc/sysctl.d/99-viewdesk-relay.conf   # persist across reboot
```

## 3. Deploy the new binary

You **cannot** `docker cp` onto `/usr/bin/hbbr` (it's a mount point →
`device or resource busy`), and you **cannot** overwrite the host file while it's
executing (`Text file busy`). So: copy up, **stop** the container, replace the
host file, start it. The container has no `chmod`, so set `+x` on the host file.

```bash
# from the dev box:
scp target/x86_64-unknown-linux-musl/release/hbbr root@192.168.100.197:/root/hbbr-new

# on the relay host:
sha256sum /root/hbbr-new                      # sanity-check the transfer
cp /opt/rustdesk/hbbr /opt/rustdesk/hbbr.bak-$(date +%F)   # rollback copy
docker stop hbbr
cp /root/hbbr-new /opt/rustdesk/hbbr
chmod +x /opt/rustdesk/hbbr
docker start hbbr
docker logs hbbr --tail 30 | grep -i udp      # expect: Listening on udp :21117 (VPN relay)
```

Rollback: `cp /opt/rustdesk/hbbr.bak-* /opt/rustdesk/hbbr && docker restart hbbr`.

## 4. Verify under load

```bash
netstat -su | grep -iE 'receive buffer|receive errors'
```
Run before and after a real load test (e.g. an RDP session through the gateway).
The `receive buffer errors` counter should stay **flat**; if it climbs, the
buffer is still too small (recheck step 2 and that the container restarted after
the sysctl).

## Note: RDP over the relay is latency-bound

Even with a clean relay (zero drops), a remote gateway peer's path is a WAN
round-trip (~130 ms in our case). RDP's serial connect handshakes plus that
latency make direct `mstsc` over the relay choke regardless of server tuning.
The working pattern is the **ViewDesk-hop**: remote into the gateway peer with
ViewDesk (WAN-adaptive transport), then run `mstsc` to the target LAN host
locally from that desktop.
