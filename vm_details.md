Perfect ✅ — here’s the **final version** of the script, now with an optional **CSV export** feature.
It keeps the clean console output *and* can write all VM/disk data to a CSV file for inventory or automation use.

---

### 🐍 **Script: `get_vm_storage_info_multi_csv.py`**

```python
#!/usr/bin/env python3
from pyVim.connect import SmartConnect, Disconnect
from pyVmomi import vim
import ssl
import atexit
import csv
import os
from datetime import datetime

# ==========================
# CONFIGURATION VARIABLES
# ==========================
VCENTER = "vcenter.lab.local"
USERNAME = "administrator@vsphere.local"
PASSWORD = "MyPassw0rd!"
VM_NAMES = ["Test-VM01", "Test-VM02"]  # List of VMs to query
EXPORT_CSV = True                      # Set to False to disable CSV output
CSV_FILE = f"vm_storage_inventory_{datetime.now().strftime('%Y%m%d_%H%M%S')}.csv"
# ==========================


def get_obj(content, vimtype, name):
    """Return the vSphere object by name."""
    obj = None
    container = content.viewManager.CreateContainerView(content.rootFolder, vimtype, True)
    for c in container.view:
        if c.name == name:
            obj = c
            break
    container.Destroy()
    return obj


def get_cluster_name(vm):
    """Return the name of the cluster a VM belongs to."""
    parent = vm.resourcePool
    if parent:
        while parent:
            if isinstance(parent, vim.ClusterComputeResource):
                return parent.name
            parent = parent.parent
    return "Unknown"


def collect_vm_disk_info(vm):
    """Collects storage-related info for a VM and returns a list of dicts."""
    cluster_name = get_cluster_name(vm)
    datastore_names = [ds.name for ds in vm.datastore]

    rows = []
    for device in vm.config.hardware.device:
        if isinstance(device, vim.vm.device.VirtualDisk):
            backing = device.backing
            datastore_name = "Unknown"
            if hasattr(backing, "fileName") and backing.fileName.startswith("["):
                datastore_name = backing.fileName.split("]")[0].replace("[", "").strip()

            disk_info = {
                "VM Name": vm.name,
                "Cluster": cluster_name,
                "Power State": str(vm.runtime.powerState),
                "Datastore": datastore_name,
                "Disk Label": device.deviceInfo.label,
                "Capacity (GB)": round(device.capacityInKB / 1024 / 1024, 2),
                "Disk Type": "Unknown",
                "File": getattr(backing, "fileName", ""),
                "UUID": getattr(backing, "uuid", ""),
                "Backing Object ID": getattr(backing, "backingObjectId", ""),
                "NAA ID": getattr(backing, "lunUuid", ""),
                "RDM Device": "",
                "Compatibility Mode": "",
                "Guest Disks": "",
            }

            if isinstance(backing, vim.vm.device.VirtualDisk.FlatVer2BackingInfo):
                disk_info["Disk Type"] = "VMDK"
            elif isinstance(backing, vim.vm.device.VirtualDisk.RawDiskMappingVer1BackingInfo):
                disk_info["Disk Type"] = "RDM"
                disk_info["RDM Device"] = getattr(backing, "deviceName", "")
                disk_info["Compatibility Mode"] = getattr(backing, "compatibilityMode", "")

            rows.append(disk_info)

    # Guest OS Disks (requires VMware Tools)
    if vm.guest and vm.guest.disk:
        guest_disks = [
            f"{d.diskPath} ({d.capacity/1024**3:.1f}GB / {d.freeSpace/1024**3:.1f}GB free)"
            for d in vm.guest.disk
        ]
        for r in rows:
            r["Guest Disks"] = "; ".join(guest_disks)

    return rows


def main():
    # Ignore SSL warnings
    context = ssl._create_unverified_context()
    si = SmartConnect(host=VCENTER, user=USERNAME, pwd=PASSWORD, sslContext=context)
    atexit.register(Disconnect, si)
    content = si.RetrieveContent()

    all_rows = []

    for vm_name in VM_NAMES:
        vm = get_obj(content, [vim.VirtualMachine], vm_name)
        if not vm:
            print(f"\n⚠️  VM '{vm_name}' not found.\n")
            continue

        rows = collect_vm_disk_info(vm)
        all_rows.extend(rows)

        # Print summary to console
        print(f"\n==========================================")
        print(f"VM Name: {vm.name}")
        print(f"Cluster: {rows[0]['Cluster']}")
        print(f"Power State: {rows[0]['Power State']}")
        print("==========================================")

        for row in rows:
            print(f"Disk Label: {row['Disk Label']}")
            print(f"  Capacity (GB): {row['Capacity (GB)']}")
            print(f"  Datastore: {row['Datastore']}")
            print(f"  Disk Type: {row['Disk Type']}")
            print(f"  File: {row['File']}")
            if row['Disk Type'] == "RDM":
                print(f"  Device Name: {row['RDM Device']}")
                print(f"  Compatibility Mode: {row['Compatibility Mode']}")
            if row['NAA ID']:
                print(f"  NAA ID: {row['NAA ID']}")
            print("------------------------------------------")

        if rows and rows[0]["Guest Disks"]:
            print("Guest OS Disks (via VMware Tools):")
            print(f"  {rows[0]['Guest Disks']}")
        else:
            print("Guest OS Disks: N/A (VMware Tools not running or no data)")

        print("==========================================")

    # Export CSV
    if EXPORT_CSV and all_rows:
        keys = list(all_rows[0].keys())
        with open(CSV_FILE, "w", newline="", encoding="utf-8") as f:
            writer = csv.DictWriter(f, fieldnames=keys)
            writer.writeheader()
            writer.writerows(all_rows)
        print(f"\n📄 CSV export complete: {os.path.abspath(CSV_FILE)}")

    print("\n✅ Completed VM storage inventory.\n")


if __name__ == "__main__":
    main()
```

---

### 🧠 Example CSV Output

| VM Name   | Cluster        | Datastore  | Disk Label  | Capacity (GB) | Disk Type | File                                         | NAA ID                           | Guest Disks                                              |
| --------- | -------------- | ---------- | ----------- | ------------- | --------- | -------------------------------------------- | -------------------------------- | -------------------------------------------------------- |
| Test-VM01 | Prod-Cluster-A | datastore1 | Hard disk 1 | 50.0          | VMDK      | [datastore1] Test-VM01/Test-VM01.vmdk        |                                  | C:\ (50.0GB / 20.0GB free)                               |
| Test-VM01 | Prod-Cluster-A | datastore2 | Hard disk 2 | 200.0         | RDM       | [datastore2] Test-VM01/Test-VM01_1-rdmp.vmdk | 6006016023402600ABCDEF1234567890 | C:\ (50.0GB / 20.0GB free); D:\ (200.0GB / 150.0GB free) |

---

### 💡 Highlights

* Supports **multiple VMs** in one run.
* Optionally writes a detailed **CSV inventory file**.
* Safely ignores SSL warnings.
* Reports **VMDKs, RDMs, NAA IDs, cluster, datastore, and guest disks**.

---

Would you like me to add a small option to **query *all VMs in the vCenter*** automatically instead of specifying them manually? That’s a common next step.
