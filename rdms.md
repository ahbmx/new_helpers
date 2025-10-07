

---

### 🧩 Requirements

1. Install pyVmomi:

   ```bash
   pip install pyvmomi
   ```

2. You’ll need vCenter credentials and hostname.

---

### ✅ Python Script: `check_vm_rdms.py`

```python
from pyVim.connect import SmartConnect, Disconnect
from pyVmomi import vim
import ssl

# --- Configuration ---
VCENTER = "vcenter.example.com"
USERNAME = "administrator@vsphere.local"
PASSWORD = "your_password"
VM_LIST = ["VM01", "VM02", "VM03"]  # List of VMs to check

# --- Disable SSL warnings (optional) ---
context = ssl._create_unverified_context()

def get_obj(content, vimtype, name):
    """
    Return an object by name from vCenter inventory.
    """
    container = content.viewManager.CreateContainerView(content.rootFolder, vimtype, True)
    for obj in container.view:
        if obj.name == name:
            return obj
    return None

def list_rdms(vm):
    """
    List RDM disks attached to a VM.
    """
    rdms = []
    for device in vm.config.hardware.device:
        if isinstance(device, vim.vm.device.VirtualDisk):
            backing = device.backing
            # Identify RDMs
            if isinstance(backing, vim.vm.device.VirtualDisk.RawDiskMappingVer1BackingInfo):
                rdms.append({
                    "Device": device.deviceInfo.label,
                    "LUN ID": backing.lunUuid,
                    "Backing File": backing.fileName,
                    "Compatibility Mode": backing.compatibilityMode,
                    "Capacity (GB)": round(device.capacityInKB / (1024 * 1024), 2)
                })
    return rdms

def main():
    # Connect to vCenter
    si = SmartConnect(host=VCENTER, user=USERNAME, pwd=PASSWORD, sslContext=context)
    content = si.RetrieveContent()
    
    print(f"{'VM Name':<20} | {'Device':<20} | {'LUN ID':<40} | {'Size (GB)':<10} | {'Mode':<10}")
    print("-" * 110)
    
    for vm_name in VM_LIST:
        vm = get_obj(content, [vim.VirtualMachine], vm_name)
        if not vm:
            print(f"{vm_name:<20} | VM not found")
            continue

        rdms = list_rdms(vm)
        if not rdms:
            print(f"{vm_name:<20} | No RDMs attached")
        else:
            for rdm in rdms:
                print(f"{vm_name:<20} | {rdm['Device']:<20} | {rdm['LUN ID']:<40} | "
                      f"{rdm['Capacity (GB)']:<10} | {rdm['Compatibility Mode']:<10}")

    Disconnect(si)

if __name__ == "__main__":
    main()
```

---

### 🧾 Example Output

```
VM Name              | Device               | LUN ID                                  | Size (GB)  | Mode      
--------------------------------------------------------------------------------------------------------------
VM01                 | Hard disk 2          | 600508b1001c7f59a3a13f7df8e3b4b5        | 500.0      | physical  
VM02                 | Hard disk 1          | 600508b1001c7f59a3a13f7df8e3b4c6        | 1000.0     | virtual   
VM03                 | No RDMs attached
```

---

### 🧠 Notes

* The script identifies RDMs by checking for `RawDiskMappingVer1BackingInfo` in each VM’s virtual disks.
* You can easily export results to a CSV if you’d like (I can add that if needed).
* It safely handles missing VMs and shows useful detail for each RDM.

---

Would you like me to add **CSV export** or **automatic discovery of all VMs** (so you don’t need to list them manually)?
