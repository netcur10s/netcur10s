# Importing ova/vmdk to Proxmox

> This guide explains how to import OVA/VMDK files into Proxmox when using ZFS-backed storage. The imported files will be saved directly into the ZFS dataset designated for VM storage, rather than /var/lib/vz/, which is used for directory-based storage.
> 

<br>

## **Step 1: Identify Your ZFS Pool Storage Name**

Run this on your Proxmox host:

```
zfs list
```

You might see something like:

```
NAME               USED  AVAIL  REFER  MOUNTPOINT
rpool              2.56G  100G   96K   /rpool
rpool/data         2.56G  100G   96K   /rpool/data
```

Or check Proxmox Web UI:

- **Datacenter → Storage**
- Look for a storage of **type: ZFS** (e.g., rpool, tank, etc.)

<br>

## **Step 2: Create a New VM with No Disk**

In the **Proxmox Web UI**:

- **Create VM**
    - Name: <vm-name>
    - Do **not** use ISO
    - **Do not use Hard Disk**
    - Finish VM creation

Note the **VM ID** (e.g., 105).

<br>

## **Step 3: Convert VMDK/Ova to Raw or QCOW2 (if not done already)**

```
qemu-img convert -f vmdk -O raw <vm-name>.vmdk <vm-name>.raw
```

ZFS prefers raw disk images for performance, but qcow2 also works.

<br>

## **Step 4: Import the Disk to ZFS Properly**

Use qm importdisk:

```
qm importdisk <vmid> <vm-name>.raw <zfs-storage-location>
```

Example:

```
qm importdisk <vmid> <vm-name>.raw rpool
```

This will create a ZFS volume like:

```
<zfs-storage-location>/vm-<vmid>-disk-0
```
<br>

## **Step 5: Attach the Imported Disk to VM**

In the Proxmox Web UI:

1. Go to VM 105 → **Hardware**
2. Click **Add → Hard Disk**
3. Choose:
    - **Bus/Device**: IDE (older OS like Kioptrix may expect this)
    - **Storage**: <zfs-storage-location>
    - **Disk**: vm-<vmid>-disk-0
4. Click **Add**

<br>

## **Step 6: Add disk to boot order**

1. Select options
2. Add disk to boot order
3. Make sure first item is vm-<vmid>-disk-0

<br>

## **Step 7: Boot the VM**

Start the VM → Open **Console** → You should see VM boot.
