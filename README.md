# VirtFusion / KVM: `Unable to get index for interface eth0: No such device`

## Symptom

Starting a VM on the hypervisor fails with:

```
error: Failed to create domain from /home/vf-data/server/56ddf10f-6e34-4d1f-af72-0eae77f67a66/server.xml
error: Unable to get index for interface eth0: No such device
```

## Cause

The VM's `server.xml` attaches its network card to a **host** interface named `eth0`,
but no interface called `eth0` exists on the hypervisor. libvirt asks the kernel for the
index of `eth0`, the kernel says "no such device", and the domain is never created.

This almost always happens for one of these reasons:

1. **Predictable NIC naming** – the host NIC is actually named `eno1`, `enp3s0`,
   `ens18`, etc. (the default on modern Debian/Ubuntu/AlmaLinux), but the VirtFusion
   hypervisor/network config says `eth0`.
2. **NIC renamed after a kernel/OS upgrade, reboot or hardware change** – it used to be
   `eth0` and no longer is.
3. **Bridge/macvtap mismatch** – the VM uses `type='direct'` (macvtap) on `eth0`, but the
   host now puts its uplink inside a bridge (`br0`) or the uplink moved to another port.

The VM itself is fine; nothing inside it needs to change.

## Diagnose (read-only)

```bash
# 1. What interfaces really exist on the host?
ip -br link
ip -br addr

# 2. What does the VM expect?
grep -A8 '<interface' /home/vf-data/server/56ddf10f-6e34-4d1f-af72-0eae77f67a66/server.xml

# 3. Any bridges on the host?
bridge link show 2>/dev/null || brctl show
```

Typical result: the XML shows

```xml
<interface type='direct'>
  <source dev='eth0' mode='bridge'/>
```

while `ip -br link` shows the uplink as e.g. `eno1` (or `br0` holding `eno1`).

## Fix

### Option A – correct the interface in VirtFusion (recommended)

VirtFusion regenerates `server.xml` from its own database on every boot, so hand-editing
the XML is overwritten. Fix it at the source:

1. VirtFusion panel → **Compute → Hypervisors → (this hypervisor) → Network** (or the
   hypervisor's network/IP block settings).
2. Change the interface/bridge from `eth0` to the real name found above
   (e.g. `eno1`, or `br0` if the host uses a bridge).
3. Save, then **boot the server** again from the panel. VirtFusion writes a new
   `server.xml` with the correct device.

### Option B – make the host NIC be called `eth0` again

Only do this if every VM on the hypervisor expects `eth0`. Pin the name with a systemd
link file (replace the MAC with the uplink's MAC from `ip link`):

```bash
cat > /etc/systemd/network/10-eth0.link <<'EOF'
[Match]
MACAddress=aa:bb:cc:dd:ee:ff

[Link]
Name=eth0
EOF
update-initramfs -u   # Debian/Ubuntu  (dracut -f on RHEL/Alma)
```

Also update `/etc/network/interfaces` or netplan/NetworkManager config to use `eth0`,
then reboot the hypervisor. **This reboots all VMs on the host and can cut network access
if the host network config is not updated too — have out-of-band (IPMI/KVM) access ready.**

### Option C – one-off manual start (temporary)

```bash
cp /home/vf-data/server/56ddf10f-6e34-4d1f-af72-0eae77f67a66/server.xml /root/server.xml.bak
sed -i "s/dev='eth0'/dev='eno1'/" /home/vf-data/server/56ddf10f-6e34-4d1f-af72-0eae77f67a66/server.xml
virsh create /home/vf-data/server/56ddf10f-6e34-4d1f-af72-0eae77f67a66/server.xml
```

Replace `eno1` with the real interface. The next boot from the panel will regenerate the
XML with `eth0` again unless Option A is also done.

## Verify

```bash
virsh list --all | grep 56ddf10f-6e34-4d1f-af72-0eae77f67a66   # should be "running"
virsh domiflist 56ddf10f-6e34-4d1f-af72-0eae77f67a66            # source = real host NIC
```

Then ping the VM's IP / connect over RDP or SSH.

## Other fixes in this repo's history

- **`Failed to reserve port 59xx`** – the VM's VNC port is already bound, usually by a
  stale `qemu` process from a crashed run. Confirm the port in `server.xml`
  (`grep '<graphics' .../server.xml`), find the holder with `ss -tlnp | grep <port>`,
  and if it is an orphaned process (not another running VM) `kill -9 <pid>`, then boot
  the VM again.
