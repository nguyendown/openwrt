# NR3053 (MT7981) OpenWrt

Unofficial OpenWrt port for the SDMC NR3053.

## Download

```bash
mkdir -p /tmp/sdmc-nr3053-firmware
curl -fL -o /tmp/sdmc-nr3053-firmware.tar.gz \
  https://github.com/nguyendown/openwrt/releases/latest/download/sdmc-nr3053-firmware.tar.gz
tar xzf /tmp/sdmc-nr3053-firmware.tar.gz -C /tmp/sdmc-nr3053-firmware
cd /tmp/sdmc-nr3053-firmware
sha256sum -c --ignore-missing sha256sums
```

Files used below:

| File | Purpose |
|---|---|
| `openwrt-mediatek-filogic-sdmc_nr3053-ram-preloader.bin` | BL2 RAM payload for `mtk_uartboot` |
| `openwrt-mediatek-filogic-sdmc_nr3053-preloader.bin` | BL2 written to NAND |
| `openwrt-mediatek-filogic-sdmc_nr3053-bl31-uboot.fip` | BL31 + U-Boot (FIP) |
| `openwrt-mediatek-filogic-sdmc_nr3053-initramfs-recovery.itb` | Recovery initramfs |
| `openwrt-mediatek-filogic-sdmc_nr3053-squashfs-sysupgrade.itb` | Production image |

## Host setup

Set the host to `192.168.1.254/24` on the Ethernet interface; U-Boot uses
`192.168.1.1` and expects the TFTP server at `192.168.1.254`.

```bash
ETH_IFACE=eno1   # adjust to your interface

# Serial access (temporary; re-run after replugging the adapter)
sudo setfacl -m u:$USER:rw /dev/ttyUSB0

# Static IP for the U-Boot TFTP network
sudo ip addr replace 192.168.1.254/24 dev "$ETH_IFACE"
sudo ip link set "$ETH_IFACE" up

# Firewall (Fedora/RHEL; runtime only)
sudo firewall-cmd --add-service=tftp
sudo chcon -R -t tftpdir_t /tmp/sdmc-nr3053-firmware
sudo chmod -R a+rX /tmp/sdmc-nr3053-firmware

# TFTP server (stop any other dnsmasq/TFTP server on port 69 first)
sudo dnsmasq --no-daemon --port=0 --interface="$ETH_IFACE" \
  --bind-interfaces --listen-address=192.168.1.254 --enable-tftp \
  --tftp-root=/tmp/sdmc-nr3053-firmware --log-facility=-
```

## Boot U-Boot via UART

1. Power off the router and close `picocom` if it is running.
2. Start `mtk_uartboot`; it waits for the BootROM:

```bash
mtk_uartboot -s /dev/ttyUSB0 \
  -p /tmp/sdmc-nr3053-firmware/openwrt-mediatek-filogic-sdmc_nr3053-ram-preloader.bin \
  --aarch64 \
  -f /tmp/sdmc-nr3053-firmware/openwrt-mediatek-filogic-sdmc_nr3053-bl31-uboot.fip
```

3. Power on the router.
4. `mtk_uartboot` prints `FIP sent.`, waits for `Received FIP`, then exits.
   Attach `picocom` immediately:

```bash
picocom -b 115200 /dev/ttyUSB0
```

`bootmenu` appears with a 3-second countdown, select an option or press a key
before it expires. If you miss the window, U-Boot falls back to TFTP boot;
power-cycle and try again. If you land at the `MT7981>`, force a menu with:

```text
MT7981> bootmenu 10
```

## Flash to NAND

Select these entries in order. After each write, press **Enter** to return to
the menu.

| Step | Menu | Entry | File |
|---|---|---|---|
| 1 | `8` | Load BL2 preloader via TFTP then write to NAND | `...-preloader.bin` |
| 2 | `7` | Load BL31+U-Boot FIP via TFTP then write to NAND | `...-bl31-uboot.fip` |
| 3 | `6` | Load recovery system via TFTP then write to NAND | `...-initramfs-recovery.itb` |
| 4 | `5` | Load production system via TFTP then write to NAND | `...-squashfs-sysupgrade.itb` |

Notes:

- Menu `7` runs `reset_factory`, which clears the U-Boot environment.
- Menus `5` and `6` remove the `rootfs_data` UBI volume, so any saved
  configuration is lost. This is expected for a first install.
- Do not power off during steps 1–2. An interrupted BL2 or FIP write leaves the
  device unbootable until it is recovered over UART.

## Recovery

The bootloader included in these releases has **no built-in web failsafe**
— recovery is **TFTP-only**.

### Recovery via TFTP

1. Set the host to `192.168.1.254/24` and put this file in the TFTP root:
   `openwrt-mediatek-filogic-sdmc_nr3053-initramfs-recovery.itb`
   ([Host setup](#host-setup)).
2. Connect a LAN port, hold **Reset**, and power on. The bootloader downloads
   the recovery image over TFTP and boots it.
3. Once the recovery system is up, open `http://192.168.1.1/` and install
   `openwrt-mediatek-filogic-sdmc_nr3053-squashfs-sysupgrade.itb` via LuCI
   (**System → Backup / Flash Firmware**). Note that this page is served by the
   booted recovery system, not by the bootloader itself.

### Web failsafe

Web failsafe is available with a different bootloader,
[quytttb/bl-mt798x-dhcpd](https://github.com/quytttb/bl-mt798x-dhcpd), which
serves a recovery page at `http://192.168.1.1/` and is compatible with this
OpenWrt port.

## Update

### Update OpenWrt

Use LuCI **System → Backup / Flash Firmware**, or `sysupgrade`:

```bash
sysupgrade /tmp/openwrt-mediatek-filogic-sdmc_nr3053-squashfs-sysupgrade.itb
```

### Update BL2 / FIP

The Linux device tree marks `BL2` and `FIP` read-only, so `mtd write` from
OpenWrt fails with:

```text
Could not open mtd device: FIP
```

`mtd unlock` only clears a flash lock, and `mtd -f` only skips TRX validation;
neither bypasses the read-only partition flag. Use U-Boot instead:

1. Reboot and press a key during the 3-second bootmenu countdown.
2. Menu `8` → `...-preloader.bin`
3. Menu `7` → `...-bl31-uboot.fip`

Menu `7` also clears the U-Boot environment. Do not power off during these
writes.

## Partition layout

| Partition | Offset | Size | Notes |
|---|---|---|---|
| `BL2` | `0x000000` | 1 MiB | Read-only; BL2 preloader |
| `u-boot-env` | `0x100000` | 1 MiB | Read-only; stock env. OpenWrt U-Boot uses the UBI `ubootenv`/`ubootenv2` volumes |
| `Factory` | `0x200000` | 2 MiB | Read-only; WiFi calibration and MAC addresses. **Never write.** Back it up |
| `FIP` | `0x400000` | 2 MiB | Read-only; BL31 + U-Boot |
| `ubi` | `0x600000` | 234 MiB | UBI volumes: `ubootenv`, `ubootenv2`, `fit`, `recovery`, `rootfs_data` |

- `fit` — production image.
- `recovery` — recovery initramfs; menu `4` boots it.
- `rootfs_data` — writable overlay/config; created when the production system
  boots and removed by menus `5`/`6`.
- `ubootenv`/`ubootenv2` — redundant U-Boot environment.

## Troubleshooting

- **Serial permission denied:** re-run `setfacl` after replugging the adapter,
  or use the correct `/dev/ttyUSB*`/`/dev/ttyACM*` device.
- **`mtk_uartboot` gets no response:** check GND/TX/RX (crossed), use a 3.3 V
  adapter, power-cycle the router, and try lower baud rates, e.g.
  `--brom-load-baudrate 115200 --bl2-load-baudrate 115200`.
- **TFTP timeouts:** verify the host IP `192.168.1.254`, the interface name,
  the firewall rule, and the `dnsmasq` log. Make sure no other TFTP server is
  using port 69.
- **Bootmenu not shown:** run `bootmenu 10` at the `MT7981>` prompt.
- **`mtd write` fails for `BL2`/`FIP`:** expected; use the U-Boot bootmenu
  (menu `8`/`7`).
