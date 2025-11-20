Great — here is an updated version of the script that **exports the LUN report to an Excel file (`lun_report.xlsx`)** using `pandas` and `openpyxl`.

You only need to install:

```bash
pip install pandas openpyxl pyvmomi
```

---

# ✔ Python Script — LUN Report Exported to Excel

```python
#!/usr/bin/env python3

from pyVim.connect import SmartConnect, Disconnect
from pyVmomi import vim
import ssl
import pandas as pd

# ------------------------------------------------------------
# Helper Functions
# ------------------------------------------------------------

def get_obj(content, vimtypes):
    """
    Return all managed objects of a given type.
    """
    container = content.viewManager.CreateContainerView(
        content.rootFolder, vimtypes, True)
    return container.view


def get_lun_info(host):
    """
    Build dictionary of LUNs visible to an ESXi host.
    Key = naa.id
    """
    lun_dict = {}
    for lun in host.config.storageDevice.scsiLun:
        if isinstance(lun, vim.HostScsiDisk):
            lun_dict[lun.canonicalName] = {
                "displayName": lun.displayName,
                "capacityGB": round(
                    lun.capacity.block * lun.capacity.blockSize / (1024**3), 2
                )
            }
    return lun_dict


def get_datastore_naa(ds):
    """
    Extract naa.id from VMFS datastore.
    """
    try:
        ext = ds.info.vmfs.extent
        if ext:
            return ext[0].diskName
    except:
        pass
    return None


# ------------------------------------------------------------
# Main
# ------------------------------------------------------------

def main():
    # ---- Connect to vCenter ----
    vc = input("vCenter hostname: ")
    user = input("Username: ")
    pwd = input("Password: ")

    ctx = ssl._create_unverified_context()
    service_instance = SmartConnect(
        host=vc, user=user, pwd=pwd, sslContext=ctx)
    content = service_instance.RetrieveContent()
    print("\nConnected to vCenter:", vc)

    # ---- List clusters ----
    clusters = get_obj(content, [vim.ClusterComputeResource])

    print("\nAvailable Clusters:")
    for i, cl in enumerate(clusters, 1):
        print(f"{i}. {cl.name}")

    choice = int(input("\nChoose a cluster by number: "))
    cluster = clusters[choice - 1]
    print(f"\nSelected cluster: {cluster.name}\n")

    # ---- Extract hosts ----
    hosts = cluster.host

    # ---- Gather LUN info ----
    print("Gathering LUN and datastore info...")
    cluster_luns = {}
    for host in hosts:
        host_luns = get_lun_info(host)
        for naa, info in host_luns.items():
            if naa not in cluster_luns:
                cluster_luns[naa] = {
                    "LUN_Display_Name": info["displayName"],
                    "NAA_ID": naa,
                    "Size_GB": info["capacityGB"],
                    "Hosts": set(),
                    "Datastores": set(),
                    "VMs": set(),
                }
            cluster_luns[naa]["Hosts"].add(host.name)

    # ---- Map LUNs to datastores and VMs ----
    datastores = cluster.datastore

    for ds in datastores:
        naa = get_datastore_naa(ds)
        if naa and naa in cluster_luns:
            cluster_luns[naa]["Datastores"].add(ds.name)
            # VMs on datastore
            for vm in ds.vm:
                cluster_luns[naa]["VMs"].add(vm.name)

    # ---- Convert to Excel ----
    rows = []
    for naa, lun in cluster_luns.items():
        rows.append({
            "LUN Display Name": lun["LUN_Display_Name"],
            "NAA ID": naa,
            "Size (GB)": lun["Size_GB"],
            "Hosts": ", ".join(sorted(lun["Hosts"])),
            "Datastores": ", ".join(sorted(lun["Datastores"])),
            "VMs": ", ".join(sorted(lun["VMs"])),
        })

    df = pd.DataFrame(rows)
    excel_file = "lun_report.xlsx"
    df.to_excel(excel_file, index=False)

    print(f"\nExcel report generated: {excel_file}")

    Disconnect(service_instance)


if __name__ == "__main__":
    main()
```

---

# ✔ What You Get

The generated **Excel file** will contain columns:

| LUN Display Name | NAA ID | Size (GB) | Hosts | Datastores | VMs |
| ---------------- | ------ | --------- | ----- | ---------- | --- |

Each row corresponds to one LUN visible in the selected cluster.

---

# Want Enhancements?

I can add:

✅ Auto-formatting in Excel (colors, column width, freeze header)
✅ Separate Excel sheets (one for LUNs, one for Hosts, one for Datastores)
✅ Add datastore capacity / free space
✅ Add multipath info (active/standby paths)
✅ Add storage array vendor detection

Just tell me what you want!
