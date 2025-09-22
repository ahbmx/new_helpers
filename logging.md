Perfect — switching from `FileHandler` to `TimedRotatingFileHandler` will allow your log files to automatically rotate based on time (e.g., daily, hourly, etc.).

---

### ✅ Changes in This Version

* Uses `**TimedRotatingFileHandler**` to rotate logs.
* Default rotation: **daily** (`when='midnight'`, keep last 7 days).
* You can customize rotation parameters: `when`, `interval`, `backupCount`.

---

### 🧩 Updated `setup_logger` Function

```python
import logging
import os
from logging.handlers import TimedRotatingFileHandler

def setup_logger(
    name='my_logger',
    log_dir=None,
    log_file='app.log',
    level='INFO',
    when='midnight',
    interval=1,
    backup_count=7
):
    """
    Sets up a logger with optional file logging and time-based rotation.

    Parameters:
        name (str): Logger name.
        log_dir (str | None): Directory for the log file. If None, disables file logging.
        log_file (str): Log file name (used only if log_dir is provided).
        level (str): Logging level as string (e.g., 'DEBUG', 'INFO').
        when (str): Time interval for rotation ('S', 'M', 'H', 'D', 'midnight', 'W0'-'W6').
        interval (int): Interval at which to rotate the log.
        backup_count (int): Number of old log files to keep.

    Returns:
        logging.Logger: Configured logger.
    """
    # Convert level string to logging constant
    numeric_level = getattr(logging, level.upper(), None)
    if not isinstance(numeric_level, int):
        raise ValueError(f"Invalid log level: {level}")

    logger = logging.getLogger(name)
    logger.setLevel(numeric_level)

    # Prevent duplicate handlers if logger already configured
    if logger.hasHandlers():
        return logger

    # Formatter
    formatter = logging.Formatter('%(asctime)s - %(name)s - %(levelname)s - %(message)s')

    # Console handler
    console_handler = logging.StreamHandler()
    console_handler.setLevel(numeric_level)
    console_handler.setFormatter(formatter)
    logger.addHandler(console_handler)

    # File handler with timed rotation
    if log_dir:
        os.makedirs(log_dir, exist_ok=True)
        log_path = os.path.join(log_dir, log_file)

        file_handler = TimedRotatingFileHandler(
            filename=log_path,
            when=when,
            interval=interval,
            backupCount=backup_count,
            encoding='utf-8',
            utc=True  # or False, depending on your timezone preference
        )
        file_handler.setLevel(numeric_level)
        file_handler.setFormatter(formatter)
        logger.addHandler(file_handler)

    return logger
```

---

### 🧪 Example Usage

```python
if __name__ == '__main__':
    # Daily rotation at midnight, keep 7 days of logs
    logger = setup_logger(
        name='my_rotating_logger',
        log_dir='logs',
        log_file='my_app.log',
        level='debug',
        when='midnight',
        interval=1,
        backup_count=7
    )

    logger.debug("Debug message")
    logger.info("Info message")
    logger.warning("Warning message")
```

---

### ⏰ Rotation Options for `when`

| Value         | Meaning                     |
| ------------- | --------------------------- |
| `'S'`         | Seconds                     |
| `'M'`         | Minutes                     |
| `'H'`         | Hours                       |
| `'D'`         | Days                        |
| `'midnight'`  | Daily at midnight           |
| `'W0'`–`'W6'` | Weekly (0=Monday, 6=Sunday) |

---

Let me know if you want compressed rotated files or more customization like separate logs per level (`info.log`, `error.log`, etc).
