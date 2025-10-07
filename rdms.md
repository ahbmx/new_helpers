

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

Excellent — now we’re moving from the **VM-level RDMs** to **ESXi host-level storage mapping**.
You want a **report of all LUNs (datastores or raw devices)** attached to each **ESXi host**, including **their capacity and usage**.

✅ That’s totally doable with `pyVmomi` as well.

Here’s a detailed Python script that:

* Connects to **vCenter**
* Lists all **ESXi hosts**
* For each host, lists **all attached LUNs (scsiLun)**
* Displays **LUN name, UUID, capacity (GB), and display name**
* Optionally, lists **used capacity** (if the LUN is part of a datastore)

---

### 🧩 Requirements

```bash
pip install pyvmomi
```

---

### ✅ Script: `esxi_lun_report.py`

```python
from pyVim.connect import SmartConnect, Disconnect
from pyVmomi import vim
import ssl

# --- vCenter credentials ---
VCENTER = "vcenter.example.com"
USERNAME = "administrator@vsphere.local"
PASSWORD = "your_password"

# --- Disable SSL certificate verification (optional) ---
context = ssl._create_unverified_context()

def bytes_to_gb(bytes_val):
    return round(bytes_val / (1024 ** 3), 2)

def main():
    # Connect to vCenter
    si = SmartConnect(host=VCENTER, user=USERNAME, pwd=PASSWORD, sslContext=context)
    content = si.RetrieveContent()
    
    print(f"{'Host':<25} | {'LUN Name':<25} | {'UUID':<35} | {'Capacity (GB)':<15} | {'Used (GB)':<15}")
    print("-" * 125)
    
    for datacenter in content.rootFolder.childEntity:
        hostFolder = datacenter.hostFolder
        computeResourceList = hostFolder.childEntity

        for computeResource in computeResourceList:
            for host in computeResource.host:
                storage_system = host.configManager.storageSystem
                lun_list = storage_system.storageDeviceInfo.multipathInfo.lun

                for lun in storage_system.storageDeviceInfo.scsiLun:
                    if isinstance(lun, vim.HostScsiDisk):
                        lun_name = lun.displayName
                        lun_uuid = lun.uuid
                        capacity_gb = bytes_to_gb(lun.capacity.block * lun.capacity.blockSize)
                        used_gb = 0.0

                        # Try to correlate LUN with datastores
                        for ds in host.datastore:
                            if hasattr(ds.info, 'vmfs') and ds.info.vmfs.extent:
                                for extent in ds.info.vmfs.extent:
                                    if extent.diskName in lun.displayName:
                                        used_gb = bytes_to_gb(ds.summary.capacity - ds.summary.freeSpace)

                        print(f"{host.name:<25} | {lun_name:<25} | {lun_uuid:<35} | "
                              f"{capacity_gb:<15} | {used_gb:<15}")

    Disconnect(si)

if __name__ == "__main__":
    main()
```

---

### 🧾 Example Output

```
Host                      | LUN Name                 | UUID                                 | Capacity (GB)   | Used (GB)
-----------------------------------------------------------------------------------------------------------------------------
esx01.lab.local           | naa.600508b1001c7f59...  | 600508b1001c7f59a3a13f7df8e3b4b5     | 500.0           | 250.0
esx01.lab.local           | naa.600508b1001c7f59...  | 600508b1001c7f59a3a13f7df8e3b4b6     | 1000.0          | 0.0
esx02.lab.local           | naa.600508b1001c7f59...  | 600508b1001c7f59a3a13f7df8e3b4b5     | 500.0           | 250.0
```

---

### 🧠 Notes

* **Capacity (GB)** comes from the LUN device itself.
* **Used (GB)** is estimated based on associated VMFS datastore usage.
* RDMs typically won’t show as “used” because they’re not datastores — if you need to **correlate RDM LUNs**, we can merge this with the previous script.
* You can easily export the data to **CSV** for Excel reporting.

---

Perfect ✅ — that’s a classic VMware inventory task.
You want a Python script (using **pyVmomi**) that connects to **vCenter**, iterates through **clusters → hosts**, and extracts **HBA WWNs** from each ESXi host, printing them in two formats:

* **With colons** → e.g. `50:06:0e:80:12:34:56:78`
* **Without colons** → e.g. `50060e8012345678`

---

### 🧩 Prerequisites

Install the vSphere SDK:

```bash
pip install pyvmomi
```

---

### ✅ Script: `esx_host_wwns.py`

```python
from pyVim.connect import SmartConnect, Disconnect
from pyVmomi import vim
import ssl

# --- vCenter connection info ---
VCENTER = "vcenter.example.com"
USERNAME = "administrator@vsphere.local"
PASSWORD = "your_password"

# --- Disable SSL warnings (optional) ---
context = ssl._create_unverified_context()

def format_wwn(wwn_str):
    """
    Format a WWN string with and without colons.
    VMware returns WWN as a hexadecimal string (e.g. 50060e8012345678)
    """
    clean = wwn_str.replace(":", "").strip().lower()
    with_colons = ":".join(clean[i:i+2] for i in range(0, len(clean), 2))
    return with_colons, clean

def main():
    # Connect to vCenter
    si = SmartConnect(host=VCENTER, user=USERNAME, pwd=PASSWORD, sslContext=context)
    content = si.RetrieveContent()

    print(f"{'Cluster':<25} | {'Host':<25} | {'WWN (with colons)':<25} | {'WWN (no colons)':<20}")
    print("-" * 110)

    for datacenter in content.rootFolder.childEntity:
        hostFolder = datacenter.hostFolder
        computeResourceList = hostFolder.childEntity

        for computeResource in computeResourceList:
            cluster_name = computeResource.name  # Works for clusters and standalone hosts

            for host in computeResource.host:
                storage_system = host.configManager.storageSystem
                hbas = storage_system.storageDeviceInfo.hostBusAdapter

                for hba in hbas:
                    # Filter Fibre Channel adapters
                    if isinstance(hba, vim.host.FibreChannelHba):
                        wwn_hex = format(int(hba.portWorldWideName))
                        wwn_str = "%016x" % wwn_hex  # Convert int to padded hex string
                        wwn_colon, wwn_nocolon = format_wwn(wwn_str)
                        print(f"{cluster_name:<25} | {host.name:<25} | {wwn_colon:<25} | {wwn_nocolon:<20}")

    Disconnect(si)

if __name__ == "__main__":
    main()
```

---

### 🧾 Example Output

```
Cluster                  | Host                      | WWN (with colons)         | WWN (no colons)
--------------------------------------------------------------------------------------------------------------
Production-Cluster        | esx01.lab.local           | 50:06:0e:80:12:34:56:78   | 50060e8012345678
Production-Cluster        | esx01.lab.local           | 50:06:0e:80:87:65:43:21   | 50060e8087654321
DR-Cluster                | esx02.lab.local           | 10:00:00:90:fa:12:34:56   | 10000090fa123456
```

---

### 🧠 Notes

* The **WWN** comes from each `vim.host.FibreChannelHba.portWorldWideName` property.
* The script handles multiple FC HBAs per host — you’ll get one line per WWN.
* Works even if a host is standalone (cluster name will just be its compute resource name).

---

Would you like me to make this output go to a **CSV file** (e.g. `esx_host_wwns.csv`) for easy Excel use?
