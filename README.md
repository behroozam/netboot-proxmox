# netboot-proxmox

A minimal Docker Compose setup to PXE boot and install **Proxmox VE** over the network — no USB drive required.

It combines [dnsmasq](https://thekelleys.org.uk/dnsmasq/doc.html) (as a proxyDHCP server) with [netboot.xyz](https://netboot.xyz) (for TFTP + asset serving), giving you a clean and reusable PXE boot environment on any Linux host.

> 📖 Full write-up: [Installing Proxmox via PXE Boot — When Your USB Drive Dies](https://your-blog-url-here)

---

## How It Works

```
PXE Client
    │
    ├─► DHCP Discovery ──► Your Router (assigns IP)
    │                   └► dnsmasq (proxyDHCP, injects boot parameters)
    │
    ├─► TFTP Request ────► netboot.xyz (serves vmlinuz + initrd)
    │
    └─► HTTP Request ────► netboot.xyz / Nginx (serves Proxmox ISO assets)
```

dnsmasq runs in **proxyDHCP mode** — it doesn't replace your router's DHCP server, it just intercepts the boot handshake and tells the PXE client where to find the boot files.

---

## Requirements

- A **Linux host** to run Docker Compose on

> ⚠️ **Windows is not supported.** Docker on Windows runs inside a VM, which prevents DHCP Discovery packets from reaching dnsmasq. Use a native Linux machine or VM with bridged networking.

- Docker and Docker Compose
- A wired ethernet connection on the target machine (PXE boot over WiFi is not supported on most BIOS firmware)

---

## Quick Start

**1. Clone the repo:**
```bash
git clone https://github.com/behroozam/netboot-proxmox.git
cd netboot-proxmox
```

**2. Prepare the Proxmox boot files:**

Mount the Proxmox ISO and copy out the two boot files:
```bash
mount -o loop proxmox-ve_9.1-1.iso /mnt
mkdir initramfs_build
cd initramfs_build
cp /mnt/boot/initrd.img .
zstd -dc initrd.img| cpio -idmv
cp ~/Dwonload/proxmox-ve_9.1-1.iso .
find . | cpio -o -H newc | zstd -19 -T0 > initrd-iso.img
cd ..
mkdir -p assets/proxmox/9.1-1/
cp initramfs_build/initrd-iso.img assets/proxmox/9.1-1/initrd.img
cp /mnt/boot/linux26   assets/proxmox/9.1-1/
umount /mnt
```

> If your target machine's NIC driver isn't included in the stock `initrd.img`, see [Injecting a Missing Driver](#injecting-a-missing-driver) below.

**3. Start the stack:**
```bash
docker-compose up -d
```

**4. Boot your target machine via PXE** (wired ethernet, LAN boot enabled in BIOS).

The netboot.xyz web UI is available at `http://<host-ip>:3000` for monitoring and configuration.

---

## Asset Directory Structure

```
assets/
└── proxmox/
    └── 9.1-1/
        ├── initrd.img   ← from the Proxmox ISO /boot/
        └── linux26      ← from the Proxmox ISO /boot/
```

For other Proxmox versions, just create a new subdirectory following the same pattern (e.g. `assets/proxmox/8.4-1/`).

---

## Ports

| Port | Protocol | Service |
|-----:|--------:|-------:|
| 3000 | TCP | netboot.xyz Web UI |
| 69 | UDP | TFTP |
| 4000 | TCP | Asset server (Nginx) |

---

## Injecting a Missing Driver

If your target machine's NIC driver isn't present in the stock Proxmox `initramfs` (e.g. `e1000e` on a ThinkPad P14s Gen3), you'll need to inject it manually before placing the file in `assets/`.

**1. Download the Proxmox kernel package:**
```bash
wget http://download.proxmox.com/debian/pve/dists/trixie/pve-no-subscription/binary-amd64/proxmox-kernel-6.17.2-1-pve-signed_6.17.2-1_amd64.deb
```

**2. Extract the driver:**
```bash
ar x proxmox-kernel-6.17.2-1-pve-signed_6.17.2-1_amd64.deb
tar -xvf data.tar.xz
# driver is at: lib/modules/6.17.2-1-pve/kernel/drivers/net/ethernet/intel/e1000e/e1000e.ko
```

**3. Unpack initramfs:**
```bash
mkdir initramfs_build && cd initramfs_build
cp /path/to/initrd.img .
zstd -dc initrd.img | cpio -idmv
```

**4. Inject the driver and repack:**
```bash
mkdir -p lib/modules/6.17.2-1-pve/kernel/drivers/net/ethernet/intel/e1000e/
cp ../lib/modules/6.17.2-1-pve/kernel/drivers/net/ethernet/intel/e1000e/e1000e.ko \
   lib/modules/6.17.2-1-pve/kernel/drivers/net/ethernet/intel/e1000e/

find . | cpio -o -H newc | zstd -19 -T0 > ../initrd.img
```

Then place the resulting `initrd.img` into `assets/proxmox/9.1-1/` as usual.

---

## Configuration

| File | Purpose |
|-----:|--------:|
| `dnsmasq.conf` | proxyDHCP settings — update the network interface if needed |
| `config/` | netboot.xyz menu and boot configuration |
| `docker-compose.yaml` | Service definitions |

---

## Troubleshooting

**PXE client gets an IP but doesn't see boot options**
Make sure dnsmasq is binding to the correct network interface in `dnsmasq.conf`.

**Target machine shows no network interface during install**
Your NIC driver is missing from `initramfs`. Follow the [Injecting a Missing Driver](#injecting-a-missing-driver) steps above.

**DHCP Discovery not reaching dnsmasq**
You're likely running Docker on Windows or macOS. Use a native Linux host with `network_mode: host`.

---

## References

- [netboot.xyz documentation](https://netboot.xyz/docs/)
- [dockurr/dnsmasq on Docker Hub](https://hub.docker.com/r/dockurr/dnsmasq)
- [Proxmox VE Downloads](https://www.proxmox.com/en/downloads)
