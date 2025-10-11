## 1. Python Script with Scheduling

Create a file called `scheduled_service.py`:

```python
#!/usr/bin/env python3
import time
import schedule
import logging
from datetime import datetime
import sys
import signal
import daemon
from daemon import pidfile

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(levelname)s - %(message)s',
    handlers=[
        logging.FileHandler('/var/log/scheduled_service.log'),
        logging.StreamHandler()
    ]
)

class ScheduledService:
    def __init__(self):
        self.running = True
        
    def signal_handler(self, signum, frame):
        """Handle shutdown signals"""
        logging.info("Received shutdown signal")
        self.running = False
        
    def job1(self):
        """Example function 1 - runs every hour"""
        logging.info("Job 1 executed - This runs every hour")
        # Add your actual task logic here
        current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        print(f"Job 1 completed at {current_time}")
        
    def job2(self):
        """Example function 2 - runs daily at 2:30 AM"""
        logging.info("Job 2 executed - This runs daily at 2:30 AM")
        # Add your actual task logic here
        current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        print(f"Job 2 completed at {current_time}")
        
    def job3(self):
        """Example function 3 - runs every 30 minutes"""
        logging.info("Job 3 executed - This runs every 30 minutes")
        # Add your actual task logic here
        current_time = datetime.now().strftime("%Y-%m-%d %H:%M:%S")
        print(f"Job 3 completed at {current_time}")
    
    def setup_schedule(self):
        """Setup the scheduling for all jobs"""
        # Schedule job1 to run every hour
        schedule.every().hour.do(self.job1)
        
        # Schedule job2 to run daily at 2:30 AM
        schedule.every().day.at("02:30").do(self.job2)
        
        # Schedule job3 to run every 30 minutes
        schedule.every(30).minutes.do(self.job3)
        
        # Add more schedules as needed
        # schedule.every().monday.at("09:00").do(self.job4)
        # schedule.every(10).minutes.do(self.job5)
        
        logging.info("Scheduled jobs setup completed")
        
    def run(self):
        """Main execution loop"""
        logging.info("Scheduled service started")
        
        # Setup signal handlers for graceful shutdown
        signal.signal(signal.SIGTERM, self.signal_handler)
        signal.signal(signal.SIGINT, self.signal_handler)
        
        # Setup the schedule
        self.setup_schedule()
        
        # Main loop
        while self.running:
            try:
                schedule.run_pending()
                time.sleep(60)  # Check every minute
            except Exception as e:
                logging.error(f"Error in main loop: {e}")
                time.sleep(60)
                
        logging.info("Scheduled service stopped")

def main():
    service = ScheduledService()
    service.run()

if __name__ == "__main__":
    # Check if we should run as daemon
    if len(sys.argv) > 1 and sys.argv[1] == "--daemon":
        # Run as daemon
        with daemon.DaemonContext(
            pidfile=pidfile.TimeoutPIDLockFile('/var/run/scheduled_service.pid'),
            signal_map={
                signal.SIGTERM: lambda signum, frame: None,
                signal.SIGINT: lambda signum, frame: None,
            }
        ):
            main()
    else:
        # Run in foreground
        main()
```

## 2. Systemd Service File

Create a systemd service file at `/etc/systemd/system/scheduled_service.service`:

```ini
[Unit]
Description=Scheduled Python Service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/python3 /path/to/your/scheduled_service.py --daemon
WorkingDirectory=/path/to/your/
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

## 3. Installation Script

Create an installation script `install_service.sh`:

```bash
#!/bin/bash

# Installation script for scheduled service

SERVICE_NAME="scheduled_service"
SCRIPT_DIR="/opt/scheduled_service"
SCRIPT_PATH="$SCRIPT_DIR/scheduled_service.py"
SERVICE_FILE="/etc/systemd/system/${SERVICE_NAME}.service"

echo "Installing Scheduled Python Service..."

# Check if running as root
if [ "$EUID" -ne 0 ]; then
    echo "Please run as root"
    exit 1
fi

# Create directory
echo "Creating installation directory..."
mkdir -p $SCRIPT_DIR

# Copy script
echo "Copying script..."
cp scheduled_service.py $SCRIPT_PATH
chmod +x $SCRIPT_PATH

# Install required packages
echo "Installing required Python packages..."
pip3 install schedule python-daemon

# Create service file
echo "Creating systemd service file..."
cat > $SERVICE_FILE << EOF
[Unit]
Description=Scheduled Python Service
After=network.target

[Service]
Type=simple
User=root
ExecStart=/usr/bin/python3 $SCRIPT_PATH --daemon
WorkingDirectory=$SCRIPT_DIR
Restart=always
RestartSec=10
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
EOF

# Create log directory and file
echo "Setting up logging..."
touch /var/log/scheduled_service.log
chmod 644 /var/log/scheduled_service.log

# Reload systemd and enable service
echo "Enabling service..."
systemctl daemon-reload
systemctl enable $SERVICE_NAME

echo "Installation complete!"
echo "You can now start the service with: systemctl start $SERVICE_NAME"
echo "View logs with: journalctl -u $SERVICE_NAME -f"
```

## 4. Requirements File

Create `requirements.txt`:

```txt
schedule==1.2.0
python-daemon==3.0.1
```

## 5. Usage Instructions

### Installation:
```bash
# Make the installation script executable
chmod +x install_service.sh

# Run installation
sudo ./install_service.sh
```

### Managing the Service:
```bash
# Start the service
sudo systemctl start scheduled_service

# Stop the service
sudo systemctl stop scheduled_service

# Check status
sudo systemctl status scheduled_service

# View logs
sudo journalctl -u scheduled_service -f

# Enable to start on boot (already done by install script)
sudo systemctl enable scheduled_service
```

### Manual Testing:
```bash
# Run in foreground for testing
python3 scheduled_service.py

# Run as daemon manually
python3 scheduled_service.py --daemon
```

## 6. Customization

To customize for your needs:

1. **Modify the jobs**: Replace `job1()`, `job2()`, `job3()` with your actual functions
2. **Change schedules**: Modify the `setup_schedule()` method with your desired timing
3. **Add error handling**: Enhance error handling in your specific functions
4. **Modify logging**: Adjust log levels and formats as needed

## Key Features:

- ✅ Runs as a system service
- ✅ Persists across reboots
- ✅ Proper logging to both file and journal
- ✅ Graceful shutdown handling
- ✅ Automatic restart on failure
- ✅ Easy to customize schedules and functions

The service will automatically start on boot and restart if it fails, ensuring your scheduled functions run reliably.
