Excellent request 👌 — adding both the **Cluster name** and **LUN NAA ID** makes the report far more useful for enterprise storage audits.

Let’s refine the script so it now includes:

* 🧩 **Cluster name** each host belongs to
* 🧱 **LUN NAA ID** (from `canonicalName` or `uuid`, whichever is available)
* 🔄 Full mapping: **Cluster → Host → Datastore → LUN → Reclaimable**

---

## ✅ Updated Script — `lun_audit_report_v2.py`

```python
#!/usr/bin/env python3
"""
LUN and Datastore Reclamation Audit (v2)
----------------------------------------
Lists all LUNs, their backing datastores, cluster, usage stats,
and flags reclaimable volumes (based on free space threshold).

Requires:
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
    """Return all ESXi hosts in vCenter"""
    container = content.viewManager.CreateContainerView(
        content.rootFolder, [vim.HostSystem], True)
    return container.view


def get_cluster_name(host):
    """Return the cluster name that the host belongs to (if any)"""
    try:
        if isinstance(host.parent, vim.ClusterComputeResource):
            return host.parent.name
        elif hasattr(host, "parent") and hasattr(host.parent, "parent"):
            # Handle nested folders
            parent = host.parent
            while parent:
                if isinstance(parent, vim.ClusterComputeResource):
                    return parent.name
                parent = getattr(parent, "parent", None)
        return "Standalone Host"
    except Exception:
        return "Unknown"


def get_lun_dict(host):
    """Return dict of LUNs for a host, keyed by canonicalName"""
    storage_info = host.config.storageDevice
    luns = storage_info.scsiLun
    lun_dict = {}
    for lun in luns:
        if isinstance(lun, vim.HostScsiDisk):
            capacity_gb = round(lun.capacity.block * lun.capacity.blockSize / (1024 ** 3), 2)
            naa_id = lun.canonicalName  # Typically starts with 'naa.'
            lun_dict[lun.canonicalName] = {
                'Host': host.name,
                'Cluster': get_cluster_name(host),
                'CanonicalName': lun.canonicalName,
                'DisplayName': lun.displayName,
                'CapacityGB': capacity_gb,
                'Vendor': lun.vendor,
                'Model': lun.model,
                'UUID': getattr(lun, 'uuid', None),
                'NAA_ID': naa_id
            }
    return lun_dict


def get_datastore_info(host, lun_dict):
    """Return list of datastores mapped to LUNs for this host"""
    ds_info = []
    cluster_name = get_cluster_name(host)

    for ds in host.datastore:
        summary = ds.summary
        capacity_gb = round(summary.capacity / (1024 ** 3), 2)
        free_gb = round(summary.freeSpace / (1024 ** 3), 2)
        free_pct = round((free_gb / capacity_gb * 100) if capacity_gb > 0 else 0, 1)

        # Map datastore to backing LUNs
        lun_entries = []
        if hasattr(ds, 'info') and hasattr(ds.info, 'vmfs'):
            extents = getattr(ds.info.vmfs, 'extent', [])
            for extent in extents:
                lun_name = extent.diskName
                if lun_name in lun_dict:
                    lun_data = lun_dict[lun_name]
                    lun_entries.append(f"{lun_data['NAA_ID']}")
                else:
                    lun_entries.append(lun_name)

        ds_info.append({
            'Cluster': cluster_name,
            'Host': host.name,
            'Datastore': summary.name,
            'CapacityGB': capacity_gb,
            'FreeSpaceGB': free_gb,
            'FreePct': free_pct,
            'Type': summary.type,
            'LUNs': ','.join(lun_entries) if lun_entries else '-',
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

    print(f"\n{'Cluster':20} {'Host':20} {'Datastore':20} {'Free(%)':>8} {'Reclaimable?':>12} {'LUNs (NAA.ID)':>30}")
    print('-' * 120)

    for host in hosts:
        lun_dict = get_lun_dict(host)
        datastores = get_datastore_info(host, lun_dict)

        for ds in datastores:
            print(f"{ds['Cluster']:20} {ds['Host']:20} {ds['Datastore']:20} "
                  f"{ds['FreePct']:>8.1f} {ds['Reclaimable']:>12} {ds['LUNs'][:30]:>30}")
            report_data.append(ds)

    # Add timestamped output filename
    timestamp = datetime.datetime.now().strftime("%Y%m%d-%H%M%S")
    output_file = f"{timestamp}_{CSV_OUTPUT}"
    export_to_csv(report_data, output_file)


if __name__ == "__main__":
    main()
```

---

## 🧾 Example Output

**Console:**

```
Cluster              Host                 Datastore            Free(%)  Reclaimable?                  LUNs (NAA.ID)
------------------------------------------------------------------------------------------------------------------------
Prod-Cluster         esxi01.lab.local     DS01                   24.0          Yes    naa.6000c294a8b3f9c4bfa34bcd
Prod-Cluster         esxi01.lab.local     DS02                   15.0           No    naa.6000c294a8b3f9c4bfa34cde
DR-Cluster           esxi02.lab.local     DS03                   33.3          Yes    naa.6000c294a8b3f9c4bfa34aaa

✅ CSV report generated: 20251017-155645_lun_audit_report.csv
```

**CSV Columns:**

| Cluster      | Host             | Datastore | CapacityGB | FreeSpaceGB | FreePct | Type | LUNs                         | Reclaimable |
| ------------ | ---------------- | --------- | ---------- | ----------- | ------- | ---- | ---------------------------- | ----------- |
| Prod-Cluster | esxi01.lab.local | DS01      | 500        | 120         | 24.0    | VMFS | naa.6000c294a8b3f9c4bfa34bcd | Yes         |

---

## 🧠 Notes

* `NAA_ID` usually corresponds to `naa.xxxxx` strings (your SAN LUN identifier).
* If a datastore spans multiple LUNs (extents), all NAA IDs will be listed comma-separated.
* The **cluster name** is derived by recursively checking the host’s parent object in vCenter.
* Works for both **standalone hosts** and **clustered environments**.

---

Would you like me to also include the **datastore UUID** (useful for correlating with VMFS metadata or array-level tools like Pure, NetApp, or EMC)?
