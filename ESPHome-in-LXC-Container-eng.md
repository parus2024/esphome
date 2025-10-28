# Setting Up ESPHome in Proxmox Using an LXC Container

This guide helps you set up ESPHome in Proxmox VE. We use an LXC container (a lightweight OS-level container) to bypass HAOS limitations (where the ESPHome add-on is always on the latest version). This allows experimenting with different ESPHome versions (e.g., 2023.12.0 or 2025.7.5) without affecting Home Assistant.

## Prerequisites
- Proxmox VE installed and running.
- Access to Proxmox Web UI (usually https://<host-IP>:8006).
- Basic Linux knowledge (apt, systemctl commands).
- Free space on the Proxmox host (at least 10-20 GB for the container + Docker images).

## Why an LXC Container?
- HAOS doesn't allow direct Docker installation (without risking stability).
- LXC isolates ESPHome from HA, provides version flexibility, and uses fewer resources than a full VM.
- You can run multiple ESPHome containers with different versions on different ports.

## Section 1: Creating an LXC Container in Proxmox

1. **Access Proxmox Web UI:**  
   Open a browser and go to https://<host-IP>:8006 (replace with your host's IP).  
   Log in as root or an administrator.

2. **Create a new container:**  
   Select your node in the left menu.  
   Click **Create CT (Create Container)**.  
   - **General:**  
     - Hostname: ESPHome-Container (or any name).  
     - Password: Set a password for root (remember it).  
     - Unprivileged container: Leave checked (for security).  
   - **Template:**  
     Select Ubuntu or Debian (download if not available: click Templates and search).  
   - **Root Disk:**  
     - Storage: Select local-lvm or another storage with space.  
     - Disk size: 10-20 GB (enough to start; can expand later).  
   - **CPU:**  
     - Cores: 1-2 (sufficient for ESPHome compilation).  
   - **Memory:**  
     - Memory: 1024-2048 MB (1-2 GB RAM).  
   - **Network:**  
     - Bridge: vmbr0 (or the same as your HAOS VM for network and HA access).  
     - IPv4: DHCP (automatic IP) or set a static address manually.  
   Click **Create** and wait for completion (1-5 minutes).

3. **Start the container:**  
   In the container list (CT), select ESPHome-Container.  
   Click **Start**.  
   Check status: Should be "Running".

4. **Connect to the container:**  
   In Proxmox UI: Select the container > Console (opens a terminal inside).  
   Or via SSH (if network is set up): `ssh root@<container-IP>` (check IP in the container's network settings in Proxmox UI).  
   Log in as root with the password you set during creation.  
   *Note: LXC uses the host's kernel but is isolated. Now you're inside Ubuntu/Debian.*

## Section 2: Installing Docker and Preparation

1. **Update the system and install Docker:**  
   In the container console (as root), run in order:  
   - `apt update`: Updates the package list.  
   - `apt upgrade -y`: Upgrades the system (with -y to skip confirmations).  
   - `apt install docker.io -y`: Installs Docker.  
   - `systemctl start docker && systemctl enable docker`: Starts and enables Docker auto-start.  
   - `usermod -aG docker root`: Adds root to the Docker group (to avoid sudo for Docker commands).  
   Check: `docker --version` — outputs the version (e.g., 24.x.x).

2. **Create a folder for projects:**  
   `mkdir ~/esphome-projects` (this is /root/esphome-projects if you're root).  
   Copy your YAML files here if needed, using other copy methods. Also include folders with fonts, images, attachments, and the secrets file.  
   *Note: Docker is needed to run ESPHome containers. The ~/esphome-projects folder is mounted into the containers.*

## Section 3: Setting Up ESPHome with Different Versions

ESPHome is a tool for flashing ESP32/Arduino devices. We run multiple Docker containers with different versions on unique ports.

1. **Pull ESPHome images:**  
   ```
   docker pull esphome/esphome:2023.12.0
   docker pull esphome/esphome:latest
   docker pull esphome/esphome:2024.6.0
   docker pull esphome/esphome:2025.7.5
   ```  
   (or other tags from https://hub.docker.com/r/esphome/esphome/tags).  
   Check: `docker images` — lists downloaded images.

2. **Run containers with versions:**  
   Use unique ports to avoid conflicts. For convenience, make ports "similar" to the version (e.g., 23120 for 2023.12.0).  
   Example commands (run one by one in the container console):  
   - For 2023.12.0 (port 23120):  
     ```
     docker run -d --name esphome-2023-12-0 --restart unless-stopped -v ~/esphome-projects:/config -p 23120:6052 -p 23121:6123 esphome/esphome:2023.12.0
     ```  
   - For latest (port 6053):  
     ```
     docker run -d --name esphome-latest --restart unless-stopped -v ~/esphome-projects:/config -p 6053:6052 -p 6124:6123 esphome/esphome:latest
     ```  
   - For 2024.6.0 (port 2460):  
     ```
     docker run -d --name esphome-2024-6-0 --restart unless-stopped -v ~/esphome-projects:/config -p 2460:6052 -p 2461:6123 esphome/esphome:2024.6.0
     ```  
   - For 2025.7.5 (port 2575):  
     ```
     docker run -d --name esphome-2025-7-5 --restart unless-stopped -v ~/esphome-projects:/config -p 2575:6052 -p 2576:6123 esphome/esphome:2025.7.5
     ```  
   *Flag explanations:*  
   - `-d`: Run in background (detached).  
   - `--name`: Unique container name.  
   - `--restart unless-stopped`: Auto-restart on container reboot.  
   - `-v ~/esphome-projects:/config`: Mounts the projects folder (files shared across containers).  
   - `-p 2575:6052`: Port forwarding (external:internal) for the web interface. Change the external port for each container.  
   - Second port (e.g., 2576:6123) — for OTA/API to avoid conflicts.

3. **Check and access:**  
   `docker ps`: List running containers (should be "Up").  
   In browser: http://<container-IP>:2575 (e.g., http://192.168.1.100:2575 for version 2025.7.5). Container IP is the same as the host or check in Proxmox.  
   Upload YAML in the web interface, compile, and flash ESP.

4. **Manage containers:**  
   - Stop: `docker stop esphome-2025-7-5`  
   - Start: `docker start esphome-2025-7-5`  
   - Logs: `docker logs esphome-2025-7-5`  
   - Enter inside: `docker exec -it esphome-2025-7-5 bash` (for manual commands).  
   *Note: Now you have multiple ESPHome versions on different ports. Projects are shared, but test YAML firmware on different versions.*

## Section 4: Setting Up FTP for File Uploads

FTP allows uploading files (fonts, images) to ~/esphome-projects. We install vsftpd (a simple FTP server). Alternatives: SCP/SFTP (no installation) or manual copying.

1. **Install vsftpd:**  
   In the container console (as root):  
   ```
   apt update
   apt install vsftpd -y
   systemctl start vsftpd
   systemctl enable vsftpd
   ```

2. **Configure:**  
   Backup: `cp /etc/vsftpd.conf /etc/vsftpd.conf.backup`  
   Edit: `nano /etc/vsftpd.conf` (or vim).  
   Add/change at the end:  
   ```
   write_enable=YES
   local_enable=YES
   chroot_local_user=YES
   allow_writeable_chroot=YES
   user_sub_token=$USER
   local_root=/root/esphome-projects  # If you're root; for regular user: /home/$USER/esphome-projects
   ```  
   - `write_enable=YES`: Allows writing (uploading files).  
   - `local_enable=YES`: Access for system users.  
   - `chroot_local_user=YES`: Restricts users to their folder.  
   - `allow_writeable_chroot=YES`: Allows writing in chroot.  
   - `local_root`: Sets root folder to ~/esphome-projects.  
   Save (Ctrl+O, Enter, Ctrl+X in nano) and restart: `systemctl restart vsftpd`.

3. **Create an FTP user (recommended if not root):**  
   ```
   useradd -m esphomeftp
   passwd esphomeftp  # enter password
   chown -R esphomeftp:esphomeftp ~/esphome-projects
   ln -s ~/esphome-projects /home/esphomeftp/projects  # symbolic link for access
   ```  
   In vsftpd.conf, change `local_root=/home/esphomeftp/projects`.

4. **Connect with FTP client:**  
   Use FileZilla or another, or command line: `ftp <container-IP>` (login: esphomeftp or root, password).  
   Upload files to the folder (e.g., fonts/ or images/).  
   If stuck: Exit with `bye` or `quit`.

5. **Troubleshooting:**  
   - Logs: `journalctl -u vsftpd` or `/var/log/vsftpd.log`.  
   - If "permission denied": Check permissions (`chmod 770 ~/esphome-projects`, `chown root:esphomeftp ~/esphome-projects`).  
   - Firewall: `ufw allow 21/tcp` (if installed).  
   - For root: Add `allow_root_login=YES` to vsftpd.conf (but better use esphomeftp).

## Section 5: Expanding LXC Container Disk (If Space Runs Out)

1. **Via Proxmox Web UI:**  
   Select container > Resources > Disk > Resize.  
   Enter new size (e.g., +10G) > Resize Disk.

2. **Update filesystem inside container:**  
   `df -h` (check path, e.g., /dev/mapper/pve-vm--101--disk--0).  
   `resize2fs /dev/mapper/pve-vm--101--disk--0` (for ext4; replace ID with yours, check `lsblk`).

3. **Via CLI on Proxmox host:**  
   On host: `pct config <ID>` (<ID> is container ID).  
   `pct resize <ID> rootfs +10G`.  
   Then update FS inside as above. 
