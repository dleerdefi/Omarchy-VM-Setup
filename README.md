# Omarchy VM Setup

KVM/libvirt virtual machine setup for Omarchy (Arch Linux) machines that also run Docker.

## The Problem

Docker and libvirt both manage firewall rules, and they conflict. Docker creates an
nftables `FORWARD` chain with `policy drop` — all forwarded traffic is blocked unless
explicitly allowed. libvirt puts its own forwarding and NAT rules in a separate nftables
table, but Docker's drop policy runs first and silently kills VM internet traffic.

The fix is a small systemd service (`libvirt-docker-fix.service`) that runs after both
Docker and libvirtd start, and adds the two rules needed to punch virbr0 traffic through
Docker's forward chain.

## Prerequisites

- Omarchy installed
- Docker installed and running (`systemctl is-active docker`)

## Setup

```bash
git clone https://github.com/dleerdefi/Omarchy-VM-Setup.git
cd omarchy-vm-setup
sudo bash setup.sh
```

Then **log out and back in** — the kvm and libvirt group memberships won't take effect
until you do.

## Verify

After re-login:

```bash
# Groups should include kvm and libvirt
groups $USER

# Services should be active/enabled
systemctl is-active libvirtd
systemctl is-enabled libvirt-docker-fix
```

## Creating a VM

Open virt-manager and use the GUI, or via CLI:

```bash
virt-install \
  --name my-vm \
  --ram 4096 \
  --vcpus 2 \
  --disk size=40 \
  --cdrom /path/to/image.iso \
  --os-variant detect=on \
  --network network=default \
  --graphics spice
```

The default NAT network (`192.168.122.0/24`) is created automatically by libvirt and
works for all VMs — no per-VM network config needed.

## SSH into VMs (optional)

Standard SSH works over the default NAT network (`ssh user@192.168.122.x`). For
cross-machine access, you can also install Tailscale inside each VM:

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

Approve it at the auth URL, then SSH to it by Tailscale IP from any machine in the network.

## How the fix works

`libvirt-docker-fix.service` runs once after Docker and libvirtd start and adds:

```
nft add rule ip filter DOCKER-USER iifname "virbr0" accept
nft add rule ip filter DOCKER-USER oifname "virbr0" ct state established,related accept
```

`DOCKER-USER` is Docker's dedicated chain for user-defined rules — Docker never flushes
it on restart, so these rules persist across Docker restarts. The service re-applies them
on every system boot.
