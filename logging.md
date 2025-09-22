Here’s a clean and scalable way to implement a reusable logging setup that meets all your requirements:

---

### 🔧 **Project Structure (after update)**

```
main.py
dvl/
├── __init__.py
├── configHelper.py       # Contains logging config values
├── functionHelper.py
├── ecsHelper.py
├── loggerHelper.py       # Contains the logging setup function
```

---

### ✅ **Requirements Recap**

* Logger setup **only once** and reused everywhere.
* Support for:

  * Console or rotating file handler (default: file logs for 7 days).
  * Customizable log directory.
  * Module name in formatter.
  * Adjustable logging level.

---

### 1️⃣ `dvl/configHelper.py` — Store configuration

```python
# dvl/configHelper.py

LOG_DIR = "logs"
LOG_LEVEL = "DEBUG"  # Can be "INFO", "WARNING", etc.
LOG_MODE = "file"    # or "console"
LOG_RETENTION_DAYS = 7
```

---

### 2️⃣ `dvl/loggerHelper.py` — Logging initializer

```python
# dvl/loggerHelper.py

import logging
import os
from logging.handlers import TimedRotatingFileHandler
from dvl import configHelper as config

_loggers = {}

def get_logger(name=None):
    global _loggers

    if name in _loggers:
        return _loggers[name]

    logger = logging.getLogger(name)
    if logger.handlers:
        # Already configured (guard in case of reload)
        _loggers[name] = logger
        return logger

    logger.setLevel(getattr(logging, config.LOG_LEVEL.upper(), logging.INFO))
    
    formatter = logging.Formatter(
        fmt='%(asctime)s [%(levelname)s] [%(name)s] %(message)s',
        datefmt='%Y-%m-%d %H:%M:%S'
    )

    if config.LOG_MODE == "console":
        handler = logging.StreamHandler()
    else:
        os.makedirs(config.LOG_DIR, exist_ok=True)
        log_path = os.path.join(config.LOG_DIR, "app.log")
        handler = TimedRotatingFileHandler(
            log_path, when='midnight', backupCount=config.LOG_RETENTION_DAYS
        )

    handler.setFormatter(formatter)
    logger.addHandler(handler)
    logger.propagate = False  # Prevent duplicate logs

    _loggers[name] = logger
    return logger
```

---

### 3️⃣ Usage in **any module** (e.g., `functionHelper.py`, `main.py`, etc.)

```python
# dvl/functionHelper.py

from dvl.loggerHelper import get_logger

logger = get_logger(__name__)

def some_function():
    logger.info("Running some_function...")
```

```python
# main.py

from dvl.loggerHelper import get_logger

logger = get_logger(__name__)

def main():
    logger.info("Application started")

if __name__ == "__main__":
    main()
```

---

### 🔁 Behavior

* `get_logger(__name__)` ensures each module gets a logger tagged with its name.
* Logging is configured **once**, reused globally.
* Log rotation and retention are handled automatically.
* Easy to switch between **console** and **file logging** via `configHelper.py`.

---

### 🧪 Optional: Add dynamic config override

If you want to override config values programmatically (e.g., from CLI args), update `configHelper` before importing `get_logger()`.

Let me know if you want to add CLI-based overrides or support for JSON/YAML config files.
