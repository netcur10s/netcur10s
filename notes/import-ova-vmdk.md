# Importing OVA/VMDK to Proxmox

> This guide explains how to import OVA/VMDK files into Proxmox when using ZFS-backed 
> storage. The imported files will be saved directly into the ZFS dataset designated for 
> VM storage, rather than `/var/lib/vz/`, which is used for directory-based storage.

> **Note:** This guide is specifically for Proxmox installations using ZFS-backed storage. 
> If your storage type is directory-based (local), the import process differs.

## Step 1: Identify Your ZFS Pool Storage Name

Run this on your Proxmox host:

```bash
zfs list
```

You might see something like:

```
NAME        USED   AVAIL  REFER  MOUNTPOINT
rpool       2.56G  100G   96K    /rpool
rpool/data  2.56G  100G   96K    /rpool/data
```

Or check the Proxmox Web UI:

- Datacenter → Storage
- Look for a storage of **type: ZFS** (e.g., rpool, tank, etc.)

## Step 2: Create a New VM with No Disk

In the **Proxmox Web UI:**

- Click **Create VM**
- Configure:
  - Name: your VM name
  - Do **not** use ISO
  - Do **not** use Hard Disk
  - Finish VM creation

Note the **VM ID** (e.g., 105).

## Step 3: Convert VMDK/OVA to Raw or QCOW2 (if not done already)

```bash
qemu-img convert -f vmdk -O raw .vmdk .raw
```

ZFS prefers raw disk images for performance, but qcow2 also works.

## Step 4: Import the Disk to ZFS Properly

Use `qm importdisk`:

```bash
qm importdisk  .raw 
```

Example:

```bash
qm importdisk 105 .raw rpool
```

This will create a ZFS volume like:

```
<zfs-storage-location>/vm-<vmid>-disk-0
```

## Step 5: Attach the Imported Disk to VM

In the **Proxmox Web UI:**

1. Go to VM 105 → **Hardware**
2. Click **Add** → **Hard Disk**
3. Choose:
   - **Bus/Device:** IDE (older OS like Kioptrix may expect this)
   - **Storage:** your ZFS storage location
   - **Disk:** vm--disk-0
4. Click **Add**

## Step 6: Add Disk to Boot Order

1. Select options
2. Add disk to boot order
3. Make sure first item is vm--disk-0

## Step 7: Boot the VM

Start the VM → Open **Console** → You should see VM boot.
