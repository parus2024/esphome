# Guide to Auto-Compiling YAML Files in Detached ESPHome Containers

**Author:** Parus
**Date:** October 27, 2025  
**Version:** 1.0  

---

## Introduction
This is a detailed guide to setting up and using auto-compilation of YAML files for ESPHome in detached containers (run with the `-d` flag). It's suitable for working with the ESPHome web interface (accessible on ports like 2575 for version 2025.7.5). Since containers are detached, the CLI command `esphome` is not directly available in the LXC shell—it needs to be executed inside the container using `docker exec`. We'll also include a script for automation, tips on locales, background execution, and additional actions. Everything step by step for convenience!

**Note:** All commands and scripts are intended for execution in an LXC environment on Proxmox or similar. Ensure you have root privileges.

---

## 1. Setting Up Detached ESPHome Containers
Detached containers allow ESPHome to run in the background with access via web UI. The container name usually matches the version, e.g., `esphome-2025-7-5` for version 2025.7.5.

- **Checking Running Containers:**  
  Run `docker ps` in LXC. The container should be listed (if not, start it), e.g.:  
  ```bash
  docker run -d --name esphome-2025-7-5 -p 2575:6052 -v /root/esphome-projects-2025.7.5:/config esphome/esphome:2025.7.5
  ```

- **Accessing the Web Interface:**  
  Open a browser and go to `http://<LXC-container-IP>:2575` (port depends on version). Here, you can upload/edit YAML files.

---

## 2. Recommended Compilation Method: Using `docker exec`
This allows compiling files inside the running container without stopping or entering interactive mode.

1. **Select the Container:** Ensure the container is running (e.g., `esphome-2025-7-5`). Check with `docker ps`.
2. **Run Compilation:**  
   ```bash
   docker exec -w /config esphome-2025-7-5 esphome compile test.yaml
   ```  
   - `-w /config`: Sets the working directory (where the folder `/root/esphome-projects-2025.7.5` is mounted).  
   - Replace `test.yaml` with your file name.  
   - For detailed output, add `--verbose`:  
     ```bash
     docker exec -w /config esphome-2025-7-5 esphome compile --verbose test.yaml
     ```
3. **Check the Result:** After compilation, in LXC, check:  
   ```bash
   ls -la /root/esphome-projects-2025.7.5/.esphome/build/test/.pioenvs/test/firmware.bin
   ```
   If the file exists—compilation was successful.

---

## 3. Alternative Compilation Methods
- **Via Web Interface:** Open a browser at `http://<LXC-IP>:2575`, upload a YAML, edit, and compile through the UI. Convenient for testing, but if LXC has no browser—use port-forwarding or SSH-tunnel from the Proxmox host.
- **Interactive Entry into Container:** For full access, enter:  
  ```bash
  docker exec -it esphome-2025-7-5 bash
  ```
  then `cd /config && esphome compile alarm-21.yaml`, exit with `exit`.
- **Automating Multiple Versions:** Use a Bash script with a loop that runs `docker exec` for each directory (example below).

**Directory Structure in /root:**  
```
/root/
  esphome-projects-2025.7.5/
    firmware.yaml
    sensor.yaml
  esphome-projects-2024.12.10/
    main.yaml
```  
The script will process folders starting with `esphome-projects-` and find `.yaml` files in them.

---

## 4. Setting Up UTF-8 Locale for Correct Cyrillic Display
If Cyrillic displays incorrectly in nano, configure the locale inside the container.

1. **Generate and Set Locale:**  
   ```bash
   locale-gen ru_RU.UTF-8
   dpkg-reconfigure locales
   ```  
   Or run `dpkg-reconfigure locales` and select `ru_RU.UTF-8` (or `en_US.UTF-8`).
2. **Install Package if Needed:**  
   ```bash
   apt update
   apt install locales
   ```
3. **Set Environment Variables:**  
   ```bash
   echo "export LANG=ru_RU.UTF-8" >> ~/.bashrc
   echo "export LC_ALL=ru_RU.UTF-8" >> ~/.bashrc
   source ~/.bashrc
   ```
4. **Verify:**  
   ```bash
   locale
   ```
   It should display `LANG=ru_RU.UTF-8` and `LC_ALL=ru_RU.UTF-8`. Now Cyrillic in nano should work correctly.

---

## 5. Running the Script in Background Mode
A standard Bash script will stop when the console is closed. To run it in the background:

- **Simple Background with `&`:**  
  ```bash
  ./your_script.sh &
  ```
  The command line will be free immediately.
- **With `nohup` (to continue when terminal closes):**  
  ```bash
  nohup ./your_script.sh > /path/to/logs/mylog.out 2>&1 &
  ```
  - Output will be saved to `mylog.out`.  
  - For your script:  
    ```bash
    nohup ./compile_esphome_2025_7_5.sh > /root/esphome_compile.log 2>&1 &
    ```
- **With `tmux` or `screen`:** Creates a virtual terminal.  
  - Install: `apt install tmux` or `apt install screen`.  
  - Run: `tmux` (or `screen`), then `./your_script.sh`, detach (`Ctrl+B D` in tmux), the script continues running.

---

## 6. Final Script `compile_esphome_2025_7_5.sh`
Here's the ready-to-use script for automation. It processes YAML files in the directory, skips `secrets.yaml`, checks for `firmware.bin`, and compiles if needed. Copy the code below into a file.

```bash
#!/bin/bash

# ================================
# Variable Setup
# ================================
# Enter the desired esphome version
VERSION="2025.7.5"
DIR="/root/esphome-projects-${VERSION}"
CONTAINER_NAME="esphome-${VERSION//./-}"
DOCKER_CONFIG_PATH="/config"
LOG_FILE="/root/esphome-projects-${VERSION}/compile_esphome_${VERSION//./_}.log"

# Clear log file at start
: > "$LOG_FILE"

# Helper function for logging
log() {
  echo "$*" | tee -a "$LOG_FILE"
}

# ================================
# Execution Start Time
# ================================
start_time=$(date +%s)
script_start_time=$(date)

# ================================
# Initialize Counters and Arrays
# ================================
processed=0
skipped=0
errors=0
declare -a error_files=()

# ================================
# Checks
# ================================

if [ ! -d "$DIR" ]; then
  log "DIRECTORY $DIR NOT FOUND!"
  exit 1
fi

if ! docker ps --format="{{.Names}}" | grep -q "^$CONTAINER_NAME$"; then
  log "CONTAINER \"$CONTAINER_NAME\" IS NOT RUNNING!"
  exit 1
fi

# ================================
# File Processing
# ================================
shopt -s lastpipe

mapfile -t yaml_files < <(find "$DIR" -maxdepth 1 -name "*.yaml" -printf "%f\n")
if [ ${#yaml_files[@]} -eq 0 ]; then
  log "No YAML files found in $DIR."
  exit 1
fi

for filename in "${yaml_files[@]}"; do
  log "----------------------------------------"
  log "Processing file: $filename"
  ((processed++))

  if [[ "$filename" == "secrets.yaml" ]]; then
    log "Skipping $filename — this is a secrets file."
    continue
  fi
  base_name="${filename%.yaml}"
  bin_path_in_container="/config/.esphome/build/$base_name/.pioenvs/$base_name/firmware.bin"
  log "Found .bin files: $bin_path_in_container"

  if docker exec "$CONTAINER_NAME" test -f "$bin_path_in_container"; then
    mod_time=$(docker exec "$CONTAINER_NAME" stat -c %Y "$bin_path_in_container")
    mod_time_readable=$(docker exec "$CONTAINER_NAME" date -d "@$mod_time" '+%Y-%m-%d %H:%M:%S')
    log "File $bin_path_in_container already exists. Skipping build for $filename. Last modification time: $mod_time_readable"
    ((skipped++))
    continue
  fi

  log "Location where .bin is placed: $bin_path_in_container"

  # copying
  log "COPYING $filename to container..."
  if docker cp "$DIR/$filename" "$CONTAINER_NAME":"$DOCKER_CONFIG_PATH/"; then
    log "Copy successful."
  else
    log "Error copying file $filename."
    ((errors++))
    error_files+=("$filename")
    continue
  fi

  # compilation
  log "Compiling $filename inside container..."
  if docker exec -w "$DOCKER_CONFIG_PATH" "$CONTAINER_NAME" esphome compile "$filename"; then
    log "✅ SUCCESS: $filename (compilation completed: $(date))"
  else
    log "❌ Error compiling $filename (error: $(date))"
    ((errors++))
    error_files+=("$filename")
  fi
done

# ================================
# Execution End Time and Total Duration
# ================================
end_time=$(date +%s)
script_end_time=$(date)

duration=$((end_time - start_time))
minutes=$((duration / 60))
seconds=$((duration % 60))

log "========================================"
log "Processing completed!"
log "Files processed: $processed"
log "Skipped (already have .bin): $skipped"
log "Compilation errors: $errors"

if [ $errors -gt 0 ]; then
  echo "Files with errors:" | tee -a "$LOG_FILE"
  for f in "${error_files[@]}"; do
    echo "- $f" | tee -a "$LOG_FILE"
  done
fi

log "Total time: ${duration} seconds (${minutes} minutes and ${seconds} seconds)."
log "Start: $script_start_time"
log "End: $script_end_time"
log "Full log available at: $LOG_FILE"

#cat "$LOG_FILE" # Output log file to console
```

---

## 7. Working with the Script
- **Editing:** `nano /root/esphome-projects-2025.7.5/compile_esphome_2025_7_5.sh`.
- **Viewing the Log:** `cat /root/esphome-projects-2025.7.5/compile_esphome_2025_7_5.log`.
- **Running:**  
  - Grant permissions: `chmod +x /root/esphome-projects-2025.7.5/compile_esphome_2025_7_5.sh`.  
  - Execute: `bash /root/esphome-projects-2025.7.5/compile_esphome_2025_7_5.sh` (or `./compile_esphome_2025_7_5.sh`).

---

## 8. Additional Actions
- **Path to Compiled bin File:** `/root/esphome-projects-2025.7.5/.esphome/build/alarm-33/.pioenvs/alarm-33/firmware.bin` (replace with your file name).
- **Deleting a Folder:** `rm -rf /root/esphome-projects-2024-6-0/`.
- **Deleting the Script:** `rm /root/esphome-projects-2025.7.5/compile_esphome_2025_7_5.sh`.
- **Checking for bin File Presence:**  
  ```bash
  docker exec esphome-2025-7-5 test -f "/config/.esphome/build/mirror-ld2420/.pioenvs/mirror-ld2420/firmware.bin" && echo "Exists" || echo "Does not exist"
  ```
- **SSH Access via PuTTY to Container:**  
  1. Install SSH: `apt update && apt install openssh-server`.  
  2. Enable: `systemctl enable ssh && systemctl start ssh`.  
  3. Check status: `service sshd status`.  
  4. Edit `/etc/ssh/sshd_config` (uncomment/add: `PasswordAuthentication yes`, `PermitRootLogin yes`).  
  5. Restart: `service ssh restart`.  
     Now connect via PuTTY to the LXC IP with root password.
