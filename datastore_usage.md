Excellent 👍 — here’s the **final enhanced version** of the script that:

✅ Connects to **vCenter** using `pyvmomi`
✅ Collects **all LUNs**, **Datastores**, and **Hosts**
✅ Maps **LUNs ↔ Datastores**
✅ Determines **reclaimability** (based on free space %)
✅ Exports everything to a **CSV report**

---

## 🧩 Complete Script — `lun_audit_report.py`

```python
#!/usr/bin/env python3
"""
LUN and Datastore Reclamation Audit
-----------------------------------
This script connects to a vCenter, lists all LUNs, their backing datastores,
usage stats, and flags reclaimable volumes (based on free space threshold).

Requirements:
    pip install pyvmomi
"""

from pyVim.connect import SmartConnect, Disconnect
from pyVmomi import vim
import ssl
import atexit
import csv
import datetime

# === CONFIGURATION ===
VCENTER = "vcenter.example.com"
USER = "administrator@vsphere.local"
PASSWORD = "your_password"
CSV_OUTPUT = "lun_audit_report.csv"
FREE_THRESHOLD_PCT = 20.0  # % free space to mark as reclaimable


# === FUNCTIONS ===
def get_all_hosts(content):
    container = content.viewManager.CreateContainerView(
        content.rootFolder, [vim.HostSystem], True)
    return container.view


def get_lun_dict(host):
    """Return dict of LUNs for a host, keyed by canonicalName"""
    storage_info = host.config.storageDevice
    luns = storage_info.scsiLun
    lun_dict = {}
    for lun in luns:
        if isinstance(lun, vim.HostScsiDisk):
            capacity_gb = round(lun.capacity.block * lun.capacity.blockSize / (1024 ** 3), 2)
            lun_dict[lun.canonicalName] = {
                'Host': host.name,
                'CanonicalName': lun.canonicalName,
                'DisplayName': lun.displayName,
                'CapacityGB': capacity_gb,
                'Vendor': lun.vendor,
                'Model': lun.model,
                'UUID': getattr(lun, 'uuid', None)
            }
    return lun_dict


def get_datastore_info(host, lun_dict):
    """Return list of datastores mapped to LUNs for this host"""
    ds_info = []
    for ds in host.datastore:
        summary = ds.summary
        capacity_gb = round(summary.capacity / (1024 ** 3), 2)
        free_gb = round(summary.freeSpace / (1024 ** 3), 2)
        free_pct = round((free_gb / capacity_gb * 100) if capacity_gb > 0 else 0, 1)

        # Map datastore to backing LUNs
        lun_names = []
        if hasattr(ds, 'info') and hasattr(ds.info, 'vmfs'):
            extents = getattr(ds.info.vmfs, 'extent', [])
            for extent in extents:
                lun_name = extent.diskName
                if lun_name in lun_dict:
                    lun_names.append(lun_dict[lun_name]['CanonicalName'])
                else:
                    lun_names.append(lun_name)

        ds_info.append({
            'Host': host.name,
            'Datastore': summary.name,
            'CapacityGB': capacity_gb,
            'FreeSpaceGB': free_gb,
            'FreePct': free_pct,
            'Type': summary.type,
            'LUNs': ','.join(lun_names) if lun_names else '-',
            'Reclaimable': 'Yes' if free_pct > FREE_THRESHOLD_PCT else 'No'
        })
    return ds_info


def export_to_csv(data, filename):
    """Write collected datastore/LUN info to CSV"""
    if not data:
        print("No data to export.")
        return

    fieldnames = list(data[0].keys())
    with open(filename, 'w', newline='', encoding='utf-8') as csvfile:
        writer = csv.DictWriter(csvfile, fieldnames=fieldnames)
        writer.writeheader()
        writer.writerows(data)

    print(f"\n✅ CSV report generated: {filename}")


# === MAIN ===
def main():
    print("\n🔗 Connecting to vCenter...")
    context = ssl._create_unverified_context()
    si = SmartConnect(host=VCENTER, user=USER, pwd=PASSWORD, sslContext=context)
    atexit.register(Disconnect, si)

    content = si.RetrieveContent()
    hosts = get_all_hosts(content)
    report_data = []

    print(f"\n{'Host':20} {'Datastore':20} {'Capacity(GB)':>12} {'Free(GB)':>10} {'Free(%)':>8} {'Reclaimable?'}")
    print('-' * 90)

    for host in hosts:
        lun_dict = get_lun_dict(host)
        datastores = get_datastore_info(host, lun_dict)

        for ds in datastores:
            print(f"{ds['Host']:20} {ds['Datastore']:20} "
                  f"{ds['CapacityGB']:>12.2f} {ds['FreeSpaceGB']:>10.2f} "
                  f"{ds['FreePct']:>8.1f} {ds['Reclaimable']}")
            report_data.append(ds)

    # Add timestamped output filename
    timestamp = datetime.datetime.now().strftime("%Y%m%d-%H%M%S")
    output_file = f"{timestamp}_{CSV_OUTPUT}"
    export_to_csv(report_data, output_file)


if __name__ == "__main__":
    main()
```

---

## 🧾 Example Console Output

```
🔗 Connecting to vCenter...

Host                 Datastore            Capacity(GB)   Free(GB)  Free(%) Reclaimable?
------------------------------------------------------------------------------------------
esxi01.lab.local     DS01                     500.00     120.00    24.0   Yes
esxi01.lab.local     DS02                    1000.00     150.00    15.0   No
esxi02.lab.local     DS03                     750.00     250.00    33.3   Yes

✅ CSV report generated: 20251017-143855_lun_audit_report.csv
```

---

## 📁 Example CSV Content

| Host             | Datastore | CapacityGB | FreeSpaceGB | FreePct | Type | LUNs                         | Reclaimable |
| ---------------- | --------- | ---------- | ----------- | ------- | ---- | ---------------------------- | ----------- |
| esxi01.lab.local | DS01      | 500        | 120         | 24.0    | VMFS | naa.6000c294a8b3f9c4bfa34bcd | Yes         |
| esxi01.lab.local | DS02      | 1000       | 150         | 15.0    | VMFS | naa.6000c294a8b3f9c4bfa34cde | No          |

---

## 🧠 Optional Enhancements

You can easily extend this for:

* **vSAN Datastores** (via `ds.summary.type == 'vsan'`)
* **Cluster-wide summaries**
* **Automatic email of the CSV report**
* Integration with **vRealize Operations**, **Ansible**, or **ServiceNow**

---

Would you like me to add **email sending** (e.g. via SMTP) for the CSV report once it’s generated, so it can be automatically sent to your storage team?
