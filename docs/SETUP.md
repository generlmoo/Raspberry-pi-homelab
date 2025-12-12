# Setup (Pi OS Lite + NVMe + OMV + Pi-hole + Jellyfin + Filebrowser)

This is the "commands and steps" companion to `Raspberry-pi-homelab.txt`.

## Key rules / lessons learned
- Use Raspberry Pi OS Lite (command-line, minimal overhead).
- Keep the OS on the microSD (do not install the OS image to the NVMe).
- The 2TB NVMe (via M.2 HAT) and the 8TB drive are data disks.
- For Filebrowser: keep its config/database on the OS disk (microSD); only point Filebrowser at mounted data directories.

## Base OS (Raspberry Pi OS Lite)
1) Flash Raspberry Pi OS Lite to the microSD using Raspberry Pi Imager.
2) Boot the Pi and SSH in.
3) Update:
   - `sudo apt update`
   - `sudo apt full-upgrade -y`
   - `sudo reboot`

## Mount the 2TB NVMe (example commands)
Identify the disk and partitions first:
- `lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINTS,MODEL`
- `sudo blkid`

Typical device names:
- NVMe disk: `/dev/nvme0n1` (partition: `/dev/nvme0n1p1`)

### If the NVMe already has a filesystem
1) Create a mount point:
   - `sudo mkdir -p /mnt/nvme2tb`
2) Mount it (replace with your partition name):
   - `sudo mount /dev/nvme0n1p1 /mnt/nvme2tb`
3) Persist using `/etc/fstab` (use the UUID from `blkid`):
   - `sudo nano /etc/fstab`
   - Add:
     - `UUID=<uuid-from-blkid> /mnt/nvme2tb ext4 defaults,noatime 0 2`
4) Test:
   - `sudo umount /mnt/nvme2tb`
   - `sudo mount -a`
   - `df -h`

### If the NVMe is blank and you need to partition + format it
Only do this if you're 100% sure you picked the right disk.
1) Create a GPT partition + single ext4 partition:
   - `sudo parted -s /dev/nvme0n1 mklabel gpt mkpart primary ext4 0% 100%`
2) Format:
   - `sudo mkfs.ext4 -L nvme2tb /dev/nvme0n1p1`
3) Mount + fstab: follow the steps above.

## Mount the 8TB drive (example commands)
Identify it:
- `lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINTS,MODEL`

Typical device names:
- USB/SATA disk: `/dev/sda` (partition: `/dev/sda1`)

1) Create mount point:
   - `sudo mkdir -p /mnt/usb8tb`
2) Mount (replace with the correct partition):
   - `sudo mount /dev/sda1 /mnt/usb8tb`
3) Persist via `/etc/fstab` using UUID:
   - `sudo nano /etc/fstab`
   - Add:
     - `UUID=<uuid-from-blkid> /mnt/usb8tb ext4 defaults,noatime 0 2`
4) Test:
   - `sudo mount -a`
   - `df -h`

## Install OpenMediaVault (OMV)
Installed on top of Raspberry Pi OS Lite:
- `sudo apt update && sudo apt install -y wget`
- `wget -O - https://github.com/OpenMediaVault-Plugin-Developers/installScript/raw/master/install | sudo bash`

After install:
- Log into OMV web UI.
- Add the mounted disks as filesystems (or ensure OMV sees them).
- Create shared folders pointing at `/mnt/nvme2tb` and/or `/mnt/usb8tb`.

## Install Pi-hole
- `curl -sSL https://install.pi-hole.net | bash`

Upstream DNS example (Cloudflare):
- `1.1.1.1`
- `1.0.0.1`

Port note:
- If OMV and Pi-hole both want port 80, move one of them to a different port so both UIs work.

## Install Filebrowser
Keep Filebrowser config on the OS disk; browse mounted data directories.

Docker-based example:
1) Install Docker:
   - `curl -fsSL https://get.docker.com | sh`
   - `sudo usermod -aG docker $USER`
   - `newgrp docker`
2) Run Filebrowser:
   - `mkdir -p $HOME/filebrowser`
   - `docker run -d --name filebrowser --restart unless-stopped -p 8080:80 -v $HOME/filebrowser:/database -v /mnt/nvme2tb:/srv/nvme2tb -v /mnt/usb8tb:/srv/usb8tb filebrowser/filebrowser`

## Install Jellyfin
Docker-based example:
- `docker run -d --name jellyfin --restart unless-stopped -p 8096:8096 -v /mnt/nvme2tb:/media/nvme2tb -v /mnt/usb8tb:/media/usb8tb jellyfin/jellyfin`

Then add Jellyfin libraries pointing at:
- `/media/nvme2tb/...`
- `/media/usb8tb/...`

## Cloudflare (domain + tunnels)
High level:
1) Buy a domain and add it to Cloudflare.
2) Create a tunnel in Cloudflare Zero Trust.
3) Install `cloudflared` on the Pi and authenticate.
4) Map hostnames to local services (examples):
   - `jellyfin.<your-domain>` -> `http://localhost:8096`
   - `files.<your-domain>` -> `http://localhost:8080`

Security tip:
- Protect admin UIs with Cloudflare Access (recommended), or do not expose them via tunnel.

## Tailscale (secure remote access)
- `curl -fsSL https://tailscale.com/install.sh | sh`
- `sudo tailscale up --ssh`

Tip:
- Use Tailscale for private SSH/admin access if something goes wrong.

