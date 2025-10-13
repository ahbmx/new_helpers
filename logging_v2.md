Perfect — let’s set up a **shared logging configuration** that works across all your modules (`storage_collector.py`, `functions.py`, `brocade.py`, etc.), logs to console by default, and optionally also logs to files (one for normal logs, one for errors).

Here’s the **cleanest, production-style setup**:

---

### 🧱 Directory Structure (for reference)

```
/opt/storage/
│
├── storage_collector.py
└── dvl/
    ├── __init__.py
    ├── functions.py
    ├── brocade.py
    ├── powermax.py
    └── reports.py
```

---

### ✅ Step 1: Create a logging helper (`dvl/logger.py`)

Add a small utility module to centralize logging setup.

```python
# /opt/storage/dvl/logger.py
import logging
import sys
from pathlib import Path

def setup_logging(log_to_file=False, log_dir="/opt/storage/logs"):
    """Configure logging for all modules in the project."""
    # Ensure directory exists if file logging is enabled
    if log_to_file:
        Path(log_dir).mkdir(parents=True, exist_ok=True)

    # Define log format
    formatter = logging.Formatter(
        fmt="%(asctime)s [%(levelname)s] [%(name)s] %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )

    # Root logger (shared across all modules)
    root_logger = logging.getLogger()
    root_logger.setLevel(logging.DEBUG)

    # --- Console handler ---
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(formatter)
    console_handler.setLevel(logging.INFO)
    root_logger.addHandler(console_handler)

    # --- File handlers (optional) ---
    if log_to_file:
        # Info/Debug logs
        info_file = Path(log_dir) / "storage_collector.log"
        info_handler = logging.FileHandler(info_file)
        info_handler.setFormatter(formatter)
        info_handler.setLevel(logging.DEBUG)
        root_logger.addHandler(info_handler)

        # Error logs only
        error_file = Path(log_dir) / "storage_collector_error.log"
        error_handler = logging.FileHandler(error_file)
        error_handler.setFormatter(formatter)
        error_handler.setLevel(logging.ERROR)
        root_logger.addHandler(error_handler)
```

---

### ✅ Step 2: Initialize the logger in your main script

In `storage_collector.py`, call the setup once — this sets up global logging for all modules.

```python
# /opt/storage/storage_collector.py
from dvl.logger import setup_logging
import logging
from dvl import functions, brocade, powermax, reports

def main():
    setup_logging(log_to_file=True)  # or False for console-only

    logger = logging.getLogger(__name__)
    logger.info("Storage collector service started.")

    # Example usage
    functions.do_something()
    brocade.collect_data()

if __name__ == "__main__":
    main()
```

---

### ✅ Step 3: Use the logger in other modules

Every module just needs to get its own logger by name:

```python
# /opt/storage/dvl/functions.py
import logging

logger = logging.getLogger(__name__)

def do_something():
    logger.debug("This is a debug message from functions module.")
    logger.info("Running storage function.")
    try:
        # Example of an error
        1 / 0
    except ZeroDivisionError:
        logger.exception("Something went wrong in functions.")
```

And similarly in `brocade.py`, `powermax.py`, etc.:

```python
# /opt/storage/dvl/brocade.py
import logging
logger = logging.getLogger(__name__)

def collect_data():
    logger.info("Collecting data from Brocade switches.")
```

---

### ✅ Result

When you run `storage_collector.py`:

#### Console output:

```
2025-10-13 10:55:12 [INFO] [__main__] Storage collector service started.
2025-10-13 10:55:12 [INFO] [dvl.brocade] Collecting data from Brocade switches.
```

#### Files (if `log_to_file=True`):

* `/opt/storage/logs/storage_collector.log` — all logs
* `/opt/storage/logs/storage_collector_error.log` — only errors/exceptions

---

Would you like me to extend this so that **log level** (e.g. DEBUG/INFO/WARNING) and **log directory** can be set via `argparse` options when you run the script (for your “test mode”)?
