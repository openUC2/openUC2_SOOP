### Create the Startup Script

First, let's create the startup script:

```sh
#!/bin/bash

# Define variables
START_SCRIPT_PATH="$HOME/start_imswitch.sh"
SERVICE_FILE_PATH="/etc/systemd/system/start_imswitch.service"

# Create the startup script
cat << 'EOF' > $START_SCRIPT_PATH
#!/bin/bash
set -x

LOGFILE=/home/uc2/start_imswitch.log
exec > $LOGFILE 2>&1

echo "Starting IMSwitch Docker container and Chromium"

# Wait for the X server to be available
while ! xset q &>/dev/null; do
  echo "Waiting for X server..."
  sleep 2
done

export DISPLAY=:0

# Start Docker container in the background
echo "Running Docker container..."
nohup sudo docker run --rm -d -p 8001:8001 -p 2222:22 \
  -e HEADLESS=1 -e HTTP_PORT=8001 \
  -e CONFIG_FILE=example_uc2_hik_flowstop.json \
  -e UPDATE_GIT=1 -e UPDATE_CONFIG=0 \
  --privileged ghcr.io/openuc2/imswitch-noqt-x64:latest &

# Wait a bit to ensure Docker starts
sleep 10

# Start Chromium
echo "Starting Chromium..."
/usr/bin/chromium-browser --start-fullscreen --ignore-certificate-errors \
 --start-fullscreen  \
 --unsafely-treat-insecure-origin-as-secure=https://0.0.0.0:8001 \
  --force-device-scale-factor=0.7 \
  https://0.0.0.0:8001/imswitch/index.html 

echo "Startup script completed"
EOF

# Make the startup script executable
chmod +x $START_SCRIPT_PATH

echo "Startup script created at $START_SCRIPT_PATH and made executable."

# Create the systemd service file
sudo bash -c "cat << EOF > $SERVICE_FILE_PATH
[Unit]
Description=Start IMSwitch Docker and Chromium
After=display-manager.service
Requires=display-manager.service

[Service]
Type=simple
ExecStart=$START_SCRIPT_PATH
User=$USER
Environment=DISPLAY=:0
Restart=on-failure
TimeoutSec=300
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=graphical.target
EOF"

# Reload systemd, enable and start the new service
sudo systemctl daemon-reload
sudo systemctl enable start_imswitch.service
sudo systemctl start start_imswitch.service

echo "Systemd service created and enabled to start at boot."
```

### Instructions:

1. Save the combined script to a file, e.g., `setup_autostart.sh`.
2. Make the script executable:

   ```sh
   chmod +x setup_autostart.sh
   ```

3. Run the script:

   ```sh
   ./setup_autostart.sh
   ```

This will create the startup script, create a systemd service, and enable it to run at boot, ensuring that your Docker container and Chromium start automatically.
