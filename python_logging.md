Here's a clean and modular implementation of your logging system based on your requirements:

---

### ✅ Folder Structure:

```
main.py
dvl/
├── __init__.py
├── configHelper.py       # Stores logging configurations
├── loggerHelper.py       # Implements logger creation logic
├── functionHelper.py
├── ecsHelper.py
```

---

## 1. `dvl/configHelper.py`: Configuration Constants

```python
# dvl/configHelper.py

import os

# Logging config values
LOG_LEVEL = "DEBUG"
LOG_DIR = "logs"
LOG_RETENTION_DAYS = 7
```

---

## 2. `dvl/loggerHelper.py`: Logging Setup and Custom Levels

```python
# dvl/loggerHelper.py

import logging
import os
import sys
from logging.handlers import TimedRotatingFileHandler
from pathlib import Path
from dvl import configHelper as cfg

# Custom log levels
SUCCESS_LEVEL_NUM = 25
SYSTEM_LEVEL_NUM = 15
COLLECTOR_LEVEL_NUM = 5

# Add custom log levels to logging
logging.addLevelName(SUCCESS_LEVEL_NUM, "SUCCESS")
logging.addLevelName(SYSTEM_LEVEL_NUM, "SYSTEM")
logging.addLevelName(COLLECTOR_LEVEL_NUM, "COLLECTOR")

# Add custom log level methods to Logger class
def success(self, message, *args, **kwargs):
    if self.isEnabledFor(SUCCESS_LEVEL_NUM):
        self._log(SUCCESS_LEVEL_NUM, message, args, **kwargs)

def system(self, message, *args, **kwargs):
    if self.isEnabledFor(SYSTEM_LEVEL_NUM):
        self._log(SYSTEM_LEVEL_NUM, message, args, **kwargs)

def collector(self, message, *args, **kwargs):
    if self.isEnabledFor(COLLECTOR_LEVEL_NUM):
        self._log(COLLECTOR_LEVEL_NUM, message, args, **kwargs)

logging.Logger.success = success
logging.Logger.system = system
logging.Logger.collector = collector

# Color mapping for levels
COLORS = {
    "DEBUG": "\033[94m",       # Blue
    "INFO": "\033[92m",        # Green
    "SUCCESS": "\033[1;32m",   # Bright Green
    "WARNING": "\033[93m",     # Yellow
    "ERROR": "\033[91m",       # Red
    "CRITICAL": "\033[1;31m",  # Bright Red
    "SYSTEM": "\033[95m",      # Magenta
    "COLLECTOR": "\033[96m",   # Cyan
    "RESET": "\033[0m"
}

class ColorFormatter(logging.Formatter):
    def format(self, record):
        level_name = record.levelname
        color = COLORS.get(level_name, "")
        reset = COLORS["RESET"]
        record.levelname = f"{color}{level_name}{reset}"
        record.msg = f"{color}{record.msg}{reset}"
        return super().format(record)

def get_logger(name=None, log_dir=None):
    log_dir = log_dir or cfg.LOG_DIR
    Path(log_dir).mkdir(parents=True, exist_ok=True)
    
    logger = logging.getLogger(name)
    if logger.handlers:
        return logger  # Avoid duplicate handlers

    logger.setLevel(getattr(logging, cfg.LOG_LEVEL))

    formatter = ColorFormatter(
        fmt="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )

    # Console handler
    ch = logging.StreamHandler(sys.stdout)
    ch.setFormatter(formatter)
    logger.addHandler(ch)

    # Rotating file handler (daily, keep for N days)
    fh = TimedRotatingFileHandler(
        filename=os.path.join(log_dir, f"{name or 'app'}.log"),
        when='midnight',
        backupCount=cfg.LOG_RETENTION_DAYS,
        encoding='utf-8'
    )
    file_formatter = logging.Formatter(
        fmt="%(asctime)s | %(levelname)s | %(name)s | %(message)s",
        datefmt="%Y-%m-%d %H:%M:%S"
    )
    fh.setFormatter(file_formatter)
    logger.addHandler(fh)

    return logger
```

---

## 3. Usage Example

### 🔹 In `main.py`:

```python
# main.py

from dvl.loggerHelper import get_logger

logger = get_logger(__name__)

logger.debug("This is a DEBUG message")
logger.info("This is an INFO message")
logger.success("This is a SUCCESS message")
logger.system("System level message")
logger.collector("Collector message")
logger.warning("This is a WARNING")
logger.error("This is an ERROR")
```

---

### 🔹 In any module, e.g., `functionHelper.py`:

```python
# dvl/functionHelper.py

from dvl.loggerHelper import get_logger

logger = get_logger(__name__)

def do_something():
    logger.info("Doing something...")
    logger.success("Completed successfully.")
```

---

### ✅ Features Recap:

* ✅ **Logger module shared across all files**
* ✅ **Custom log levels: `SUCCESS`, `SYSTEM`, `COLLECTOR`**
* ✅ **Colorized console output**
* ✅ **Daily rotating log files (N-day retention)**
* ✅ **Module name included in each log**
* ✅ **Log directory auto-creation**
* ✅ **Configurable via `configHelper.py`**

---

Would you like me to zip this up into a working example or generate a test run output?
