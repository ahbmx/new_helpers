

## ✅ Multithreaded Ping Script to Update `ping_status` in PostgreSQL

---

### 🧱 Assumptions

* Your table is named `device_inventory`.
* You're pinging based on `device_name` (hostname or IP).
* You already have a column `ping_status`.

---

### 🐍 Full Python Script

```python
import pandas as pd
from sqlalchemy import create_engine
from concurrent.futures import ThreadPoolExecutor, as_completed
from datetime import datetime
import subprocess
import platform

# ---------- Configuration ----------
DB_URL = "postgresql+psycopg2://user:password@localhost:5432/mydatabase"
PING_COUNT = "1"
MAX_THREADS = 20

# ---------- Ping Function ----------
def ping_device(host):
    param = "-n" if platform.system().lower() == "windows" else "-c"
    command = ["ping", param, PING_COUNT, host]

    try:
        result = subprocess.run(command, stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)
        return "reachable" if result.returncode == 0 else "unreachable"
    except Exception:
        return "error"

# ---------- Load Devices ----------
engine = create_engine(DB_URL)
df = pd.read_sql("SELECT device_id, device_name FROM device_inventory", engine)

# ---------- Multithreaded Ping ----------
results = []

with ThreadPoolExecutor(max_workers=MAX_THREADS) as executor:
    future_to_device = {
        executor.submit(ping_device, row['device_name']): row
        for _, row in df.iterrows()
    }

    for future in as_completed(future_to_device):
        row = future_to_device[future]
        try:
            status = future.result()
        except Exception:
            status = "error"

        results.append({
            "device_id": row["device_id"],
            "ping_status": status,
            "last_updated": datetime.now()
        })

# ---------- Update DB ----------
with engine.begin() as conn:
    for result in results:
        conn.execute("""
            UPDATE device_inventory
            SET ping_status = %s,
                last_updated = %s
            WHERE device_id = %s
        """, (result["ping_status"], result["last_updated"], result["device_id"]))

print("Ping statuses updated.")
```

---

### ⚡ Output Example

After running, your `device_inventory` table might look like:

| device\_id | device\_name | ping\_status | last\_updated       |
| ---------- | ------------ | ------------ | ------------------- |
| 1          | Router001    | reachable    | 2025-09-12 15:30:00 |
| 2          | Switch002    | unreachable  | 2025-09-12 15:30:01 |
| 3          | Firewall003  | error        | 2025-09-12 15:30:02 |

---

### 🧠 Optional Upgrades

* Add `ping_response_time_ms` if needed (use `time` module to measure).
* Log errors or failed hosts to a file.
* Schedule the script using `cron`, `systemd`, or `Task Scheduler`.

---

Would you like a version that logs failed pings, or one that uses `asyncio` + `asyncping` or similar libraries for ultra-performance?
