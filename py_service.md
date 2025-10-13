Perfecto ✅
Te ayudaré a crear **un servicio persistente de Linux (systemd)** que ejecuta un script Python ubicado en `/opt/storage/collector/storage_collector.py`.
El script usará `schedule`, soportará un **modo de prueba** mediante `argparse`, y generará **logs separados** para eventos normales y errores.

---

## 🧩 Estructura general

```
/opt/storage/collector/
├── storage_collector.py
├── logs/
│   ├── storage_collector.log
│   └── storage_collector_error.log
└── storage_collector.service  (archivo systemd)
```

---

## 🐍 1. Script principal: `storage_collector.py`

Guárdalo en `/opt/storage/collector/storage_collector.py`
y asegúrate de que tenga permisos de ejecución:

```bash
sudo chmod +x /opt/storage/collector/storage_collector.py
```

Aquí tienes el código completo:

```python
#!/usr/bin/env python3
import schedule
import time
import argparse
import logging
from datetime import datetime
import sys
import os

# ========================
# CONFIGURACIÓN DE LOGS
# ========================
LOG_DIR = "/opt/storage/collector/logs"
os.makedirs(LOG_DIR, exist_ok=True)

info_log_file = os.path.join(LOG_DIR, "storage_collector.log")
error_log_file = os.path.join(LOG_DIR, "storage_collector_error.log")

# Logger general
logger = logging.getLogger("storage_collector")
logger.setLevel(logging.INFO)

# Handler de información
info_handler = logging.FileHandler(info_log_file)
info_handler.setLevel(logging.INFO)

# Handler de errores
error_handler = logging.FileHandler(error_log_file)
error_handler.setLevel(logging.ERROR)

# Formatos
formatter = logging.Formatter("%(asctime)s - %(levelname)s - %(message)s")
info_handler.setFormatter(formatter)
error_handler.setFormatter(formatter)

# Agregar handlers al logger
logger.addHandler(info_handler)
logger.addHandler(error_handler)

# ========================
# FUNCIONES DEL COLECTOR
# ========================

def collect_data():
    """Ejemplo de tarea programada."""
    logger.info("Iniciando tarea: colectar datos.")
    try:
        # Simula proceso
        time.sleep(2)
        logger.info("Datos recolectados exitosamente.")
    except Exception as e:
        logger.error(f"Error al recolectar datos: {e}")

def clean_old_files():
    """Ejemplo de otra tarea programada."""
    logger.info("Iniciando tarea: limpieza de archivos antiguos.")
    try:
        # Simula proceso
        time.sleep(1)
        logger.info("Archivos antiguos eliminados.")
    except Exception as e:
        logger.error(f"Error durante la limpieza: {e}")

# ========================
# PROGRAMACIÓN DE TAREAS
# ========================
def schedule_jobs():
    # Ejecutar cada 10 minutos
    schedule.every(10).minutes.do(collect_data)

    # Ejecutar todos los días a las 02:00 AM
    schedule.every().day.at("02:00").do(clean_old_files)

    logger.info("Tareas programadas correctamente.")

# ========================
# MAIN
# ========================
def main():
    parser = argparse.ArgumentParser(description="Storage Collector Service")
    parser.add_argument(
        "--test",
        help="Modo de prueba: ejecuta las tareas inmediatamente",
        action="store_true"
    )
    args = parser.parse_args()

    if args.test:
        logger.info("Modo de prueba activado.")
        collect_data()
        clean_old_files()
        logger.info("Ejecución de prueba completada.")
        sys.exit(0)

    # Modo normal (servicio)
    schedule_jobs()
    logger.info("Iniciando bucle de ejecución...")

    while True:
        schedule.run_pending()
        time.sleep(30)


if __name__ == "__main__":
    main()
```

---

## 🧰 2. Crear el servicio systemd

Crea el archivo del servicio:

```bash
sudo nano /etc/systemd/system/storage_collector.service
```

Contenido del archivo:

```ini
[Unit]
Description=Storage Collector Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/bin/python3 /opt/storage/collector/storage_collector.py
Restart=always
User=root
WorkingDirectory=/opt/storage/collector
StandardOutput=syslog
StandardError=syslog

[Install]
WantedBy=multi-user.target
```

---

## ⚙️ 3. Activar y habilitar el servicio

```bash
sudo systemctl daemon-reload
sudo systemctl enable storage_collector.service
sudo systemctl start storage_collector.service
```

Verifica su estado:

```bash
sudo systemctl status storage_collector.service
```

Ver logs del sistema:

```bash
journalctl -u storage_collector.service -f
```

Y logs del script:

```bash
tail -f /opt/storage/collector/logs/storage_collector.log
tail -f /opt/storage/collector/logs/storage_collector_error.log
```

---

## 🧪 4. Modo de prueba manual

Puedes correr las tareas inmediatamente sin esperar la programación:

```bash
python3 /opt/storage/collector/storage_collector.py --test
```

Esto ejecutará las funciones una sola vez y saldrá.

---

¿Quieres que te agregue una tarea adicional de ejemplo (por ejemplo, subir datos a un servidor remoto o comprimir archivos)?
Puedo incluirla con su programación en el mismo esquema.
