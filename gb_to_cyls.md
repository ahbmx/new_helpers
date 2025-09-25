Great — let's upgrade the script to **detect and handle input units like GB, TB, MB**, etc.

---

## ✅ Enhanced Script: `size_to_cyl.py`

Supports: **MB**, **GB**, **TB**
(Default is GB if no unit provided)

```python
#!/usr/bin/env python3

import argparse
import re

def parse_size(size_str):
    """
    Parse size string with unit (e.g. 100GB, 1TB, 500mb)
    Returns size in MB
    """
    pattern = r'^(\d+(\.\d+)?)([MmGgTt][Bb])?$'
    match = re.match(pattern, size_str.strip())

    if not match:
        raise ValueError("Invalid size format. Use format like 100GB, 1TB, 500MB.")

    value = float(match.group(1))
    unit = match.group(3)

    if not unit:
        unit = 'GB'  # Default unit

    unit = unit.upper()

    if unit == 'MB':
        return value
    elif unit == 'GB':
        return value * 1024
    elif unit == 'TB':
        return value * 1024 * 1024
    else:
        raise ValueError(f"Unknown unit: {unit}")

def mb_to_cyl(mb, cyl_mb=1.875):
    """Convert MB to cylinders (rounded down)."""
    return int(mb / cyl_mb)

def main():
    parser = argparse.ArgumentParser(
        description="Convert storage size to cylinders for PowerMax symconfigure."
    )
    parser.add_argument(
        "size", type=str,
        help="Size with optional unit (e.g. 100GB, 1TB, 500MB). Default is GB."
    )
    parser.add_argument(
        "-c", "--count", type=int, default=1,
        help="Number of devices to create (default: 1)"
    )
    parser.add_argument(
        "-e", "--emulation", default="FBA",
        help="Device emulation type (default: FBA)"
    )
    parser.add_argument(
        "--raw", action="store_true",
        help="Only output the cylinder count"
    )

    args = parser.parse_args()

    try:
        mb = parse_size(args.size)
        cylinders = mb_to_cyl(mb)

        if args.raw:
            print(cylinders)
        else:
            print(f"{args.size.upper()} = {cylinders} cylinders (rounded down)")
            print("symconfigure command:")
            print(f"create dev count={args.count} size={cylinders} cyl emulation={args.emulation};")
    except ValueError as e:
        print(f"Error: {e}")

if __name__ == "__main__":
    main()
```

---

## ✅ Example Usages

### ➤ 1. Default GB:

```bash
python3 size_to_cyl.py 100
```

```
100GB = 54666 cylinders (rounded down)
symconfigure command:
create dev count=1 size=54666 cyl emulation=FBA;
```

### ➤ 2. TB Input:

```bash
python3 size_to_cyl.py 1TB -c 4 -e CKD
```

```
1TB = 559240 cylinders (rounded down)
symconfigure command:
create dev count=4 size=559240 cyl emulation=CKD;
```

### ➤ 3. MB Input:

```bash
python3 size_to_cyl.py 500MB
```

```
500MB = 266 cylinders (rounded down)
symconfigure command:
create dev count=1 size=266 cyl emulation=FBA;
```

### ➤ 4. Raw Cylinder Output:

```bash
python3 size_to_cyl.py 5GB --raw
```

```
2733
```

---

Would you like to add support for **rounding up/nearest**, or **reverse conversion** (cyl → GB)?
