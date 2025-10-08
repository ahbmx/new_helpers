To create a Python script using **pyVmomi** that retrieves all **VMDKs** and **RDMs** attached to one or more **VMs**, and maps them to:

* **Windows**: Drive letters (e.g., `C:\`, `D:\`)
* **Linux**: Device paths (e.g., `/dev/sda`, `/dev/sdb`)

you'll need to:

---

### ✅ **Key Requirements**

1. **vCenter access** with credentials.
2. Python environment with `pyVmomi`.
3. VMware Tools must be running in the guest OS to get the guest disk info (for drive letter/device).
4. Optional: Handle both **single VM** and **multiple VMs** as input.

---

### 📦 **Install pyVmomi**

```bash
pip install pyvmomi
```

---

### 🧠 How it Works

1. Connect to vCenter.
2. Retrieve the VM(s) by name.
3. For each VM:

   * Collect all virtual disks (VMDK and RDM).
   * Collect guest OS disk mappings (via VMware Tools).
   * Map virtual disk to device/drive letter.

---

### 🐍 **Python Script: Get VMDKs, RDMs, Drive Letters / Devices**

```python
from pyVim.connect import SmartConnect, Disconnect
from pyVmomi import vim
import ssl
import atexit

# ---- Configuration ----
VCENTER = "your-vcenter-host"
USERNAME = "administrator@vsphere.local"
PASSWORD = "your-password"
VM_NAMES = ["vm-name-1", "vm-name-2"]  # Can be one or multiple VM names
# ------------------------

def get_all_vms(content):
    obj_view = content.viewManager.CreateContainerView(content.rootFolder, [vim.VirtualMachine], True)
    vms = obj_view.view
    obj_view.Destroy()
    return vms

def find_vm_by_name(content, name):
    vms = get_all_vms(content)
    for vm in vms:
        if vm.name.lower() == name.lower():
            return vm
    return None

def get_vm_disks(vm):
    disks = []
    for device in vm.config.hardware.device:
        if isinstance(device, vim.vm.device.VirtualDisk):
            backing = device.backing
            disk_info = {
                "label": device.deviceInfo.label,
                "capacity_gb": round(device.capacityInKB / (1024 * 1024), 2),
                "file": getattr(backing, 'fileName', None),
                "backing_type": type(backing).__name__,
                "unit_number": device.unitNumber,
                "controller_key": device.controllerKey,
                "device_key": device.key,
                "rdm": isinstance(backing, vim.vm.device.VirtualDisk.RawDiskMappingVer1BackingInfo),
                "device": None  # Placeholder for guest device name (e.g., /dev/sda, C:)
            }
            disks.append(disk_info)
    return disks

def map_guest_disks(vm, disks):
    # Only works if VMware Tools is running
    if not vm.guest.toolsRunningStatus == 'guestToolsRunning':
        return disks  # Can't map guest disks

    guest_disks = vm.guest.disk

    for guest_disk in guest_disks:
        # Match by capacity
        for disk in disks:
            # Allow some margin due to rounding
            if abs(disk["capacity_gb"] - guest_disk.capacity / (1024**3)) < 1:
                disk["device"] = guest_disk.diskPath
                break
    return disks

def main():
    # Skip SSL verification
    context = ssl._create_unverified_context()
    si = SmartConnect(host=VCENTER, user=USERNAME, pwd=PASSWORD, sslContext=context)
    atexit.register(Disconnect, si)

    content = si.RetrieveContent()

    for vm_name in VM_NAMES:
        vm = find_vm_by_name(content, vm_name)
        if not vm:
            print(f"VM '{vm_name}' not found.")
            continue

        print(f"\n== Disks for VM: {vm.name} ==")

        disks = get_vm_disks(vm)
        disks = map_guest_disks(vm, disks)

        for disk in disks:
            print(f" - {disk['label']}")
            print(f"   > Size: {disk['capacity_gb']} GB")
            print(f"   > File: {disk['file']}")
            print(f"   > Type: {'RDM' if disk['rdm'] else 'VMDK'}")
            print(f"   > Guest Device: {disk['device'] if disk['device'] else 'N/A'}")

if __name__ == "__main__":
    main()
```

---

### 🧪 Example Output

```
== Disks for VM: win-server-01 ==
 - Hard disk 1
   > Size: 50.0 GB
   > File: [datastore1] win-server-01/win-server-01.vmdk
   > Type: VMDK
   > Guest Device: C:\

 - Hard disk 2
   > Size: 100.0 GB
   > File: [datastore1] win-server-01/win-server-01_1.vmdk
   > Type: RDM
   > Guest Device: D:\
```

Or for Linux:

```
== Disks for VM: ubuntu-01 ==
 - Hard disk 1
   > Size: 40.0 GB
   > File: [datastore1] ubuntu-01/ubuntu-01.vmdk
   > Type: VMDK
   > Guest Device: /dev/sda
```

---

### ⚠️ Limitations

* Mapping is **best-effort**: relies on size matching between virtual and guest disks.
* If VMware Tools is **not installed** or **not running**, guest mappings won't be available.
* If multiple disks have identical sizes, mapping may be incorrect.

---

### 💡 Ideas for Improvement

* Use `deviceKey` to correlate guest and virtual disks more accurately (if VMware exposes it).
* Add CSV/JSON export.
* Handle vSphere tags or folders for grouping.

---

Would you like a version of this script that takes VM names from a file or exports to CSV?
