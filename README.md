# Proxmox 9.1-1 Netboot Installer

This repository provides a containerized PXE/iPXE boot environment specifically configured to support the Proxmox VE 9.1-1 live installation. It uses dnsmasq for DHCP/TFTP and leverages docker-compose for quick deployment.

Inspired by cjlittle/netboot.xyz-proxmox.

📂 Project Structure

.
├── assets/               # Proxmox ISO, kernels, and initrd files
├── config/               # iPXE boot scripts and dnsmasq configurations
├── dnsmasq.conf          # Main network and PXE configuration
├── docker-compose.yaml   # Container orchestration for the netboot stack
└── README.md

🚀 Quick Start

1. Prerequisites
- Docker and Docker Compose installed.
- The Proxmox 9.1-1 ISO or extracted boot files (vmlinuz, initrd).
- A network environment where you can run a DHCP/TFTP server (ensure no conflicts with existing DHCP servers).

2. Prepare Assets
Place your Proxmox boot files within the assets/ directory. Ensure your iPXE scripts in config/ point to these local paths.

3. Configure Network
Edit dnsmasq.conf to match your local network interface and IP range:

# Example snippet
interface=eth0
dhcp-range=192.168.1.50,192.168.1.100,255.255.255.0,1h

4. Deployment
Run the stack using Docker Compose:

docker-compose up -d

🛠 Configuration Details

Docker Compose
The docker-compose.yaml manages the dnsmasq container, mapping the local dnsmasq.conf and assets/ folder so they are accessible to booting clients.

iPXE Booting
The configuration is optimized for Proxmox 9.1-1. When a client boots via PXE, it will:
- Receive an IP via DHCP.
- Chainload to iPXE.
- Fetch the Proxmox kernel and initrd from the assets/ directory via HTTP/TFTP.
- Launch the Proxmox Live Installer.

📝 Customization
- Kernel Parameters: You can modify the iPXE scripts in config/ to add arguments like console=ttyS0 for serial installs or proxmox-start-installer automation.
- Static IPs: Update dnsmasq.conf to assign static leases based on MAC addresses for specific nodes in your cluster.

🤝 Contributing
Feel free to open an issue or submit a pull request if you have optimizations for the Proxmox 9.x boot flow.
