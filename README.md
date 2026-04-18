# Bypassing AT&T's BGW320-500 Gateway with UDM Pro Max Using wpa_supplicant

This guide details the process for bypassing the AT&T BGW320-500 gateway and replacing it with a Unifi Dream Machine Pro Max using a `wpa_supplicant` bypass method via an ODI DFP-34X-2C2 (RTL9601D) SFP ONU stick.

This is specifically tailored for the **UDM Pro Max** and draws heavily from [drifterxe's UCG Fiber guide](https://github.com/drifterxe/Bypassing-AT-T-s-BGW320-500-Gateway-with-Unifi-Cloud-Gateway-UCG-Fiber-Using-wpa_supplicant) and [evie-lau's Unifi gateway wpa_supplicant guide](https://github.com/evie-lau/Unifi-gateway-wpa-supplicant). Credit to the 8311 Discord community for the foundational work.

---

## ⚠️ Important: Stick Compatibility on UDM Pro Max

**Not all GPON ONU sticks work on the UDM Pro Max.** The UDM Pro Max uses a DesignWare I2C controller that is **hardware-incompatible** with Lantiq/Falcon-based sticks, including:

- FS.com GPON-ONU-34-20BI

This stick uses a software-emulated EEPROM over I2C. When the UDM Pro Max polls the SFP cage, the Lantiq SoC holds the I2C bus low instead of NACKing cleanly, causing the `i2c_designware fd880000.i2c-pld` controller to lock up. This takes down **all** SFP ports and requires a full power cycle to recover. There is no firmware fix — it is a hardware-level incompatibility.

**Use the ODI DFP-34X-2C2 (RTL9601D chipset).** It has a real EEPROM, no I2C issues, and works reliably on UDM Pro Max hardware.

---

## Hardware

- **UDM Pro Max** — WAN SFP+ port is `eth9`
- **ODI DFP-34X-2C2** SFP ONU stick (sold as Luleey LL-XS2510 or similar)
- **iszo media converter** (for initial stick configuration without the UDM)
- AT&T BGW320-500 (to extract certificates)

---

## Resources

- [PON Madness - AT&T GPON ONT Cloning/Bypass](https://docs.google.com/document/d/1gcT0sJKLmV816LK0lROCoywk9lXbPQ7l_4jhzGIgoTo/) — firmware and cert extraction
- [Unifi-gateway-wpa-supplicant by Evie Lau](https://github.com/evie-lau/Unifi-gateway-wpa-supplicant) — wpa_supplicant service setup
- [Certs Extraction Repo by 0x888e](https://github.com/0x888e/certs) — certificate extraction tool
- [drifterxe's UCG Fiber guide](https://github.com/drifterxe/Bypassing-AT-T-s-BGW320-500-Gateway-with-Unifi-Cloud-Gateway-UCG-Fiber-Using-wpa_supplicant) — basis for this guide

---

## Step 1 — Extract Certificates from BGW320-500

Follow the [certs extraction guide](https://github.com/0x888e/certs) to extract the EAP-TLS certificates from your BGW320-500. The repo includes a script that automates the extraction process. You will need:

- `ca.pem` — AT&T root CA / intermediate chain
- `client.pem` — client certificate
- `client.key` — client private key (PKCS#1 or PKCS#8)

Note that newer BGW320 firmware (6.x+) requires downgrading the BGW to firmware 3.18.1 before extraction. See the 0x888e/certs README for the current downgrade path and community firmware mirrors. BGW210 device certs also work with the BGW320 GPON circuit if you still have access to an old BGW210.

**Strongly recommended:** back up the extracted certs to multiple locations (NAS, password manager, off-site). Re-extraction on newer BGW firmware is a multi-hour process.

---

## Step 2 — Configure the ONU Stick

Connect the DFP-34X-2C2 to your Mac/PC via the iszo media converter. Add a secondary IP to your NIC in the `192.168.1.0/24` range and access the stick's web UI at `http://192.168.1.1`.

SSH access (legacy ciphers required):

```bash
ssh -oKexAlgorithms=+diffie-hellman-group1-sha1 \
    -oCiphers=+3des-cbc \
    -oHostKeyAlgorithms=+ssh-rsa \
    admin@192.168.1.1
```

> **PuTTY users:** The flags above are OpenSSH command-line options and won't work directly in PuTTY. Instead, enable the equivalent settings in PuTTY's GUI: **Connection → SSH → Kex** (move `Diffie-Hellman group 1` to the top), **Connection → SSH → Cipher** (enable `3DES`), and **Connection → SSH → Host Keys** (enable `RSA`). Windows users on OpenSSH (PowerShell/CMD) can use the command above as-is.

Configure the stick to impersonate your BGW320-500:

```
flash set GPON_SN HUMA<8 hex chars from BGW serial>
flash set PON_VENDOR_ID HUMA
flash set GPON_ONU_MODEL iONT320500G
flash set HW_HWVER BGW320-500_2.1
flash set OMCI_SW_VER1 BGW320_6.30.8
flash set OMCI_SW_VER2 BGW320_6.30.8
flash set OMCI_OLT_MODE 1
flash set OMCI_FAKE_OK 1
```

Verify all values then reboot the stick:

```
flash get GPON_SN
flash get PON_VENDOR_ID
flash get GPON_ONU_MODEL
flash get HW_HWVER
flash get OMCI_SW_VER1
flash get OMCI_SW_VER2
flash get OMCI_OLT_MODE
flash get OMCI_FAKE_OK
reboot
```

---

## Step 3 — Install wpasupplicant on UDM Pro Max

UniFi OS does not include `wpasupplicant` by default. Install it via apt **while you still have working internet** (through the BGW, before cutover), and cache the `.deb` files in a persistent location so the `reinstall-wpa.service` (Step 10) can reinstall after firmware updates wipe `/sbin/wpa_supplicant`.

SSH into your UDM:

```bash
# Install wpasupplicant
apt update -y
apt install -y wpasupplicant

# Verify the service template was installed
ls /lib/systemd/system/wpa_supplicant-wired@.service

# Cache packages for firmware-update survival
mkdir -p /etc/wpa_supplicant/packages
cd /etc/wpa_supplicant/packages
wget http://security.debian.org/debian-security/pool/updates/main/w/wpa/wpasupplicant_2.9.0-21+deb11u3_arm64.deb
wget http://ftp.us.debian.org/debian/pool/main/p/pcsc-lite/libpcsclite1_1.9.1-1_arm64.deb
```

> **Note:** Package versions above are for Debian bullseye (UniFi OS 3.x/4.x on UDM Pro Max). If UniFi OS ships a newer Debian base in the future, substitute the latest `arm64` package URLs from `packages.debian.org`.

---

## Step 4 — Copy certs and config to UDM Pro Max

From the machine with your extracted certs:

```bash
# Copy certs (rename as you go)
scp CA_<serial>.pem         root@<UDM_IP>:/etc/wpa_supplicant/ca.pem
scp Client_<serial>.pem     root@<UDM_IP>:/etc/wpa_supplicant/client.pem
scp PrivateKey_PKCS1_<serial>.pem root@<UDM_IP>:/etc/wpa_supplicant/client.key

# Lock down permissions
ssh root@<UDM_IP> "chmod 600 /etc/wpa_supplicant/*.pem /etc/wpa_supplicant/*.key"
```

> **Windows users:** The `ssh` and `scp` commands above work natively in PowerShell and Command Prompt on Windows 10 (1809+) and Windows 11 — no PuTTY required. If OpenSSH Client isn't installed, go to **Settings → System → Optional Features** and add it. PuTTY users will need to run the `chmod` command interactively in their SSH session and use `pscp` instead of `scp` for the file transfers.

In your UDM SSH session, create the wpa_supplicant config file:

```bash
cat > /etc/wpa_supplicant/wpa_supplicant-wired-eth9.0.conf << 'EOF'
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=root
eapol_version=1
ap_scan=0
fast_reauth=1

network={
    key_mgmt=IEEE8021X
    eap=TLS
    identity="anonymous"
    ca_cert="/etc/wpa_supplicant/ca.pem"
    client_cert="/etc/wpa_supplicant/client.pem"
    private_key="/etc/wpa_supplicant/client.key"
    eapol_flags=0
}
EOF
chmod 600 /etc/wpa_supplicant/wpa_supplicant-wired-eth9.0.conf
```

---

## Step 5 — Create the eth9.0 VLAN0 Systemd Drop-in

> **Why eth9.0?** The DFP-34X-2C2's `omci_app` daemon intercepts EAP frames on the raw `eth9` interface internally, so they never reach AT&T's authenticator. Running wpa_supplicant on a VLAN ID 0 subinterface (`eth9.0`) bypasses this interception and allows EAP frames to pass through to the OLT.

Create the drop-in directory and config:

```bash
mkdir -p /etc/systemd/system/wpa_supplicant-wired@eth9.0.service.d

cat > /etc/systemd/system/wpa_supplicant-wired@eth9.0.service.d/vlan0.conf << 'EOF'
[Unit]
After=network-pre.target

[Service]
ExecStartPre=/bin/bash -c 'for i in $(seq 1 30); do ip link show eth9 | grep -q "state UP" && break; sleep 1; done'
ExecStartPre=-/sbin/ip link add link eth9 name eth9.0 type vlan id 0 egress-qos-map 0:0
ExecStartPre=/sbin/ip link set eth9.0 up
ExecStopPost=-/sbin/ip link delete eth9.0
EOF

systemctl daemon-reload
systemctl enable wpa_supplicant-wired@eth9.0
```

> The drop-in creates `eth9.0` at service start and deletes it at service stop. But this alone is **not enough** to survive reboots — see Step 7 for why.

---

## Step 6 — Create `on-boot.service` (if missing)

UniFi OS does not always include an `on-boot.service` to execute scripts in `/data/on_boot.d/`. A factory reset will wipe it if it existed. Check first:

```bash
systemctl status on-boot.service
```

If it returns `Unit on-boot.service could not be found`, create it:

```bash
cat > /etc/systemd/system/on-boot.service << 'EOF'
[Unit]
Description=Run on_boot.d scripts
After=network-pre.target
Wants=network-pre.target

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=/bin/bash -c 'for f in /data/on_boot.d/*.sh; do [ -x "$f" ] && "$f"; done'

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable on-boot.service
```

This service runs the scripts in `/data/on_boot.d/` at every boot, in filename (numeric) order. Steps 7, 8, and 9 depend on it.

---

## Step 7 — Pre-create eth9.0 at boot (CRITICAL for reboot reliability)

The `wpa_supplicant-wired@eth9.0.service` template inherits `Requires=sys-subsystem-net-devices-%i.device` from the base unit. This expands to `sys-subsystem-net-devices-eth9.0.device` — a systemd device unit that only exists once `eth9.0` exists in the kernel.

**At boot, `eth9.0` does not exist yet**, so systemd fails the service with `Dependency failed` *before* the drop-in's `ExecStartPre` ever runs to create it. The drop-in only works when the service is started manually (after `eth9.0` has been interactively created) or restarted — not on a cold boot.

The fix is to pre-create `eth9.0` via an `on_boot.d` script that runs before systemd evaluates the service's dependencies:

```bash
mkdir -p /data/on_boot.d

cat > /data/on_boot.d/10-eth9-vlan0.sh << 'EOF'
#!/bin/sh
# Pre-create eth9.0 at boot so systemd's inherited
# Requires=sys-subsystem-net-devices-eth9.0.device is satisfiable when
# wpa_supplicant-wired@eth9.0.service is evaluated. Without this the service
# fails with "Dependency failed" on every cold boot because systemd evaluates
# Requires= before executing ExecStartPre.
for i in $(seq 1 30); do
    if ip link show eth9 > /dev/null 2>&1; then
        ip link add link eth9 name eth9.0 type vlan id 0 egress-qos-map 0:0 2>/dev/null
        ip link set eth9.0 up 2>/dev/null
        break
    fi
    sleep 2
done
EOF
chmod +x /data/on_boot.d/10-eth9-vlan0.sh
```

The `2>/dev/null` silences the "File exists" error on any re-runs, and the `-` prefix on the drop-in's `ExecStartPre` already tolerates the interface already existing by the time the service starts.

---

## Step 8 — ONU Management IP Boot Script

To access the stick's web UI and SSH while it's installed in the UDM, create `/data/on_boot.d/20-onu-ip.sh`:

```bash
cat > /data/on_boot.d/20-onu-ip.sh << 'EOF'
#!/bin/sh
# Add ONU management IP to eth9 for 192.168.1.1 access
for i in $(seq 1 30); do
    if ip link show eth9 | grep -q "state UP"; then
        ifconfig eth9:2 192.168.1.2 netmask 255.255.255.0 up 2>/dev/null
        if ! iptables -w -t nat -C POSTROUTING -o eth9 -d 192.168.1.1 -j SNAT --to 192.168.1.2 2>/dev/null; then
            iptables -w -t nat -A POSTROUTING -o eth9 -d 192.168.1.1 -j SNAT --to 192.168.1.2
        fi
        break
    fi
    sleep 2
done
EOF
chmod +x /data/on_boot.d/20-onu-ip.sh
```

> **Note:** The `-w` flag on iptables is required. Without it the rule silently fails at boot because UniFi's firewall initialization holds the xtables lock when `on_boot.d` scripts run.

After applying, the stick is reachable at `http://192.168.1.1` and via SSH from the UDM.

---

## Step 9 — ONU Management IP Watchdog

UniFi OS's `ubios-udapi-server` occasionally restarts and flushes interface state, which can cause the `192.168.1.2` alias and iptables SNAT rule created in Step 8 to disappear between reboots. This watchdog detects and restores them automatically every minute.

Create the watchdog script:

```bash
cat > /usr/local/bin/eth9-watchdog.sh << 'EOF'
#!/bin/sh
# Watchdog: restore eth9 ONU management IP and iptables SNAT rule if missing

LOGFILE="/var/log/eth9-watchdog.log"

log() {
    echo "$(date '+%Y-%m-%d %H:%M:%S') $1" >> "$LOGFILE"
}

# Check and restore the 192.168.1.2 alias
if ! ip addr show eth9 | grep -q '192.168.1.2'; then
    log "192.168.1.2 missing from eth9 — restoring alias and SNAT rule"
    ifconfig eth9:2 192.168.1.2 netmask 255.255.255.0 up 2>/dev/null
    if ! iptables -w -t nat -C POSTROUTING -o eth9 -d 192.168.1.1 -j SNAT --to 192.168.1.2 2>/dev/null; then
        iptables -w -t nat -A POSTROUTING -o eth9 -d 192.168.1.1 -j SNAT --to 192.168.1.2
    fi
    log "Restored 192.168.1.2 alias and SNAT rule on eth9"
fi
EOF
chmod +x /usr/local/bin/eth9-watchdog.sh
```

Set up the persistent cron entry. UniFi OS does not persist `crontab -e` across reboots — cron state must be stored in `/data/crontabs/root` and reloaded at boot:

```bash
mkdir -p /data/crontabs

cat > /data/crontabs/root << 'EOF'
* * * * * /usr/local/bin/eth9-watchdog.sh
EOF

chmod 600 /data/crontabs/root
crontab /data/crontabs/root

# Verify it loaded
crontab -l
```

Create the boot script to restore cron on every reboot:

```bash
cat > /data/on_boot.d/30-cron.sh << 'EOF'
#!/bin/sh
# Restore persistent crontab on boot
if [ -f /data/crontabs/root ]; then
    crontab /data/crontabs/root
fi
EOF
chmod +x /data/on_boot.d/30-cron.sh
```

After a minute, verify the watchdog is firing correctly (no output means the alias is present and healthy):

```bash
grep eth9-watchdog /var/log/eth9-watchdog.log
```

> **Graylog / syslog users:** The watchdog logs to `/var/log/eth9-watchdog.log`. If you want a syslog-visible alert for Graylog, add `| logger -t eth9-watchdog` to the `log()` function's `echo` command.

---

## Step 10 — Firmware-update survival service

UniFi firmware updates wipe `apt`-installed packages including `wpasupplicant`. This service detects the missing binary on boot and reinstalls from the `.deb` files cached in Step 3.

```bash
cat > /etc/systemd/system/reinstall-wpa.service << 'EOF'
[Unit]
Description=Reinstall and start/enable wpa_supplicant
AssertPathExistsGlob=/etc/wpa_supplicant/packages/wpasupplicant*arm64.deb
AssertPathExistsGlob=/etc/wpa_supplicant/packages/libpcsclite1*arm64.deb
ConditionPathExists=!/sbin/wpa_supplicant

After=network-online.target
Requires=network-online.target

StartLimitIntervalSec=300
StartLimitBurst=10

[Service]
Type=oneshot
ExecStartPre=/usr/bin/dpkg -Ri /etc/wpa_supplicant/packages
ExecStart=/bin/systemctl start wpa_supplicant-wired@eth9.0
ExecStartPost=/bin/systemctl enable wpa_supplicant-wired@eth9.0

Restart=on-failure
RestartSec=20

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable reinstall-wpa.service
```

---

## Step 11 — Configure UniFi WAN for VLAN 962

In the UniFi dashboard, navigate to **Settings → Internet** and configure the SFP+ WAN (physical port `eth9`):

- **Connection Type:** DHCP
- **VLAN ID:** Enable → **962** (AT&T residential fiber VLAN)
- **IPv6:** DHCPv6-PD (if desired)
- **MTU:** 1500

> **Important:** Unlike Lantiq-based sticks that strip VLAN 962 internally, the DFP-34X-2C2 passes 962-tagged frames through to the UDM. The UDM must be configured to tag WAN traffic with VLAN 962 or DHCP will fail silently.

Apply the change. WAN will be offline until you complete the fiber swap below.

---

## Step 12 — Fiber Swap

1. Unplug fiber from the BGW320. **Wait 30–60 seconds** for AT&T's OLT to drop the BGW's registration — skipping this step causes dual-registration flapping with cloned credentials.
2. Insert the DFP-34X-2C2 into the UDM SFP+ port (`eth9`).
3. Plug fiber into the stick.
4. Watch the logs in one SSH session:
   ```bash
   journalctl -u wpa_supplicant-wired@eth9.0 -f
   ```
5. Watch interface state in a second SSH session:
   ```bash
   watch -n 1 'ip -br addr show eth9.0; ip -br addr show eth9.962; ip route show default'
   ```

You should see `CTRL-EVENT-EAP-SUCCESS` followed by `CTRL-EVENT-CONNECTED` within ~60 seconds, and a public IP on `eth9.962` shortly after via DHCP.

---

## Verification

Everything in place:

```bash
# Service states (all three should say "enabled")
systemctl is-enabled wpa_supplicant-wired@eth9.0 on-boot.service reinstall-wpa.service

# Supplicant should be active
systemctl is-active wpa_supplicant-wired@eth9.0

# Three interfaces should have addresses
ip -br addr show eth9.0        # UP, LOWER_UP
ip -br addr show eth9.962      # has public IPv4 + IPv6
ip addr show eth9 | grep 192.168.1  # management alias

# Cron loaded
crontab -l

# Connectivity
curl -s --max-time 5 ifconfig.me && echo
ping -c 3 1.1.1.1
```

Supplicant bound to `eth9.0`:

```bash
pgrep -af wpa_supplicant
```

Expected (plus the system-wide dbus instance which is unrelated):

```
/sbin/wpa_supplicant -c/etc/wpa_supplicant/wpa_supplicant-wired-eth9.0.conf -Dwired -ieth9.0
```

GPON state on the stick (SSH to 192.168.1.1):

```bash
omcicli ponstate
# Expected: O5
```

---

## Reboot Test

To prove the setup survives a cold boot without intervention:

```bash
shutdown -r +1
```

Wait 2–3 minutes, SSH back in, and check:

```bash
journalctl -u wpa_supplicant-wired@eth9.0 -b --no-pager | head -20
journalctl -u on-boot.service -b --no-pager
ip -br addr show eth9.0 eth9.962
ip addr show eth9 | grep 192.168.1
crontab -l
curl -s --max-time 5 ifconfig.me && echo
```

Expected timeline on a healthy boot:

- T+0s — UDM boot
- T+~15s — `on-boot.service` runs `10-eth9-vlan0.sh` (creates `eth9.0`), `20-onu-ip.sh` (adds management IP + SNAT), and `30-cron.sh` (restores crontab)
- T+~20s — `wpa_supplicant-wired@eth9.0.service` starts cleanly (no "Dependency failed")
- T+~45s — EAP-TLS auth succeeds, DHCP issues WAN IP
- T+~60s — Internet fully restored
- T+~60s — `eth9-watchdog.sh` fires for the first time via cron

If you see `Dependency failed` in the journal, `10-eth9-vlan0.sh` did not run — check it's executable and that `on-boot.service` is enabled.

---

## Troubleshooting

**"Dependency failed" on boot** — `/data/on_boot.d/10-eth9-vlan0.sh` is missing or not executable, or `on-boot.service` isn't enabled. See Steps 6 and 7.

**EAP auth fails on raw eth9** — wpa_supplicant must run on `eth9.0` (VLAN0 subinterface), not `eth9`. The `omci_app` daemon on the DFP-34X-2C2 intercepts EAP frames on the raw interface.

**I2C bus lockup / all SFP ports disappear** — You have a Lantiq-based stick inserted. Replace with the DFP-34X-2C2. See compatibility warning at the top.

**ONU management IP not accessible after reboot** — Check `journalctl -u on-boot.service -b` for `xtables lock` errors. Ensure the `-w` flag is present in `20-onu-ip.sh`. Check `cat /var/log/eth9-watchdog.log` to see if the watchdog has been restoring it.

**ONU management IP disappears mid-session** — `ubios-udapi-server` restarted and flushed interface state. The `eth9-watchdog.sh` cron will restore it within 60 seconds. Check `/var/log/eth9-watchdog.log` for timestamps.

**wpa_supplicant fails to start after firmware update** — `reinstall-wpa.service` should handle this. Check `journalctl -u reinstall-wpa.service` for errors. Verify cached `.debs` still exist in `/etc/wpa_supplicant/packages/`.

**No WAN IP after EAP-SUCCESS** — VLAN 962 is not enabled on the SFP+ WAN in the UniFi UI. See Step 11.

**EAP times out on first auth after cutover** — AT&T's OLT may hold stale registration from the BGW. Pull fiber from the stick, wait 60 seconds, reinsert.

**Cron not running after reboot** — `/data/on_boot.d/30-cron.sh` is missing or not executable. Verify with `crontab -l` after boot; if empty, run `crontab /data/crontabs/root` manually and check `30-cron.sh` permissions.

---

## File list

Final list of files created during this guide:

```
/etc/
├── systemd/system/
│   ├── on-boot.service
│   ├── reinstall-wpa.service
│   └── wpa_supplicant-wired@eth9.0.service.d/
│       └── vlan0.conf
└── wpa_supplicant/
    ├── ca.pem
    ├── client.pem
    ├── client.key
    ├── wpa_supplicant-wired-eth9.0.conf
    └── packages/
        ├── wpasupplicant_*_arm64.deb
        └── libpcsclite1_*_arm64.deb

/usr/local/bin/
└── eth9-watchdog.sh

/data/
├── crontabs/
│   └── root
└── on_boot.d/
    ├── 10-eth9-vlan0.sh
    ├── 20-onu-ip.sh
    └── 30-cron.sh
```

---

## Credits

- [drifterxe](https://github.com/drifterxe) — UCG Fiber bypass guide
- [Evie Lau](https://github.com/evie-lau) — wpa_supplicant service setup pattern and firmware-survival approach
- [0x888e](https://github.com/0x888e) — BGW certs extraction tool
- [rajkosto](https://github.com/rajkosto) — ODI stick modded firmware
- [8311 Discord community](https://8311.us) — foundational AT&T bypass research