# Bypassing AT&T's BGW320-500 Gateway with UDM Pro Max Using wpa_supplicant

This guide details the process for bypassing the AT&T BGW320-500 gateway and replacing it with a Unifi Dream Machine Pro Max using a `wpa_supplicant` bypass method via an ODI DFP-34X-2C2 (RTL9601D) SFP ONU stick.

This is specifically tailored for the **UDM Pro Max** and draws heavily from [drifterxe's UCG Fiber guide](https://github.com/drifterxe/Bypassing-AT-T-s-BGW320-500-Gateway-with-Unifi-Cloud-Gateway-UCG-Fiber-Using-wpa_supplicant). Credit to the 8311 Discord community for the foundational work.

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
- [Unifi-gateway-wpa-supplicant by Evie Lau](https://github.com/evie-lau/unifi-gateway-wpa-supplicant) — wpa_supplicant service setup
- [Certs Extraction Repo by 0x888e](https://github.com/0x888e/certs) — certificate extraction tool
- [drifterxe's UCG Fiber guide](https://github.com/drifterxe/Bypassing-AT-T-s-BGW320-500-Gateway-with-Unifi-Cloud-Gateway-UCG-Fiber-Using-wpa_supplicant) — basis for this guide

---

## Step 1 — Extract Certificates from BGW320-500

Follow the [certs extraction guide](https://github.com/0x888e/certs) to extract the EAP-TLS certificates from your BGW320-500. The repo includes a script that automates the extraction process. You will need:
 
- `ca.pem` — AT&T root CA
- `client.pem` — client certificate
- `client.key` — client private key
 
Copy these to `/etc/wpa_supplicant/` on the UDM Pro Max.

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

Configure the stick to impersonate your BGW320-500:
```bash
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
```bash
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

## Step 3 — Configure wpa_supplicant on UDM Pro Max

Follow the [Unifi-gateway-wpa-supplicant guide by Evie Lau](https://github.com/evie-lau/unifi-gateway-wpa-supplicant) for full details on the wpa_supplicant service setup. The key difference when using the DFP-34X-2C2 on UniFi hardware is using `eth9.0` instead of `eth9` — covered in Step 4.

Create the wpa_supplicant config at `/etc/wpa_supplicant/wpa_supplicant-wired-eth9.0.conf`:

```
ctrl_interface=/var/run/wpa_supplicant
ctrl_interface_group=root
eapol_version=1
ap_scan=0
fast_reauth=1

network={
    key_mgmt=IEEE8021X
    eap=TLS
    identity="xx:xx:xx:xx:xx:xx" # Internet (ONT) interface MAC address must match this value
    ca_cert="/etc/wpa_supplicant/ca.pem"
    client_cert="/etc/wpa_supplicant/client.pem"
    private_key="/etc/wpa_supplicant/client.key"
    eapol_flags=0
}
```

Enable the wpa_supplicant service for eth9.0:
```bash
systemctl enable wpa_supplicant-wired@eth9.0
```

---

## Step 4 — Create the eth9.0 VLAN0 Systemd Drop-in

> **Why eth9.0?** The DFP-34X-2C2's `omci_app` daemon intercepts EAP frames on the raw `eth9` interface internally, so they never reach AT&T's authenticator. Running wpa_supplicant on a VLAN ID 0 subinterface (`eth9.0`) bypasses this interception and allows EAP frames to pass through to the OLT.

Create the drop-in directory and config:
```bash
mkdir -p /etc/systemd/system/wpa_supplicant-wired@eth9.0.service.d
```

Create `/etc/systemd/system/wpa_supplicant-wired@eth9.0.service.d/vlan0.conf`:
```ini
[Unit]
After=network-pre.target

[Service]
ExecStartPre=/bin/bash -c 'for i in $(seq 1 30); do ip link show eth9 | grep -q "state UP" && break; sleep 1; done'
ExecStartPre=-/sbin/ip link add link eth9 name eth9.0 type vlan id 0 egress-qos-map 0:0
ExecStartPre=/sbin/ip link set eth9.0 up
ExecStopPost=-/sbin/ip link delete eth9.0
```

Reload and start:
```bash
systemctl daemon-reload
systemctl start wpa_supplicant-wired@eth9.0
systemctl status wpa_supplicant-wired@eth9.0
```

Check for successful authentication:
```bash
journalctl -u wpa_supplicant-wired@eth9.0 | grep -E "EAP-SUCCESS|CONNECTED|FAILURE"
```

---

## Step 5 — ONU Management IP (Optional but Useful)

To access the stick's web UI and SSH while it's installed in the UDM, create `/data/on_boot.d/20-onu-ip.sh`:

```bash
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
```

```bash
chmod +x /data/on_boot.d/20-onu-ip.sh
```

> **Note:** The `-w` flag on iptables is required. Without it the rule silently fails at boot because UniFi's firewall initialization holds the xtables lock when `on_boot.d` scripts run.

After applying, access the stick at `http://192.168.1.1` or via SSH from the UDM.

---

## Verification

Check wpa_supplicant is running correctly:
```bash
pgrep -a wpa_supplicant
```

Expected output includes a process on `eth9.0`:
```
/sbin/wpa_supplicant -c/etc/wpa_supplicant/wpa_supplicant-wired-eth9.0.conf -Dwired -ieth9.0
```

Check GPON registration on the stick:
```bash
# SSH to the stick at 192.168.1.1
omcicli get state
# Should return: ONU state: 5
```

Check WAN connectivity — after successful EAP-TLS auth, DHCP should acquire a WAN IP on eth9.

---

## Troubleshooting

**EAP auth fails on raw eth9** — wpa_supplicant must run on `eth9.0` (VLAN0 subinterface), not `eth9`. The omci_app daemon on the DFP-34X-2C2 intercepts EAP frames on the raw interface.

**I2C bus lockup / all SFP ports disappear** — You have a Lantiq-based stick inserted. Replace with the DFP-34X-2C2. See compatibility warning at the top of this guide.

**ONU management IP not accessible after reboot** — Run `20-onu-ip.sh` manually and check `systemctl status on-boot.service` for the xtables lock error. Ensure the `-w` flag is present in the iptables commands.

**wpa_supplicant fails to start** — Verify eth9.0 exists (`ip link show eth9.0`) and eth9 is in state UP before the service starts. The `ExecStartPre` wait loop handles this but confirm eth9 comes up at all.