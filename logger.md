To achieve what you want — a reusable, customizable logger accessible across your Python project — we’ll break this into organized parts according to your project structure. Here's a step-by-step solution:

---

## ✅ **Project Layout**

```
main.py
dvl/
│
├── __init__.py
├── configHelper.py
├── functionHelper.py
├── ecsHelper.py
└── loggerHelper.py   <-- NEW MODULE
```

---

## 1️⃣ `configHelper.py` – Centralized Config

Here you can define default configurations.

```python
# dvl/configHelper.py

DEFAULT_LOG_DIR = "logs"
DEFAULT_LOG_RETENTION_DAYS = 7
DEFAULT_LOG_MODE = "file"  # "file" or "console"
ENABLE_LOG_COLORS = True
```

---

## 2️⃣ `loggerHelper.py` – The Logger Singleton

This module initializes and provides a reusable logger.

```python
# dvl/loggerHelper.py

import logging
import os
import sys
from logging.handlers import TimedRotatingFileHandler
from dvl import configHelper

# Custom log levels
COLLECTOR_LEVEL = 15
SUCCESS_LEVEL = 25
DEVICE_LEVEL = 35

logging.addLevelName(COLLECTOR_LEVEL, "COLLECTOR")
logging.addLevelName(SUCCESS_LEVEL, "SUCCESS")
logging.addLevelName(DEVICE_LEVEL, "DEVICE")

# Color mapping for each level (ANSI codes)
LOG_COLORS = {
    "CRITICAL": "\033[1;31m",   # Bright Red
    "ERROR": "\033[31m",        # Red
    "WARNING": "\033[33m",      # Yellow
    "INFO": "\033[32m",         # Green
    "DEBUG": "\033[34m",        # Blue
    "COLLECTOR": "\033[35m",    # Magenta
    "SUCCESS": "\033[36m",      # Cyan
    "DEVICE": "\033[37m",       # White
    "RESET": "\033[0m"
}

_logger_instance = None  # Singleton logger

def get_logger(name=None, log_dir=None, mode=None, retention_days=None, enable_colors=None):
    global _logger_instance
    if _logger_instance:
        return _logger_instance

    log_dir = log_dir or configHelper.DEFAULT_LOG_DIR
    mode = mode or configHelper.DEFAULT_LOG_MODE
    retention_days = retention_days or configHelper.DEFAULT_LOG_RETENTION_DAYS
    enable_colors = enable_colors if enable_colors is not None else configHelper.ENABLE_LOG_COLORS

    os.makedirs(log_dir, exist_ok=True)

    logger = logging.getLogger(name if name else "dvl_logger")
    logger.setLevel(logging.DEBUG)

    if logger.hasHandlers():
        logger.handlers.clear()

    if mode == "console":
        handler = logging.StreamHandler(sys.stdout)
    else:
        handler = TimedRotatingFileHandler(
            filename=os.path.join(log_dir, "app.log"),
            when="D",
            interval=1,
            backupCount=retention_days
        )

    formatter = ColoredFormatter(enable_colors)
    handler.setFormatter(formatter)
    logger.addHandler(handler)

    # Add custom logging methods
    def collector(self, msg, *args, **kwargs):
        self.log(COLLECTOR_LEVEL, msg, *args, **kwargs)

    def success(self, msg, *args, **kwargs):
        self.log(SUCCESS_LEVEL, msg, *args, **kwargs)

    def device(self, msg, *args, **kwargs):
        self.log(DEVICE_LEVEL, msg, *args, **kwargs)

    logging.Logger.collector = collector
    logging.Logger.success = success
    logging.Logger.device = device

    _logger_instance = logger
    return logger


class ColoredFormatter(logging.Formatter):
    def __init__(self, enable_colors):
        self.enable_colors = enable_colors
        super().__init__(fmt="%(asctime)s [%(levelname)s] (%(module)s): %(message)s", datefmt="%Y-%m-%d %H:%M:%S")

    def format(self, record):
        levelname = record.levelname
        if self.enable_colors and levelname in LOG_COLORS:
            color = LOG_COLORS[levelname]
            reset = LOG_COLORS["RESET"]
            record.levelname = f"{color}{levelname}{reset}"
            record.msg = f"{color}{record.msg}{reset}"
        return super().format(record)
```

---

## 3️⃣ `functionHelper.py` – Path Utilities

```python
# dvl/functionHelper.py

import os

def get_cwd():
    """Get the absolute path where main.py is run."""
    return os.getcwd()

def ensure_directories(paths):
    """Create directories from a single path or a list of paths."""
    if isinstance(paths, str):
        paths = [paths]

    for path in paths:
        os.makedirs(path, exist_ok=True)
```

---

## 4️⃣ Example Usage in `main.py`

```python
# main.py

from dvl import functionHelper
from dvl import loggerHelper

# Get current working directory and ensure log dir exists
cwd = functionHelper.get_cwd()
log_dir = f"{cwd}/logs"

# Initialize logger once
logger = loggerHelper.get_logger(log_dir=log_dir, mode="file")

logger.info("This is a standard info log.")
logger.success("This is a success log.")
logger.device("This is a device log.")
logger.collector("This is a collector log.")
logger.error("This is an error log.")
```

---

## 🧪 Usage in Other Modules

Inside **any other module**, just do:

```python
# dvl/ecsHelper.py

from dvl.loggerHelper import get_logger

logger = get_logger()

def run_task():
    logger.debug("Running ECS task")
```

Because the logger is implemented as a singleton, `get_logger()` will return the already-initialized logger without re-initializing it.

---

## ✅ Summary of Features Implemented

* ✅ Central logging config (`configHelper`)
* ✅ Singleton logger accessible globally
* ✅ Console vs file logging with retention days
* ✅ Automatic directory creation
* ✅ Custom log levels: `collector`, `success`, `device`
* ✅ Color support per log level (configurable)
* ✅ Log format includes module name
* ✅ Path utilities in `functionHelper`

---

Let me know if you want a version of this with unit tests or a class-based logger for extensibility.
