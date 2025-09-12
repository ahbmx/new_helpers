Awesome — let’s build a single **CLI tool** that can:

1. **Upload** an Excel sheet to a PostgreSQL table.
2. **Download** data from PostgreSQL and export it to an Excel file.

---

## 🛠️ Tool Features

* Uses `argparse` for CLI arguments.
* Supports `--upload` or `--download` mode.
* Has `--input`, `--output`, `--table`, `--db-url` arguments.

---

## 📦 Requirements

Install dependencies:

```bash
pip install pandas sqlalchemy openpyxl psycopg2-binary
```

---

## 📜 `db_excel_tool.py`

```python
import argparse
import pandas as pd
import os
from sqlalchemy import create_engine
from sqlalchemy.dialects.postgresql import insert

# ---------- Functions ----------

def upload_excel_to_db(file_path, table_name, db_url):
    if not os.path.exists(file_path):
        raise FileNotFoundError(f"Excel file '{file_path}' not found.")

    df = pd.read_excel(file_path)

    if df.empty:
        raise ValueError("Excel file is empty.")

    engine = create_engine(db_url)

    with engine.begin() as conn:
        for _, row in df.iterrows():
            stmt = insert(table_name).values(row.to_dict())
            stmt = stmt.on_conflict_do_update(
                index_elements=['device_name'],  # Adjust as needed
                set_={col: stmt.excluded[col] for col in row.index if col != 'device_name'}
            )
            conn.execute(stmt)

    print(f"✅ Uploaded '{file_path}' to '{table_name}'.")

def export_db_to_excel(table_name, output_path, db_url):
    engine = create_engine(db_url)

    df = pd.read_sql(f"SELECT * FROM {table_name}", engine)

    if df.empty:
        print(f"⚠️ Table '{table_name}' is empty.")
    else:
        df.to_excel(output_path, index=False)
        print(f"✅ Exported '{table_name}' to '{output_path}'.")

# ---------- CLI Parser ----------

def main():
    parser = argparse.ArgumentParser(description="Upload/download Excel data to/from PostgreSQL.")

    mode_group = parser.add_mutually_exclusive_group(required=True)
    mode_group.add_argument("--upload", action="store_true", help="Upload Excel to DB")
    mode_group.add_argument("--download", action="store_true", help="Download table to Excel")

    parser.add_argument("--input", type=str, help="Path to input Excel file (for upload)")
    parser.add_argument("--output", type=str, help="Path to output Excel file (for download)")
    parser.add_argument("--table", type=str, required=True, help="Database table name")
    parser.add_argument("--db-url", type=str, required=True, help="PostgreSQL DB connection URL")

    args = parser.parse_args()

    if args.upload:
        if not args.input:
            parser.error("--upload requires --input")
        upload_excel_to_db(args.input, args.table, args.db_url)

    elif args.download:
        if not args.output:
            parser.error("--download requires --output")
        export_db_to_excel(args.table, args.output, args.db_url)

if __name__ == "__main__":
    main()
```

---

## 🧪 Example Usage

### 🔼 Upload Excel to PostgreSQL

```bash
python db_excel_tool.py --upload --input devices.xlsx --table device_inventory --db-url postgresql+psycopg2://user:pass@localhost:5432/mydb
```

### 🔽 Export DB Table to Excel

```bash
python db_excel_tool.py --download --output export.xlsx --table device_inventory --db-url postgresql+psycopg2://user:pass@localhost:5432/mydb
```

---

## 🧠 Tips

* Ensure `device_name` has a unique constraint in the database for upserts to work.
* For custom conflict handling (e.g., `device_id`), change the `index_elements` in the upsert logic.
* You can add optional logging or verbose mode easily if needed.

---

Would you like this tool packaged as a pip-installable module, or turned into a `.exe` with PyInstaller?
