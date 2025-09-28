# Home Server Setup

A complete Docker-based home server solution with media streaming, monitoring, networking, and automation tools.

## 🚀 Features

- **Media Server**: Plex/Jellyfin with *arr stack automation
- **Monitoring**: Glances system monitoring with web interface
- **Dashboard**: Homepage for centralized service management
- **Networking**: NPM reverse proxy with SSL/TLS support
- **Storage**: MergerFS pooled storage with automated mounting
- **Security**: Let's Encrypt SSL certificates
- **Container Management**: Portainer for web-based Docker management

## 📋 Prerequisites

- Ubuntu 20.04+ or compatible Linux distribution
- Minimum 8GB RAM (16GB+ recommended)
- Multiple storage drives for media (optional but recommended)
- Domain name pointing to your server IP (optional)
- Basic knowledge of Docker and command line

## 🔧 Initial System Setup

### 1. Update System and Install Dependencies

```
# Update system packages
sudo apt update && sudo apt upgrade -y

# Install essential packages
sudo apt install -y curl wget git htop nano mergerfs

# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Install Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

# Log out and back in for Docker group to take effect
```

### 2. Clone Repository

```
git clone https://github.com/Shrishesha4/HL-origin
cd HL-origin
```

### 3. Storage Setup

#### Configure Storage Drives

```
# Identify your storage drives
sudo lsblk
sudo blkid

# Format drives (replace with your actual device names)
sudo mkfs.ext4 /dev/sdb1  # First drive
sudo mkfs.ext4 /dev/sdc1  # Second drive

# Create mount points
sudo mkdir -p /mnt/sdb1 /mnt/sdc1
```

#### Setup MergerFS Pool

Add the following to `/etc/fstab` (replace UUIDs with your actual ones):

```
# Get UUIDs for your drives
sudo blkid

# Edit fstab
sudo nano /etc/fstab

# Add these lines (replace UUIDs):
UUID=your-first-drive-uuid /mnt/sdb1 ext4 defaults 0 2
UUID=your-second-drive-uuid /mnt/sdc1 ext4 defaults 0 2
/mnt/sdb1:/mnt/sdc1:/home /pool fuse.mergerfs defaults,nonempty,allow_other,use_ino,cache.files=off,moveonenospc=true,category.create=mfs,dropcacheonclose=true,minfreespace=50G 0 0

# Mount all
sudo mount -a

# Verify pool is working
df -h | grep pool
ls -la /pool
```

### 4. Directory Structure Setup

```
# Create necessary directories
sudo mkdir -p /pool/{media,downloads,configs}
sudo mkdir -p /pool/media/{movies,tv,music}
sudo mkdir -p /pool/downloads/{complete,incomplete}

# Set permissions
sudo chown -R $USER:$USER /pool
sudo chmod -R 755 /pool
```

## 🐳 Setup Methods

Choose one of the following deployment methods:

---

## Method 1: Docker Compose (Recommended for CLI users)

### 1. Environment Configuration

Create `.env` file in the root directory:

```
# Copy example file if it exists, otherwise create manually
nano .env
```

Add your configuration:

```
# Timezone
TZ=Asia/Kolkata

# User/Group IDs
PUID=1000
PGID=1000

# Domain configuration (optional)
DOMAIN=yourdomain.com

# Paths
MEDIA_ROOT=/pool/media
DOWNLOAD_ROOT=/pool/downloads
CONFIG_ROOT=/pool/configs

# Database passwords (generate secure ones)
MYSQL_ROOT_PASSWORD=your-secure-password
POSTGRES_PASSWORD=your-secure-password
```

### 2. Start Services

```
# Start networking services first
cd docker-compose/network
docker-compose up -d

# Start media server stack
cd ../media-server
docker-compose up -d

# Start monitoring tools
cd ../tools
docker-compose up -d
```

---

## Method 2: Portainer (Web-based GUI)

### 1. Install Portainer

```
# Create Portainer volume
docker volume create portainer_data

# Start Portainer
docker run -d \
  -p 8000:8000 \
  -p 9443:9443 \
  --name portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

### 2. Access Portainer

1. Open browser and navigate to `https://your-server-ip:9443`
2. Create admin user on first visit
3. Connect to local Docker environment

### 3. Deploy Stacks via Portainer

#### Option A: Git Repository Deployment

1. In Portainer, go to **Stacks** → **Add Stack**
2. Choose **Repository** method
3. Enter repository URL: `https://github.com/Shrishesha4/HL-origin`
4. Set **Compose file path**: `docker-compose/media-server/docker-compose.yml`
5. Add environment variables in the **Environment** section:
   ```
   TZ=Asia/Kolkata
   PUID=1000
   PGID=1000
   MEDIA_ROOT=/pool/media
   DOWNLOAD_ROOT=/pool/downloads
   CONFIG_ROOT=/pool/configs
   ```
6. Click **Deploy the stack**

#### Option B: Upload Compose Files

1. Go to **Stacks** → **Add Stack**
2. Choose **Upload** method
3. Upload your `docker-compose.yml` files
4. Set environment variables as above
5. Deploy each stack (network, media-server, tools)

#### Option C: Web Editor Method

1. Go to **Stacks** → **Add Stack**
2. Choose **Web editor**
3. Copy and paste your docker-compose content
4. Configure environment variables
5. Deploy the stack

### 4. Portainer Stack Deployment Order

Deploy in this sequence to avoid dependency issues:

1. **Network Stack**: Deploy `docker-compose/network/docker-compose.yml`
2. **Media Server Stack**: Deploy `docker-compose/media-server/docker-compose.yml` 
3. **Tools Stack**: Deploy `docker-compose/tools/docker-compose.yml`

### 5. Managing Stacks in Portainer

- **View logs**: Stack → Select service → Logs
- **Restart services**: Stack → Select service → Restart
- **Update images**: Stack → Editor → Re-deploy
- **Scale services**: Stack → Select service → Scale/Edit

---

## 📊 Service Configuration

### SSL Certificate Setup

```
# If using custom certificates:
sudo mkdir -p configs/data/custom_ssl/
sudo cp /path/to/your/cert.pem configs/data/custom_ssl/
sudo cp /path/to/your/key.pem configs/data/custom_ssl/
```

### Configure Individual Services

#### Nginx Proxy Manager (NPM)
1. Access NPM at `http://your-server-ip:81`
2. Login with default credentials:
   - Email: `admin@example.com`
   - Password: `changeme`
3. Change default password immediately
4. Configure SSL certificates and proxy hosts

#### Homepage Dashboard
1. Edit `configs/homepage/services.yaml` to add your services
2. Update `configs/homepage/widgets.yaml` for monitoring widgets
3. Access at configured URL through NPM

#### Media Server (*arr stack)
1. Configure each service through their web interfaces
2. Set up indexers and download clients
3. Configure quality profiles and root folders

## 📊 Monitoring Setup

### Glances Configuration

Create `configs/glances/glances.conf`:

```
[fs]
# Allow MergerFS filesystem
allow=mergerfs
# Hide unwanted mounts
hide=/snap.*,/dev/loop.*,/run/.*
```

## 🔒 Security Recommendations

### Firewall Setup

```
# Install UFW
sudo apt install ufw

# Configure basic rules
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow ssh
sudo ufw allow 80
sudo ufw allow 443
sudo ufw allow 81   # NPM admin
sudo ufw allow 9443 # Portainer HTTPS
sudo ufw allow 8000 # Portainer tunnel (optional)

# Enable firewall
sudo ufw enable
```

### Secure Portainer

1. Change default admin password
2. Enable HTTPS-only access
3. Consider restricting access to local network only:
   ```
   sudo ufw allow from 192.168.0.0/16 to any port 9443
   ```

## 🔄 Maintenance

### Regular Updates

#### Via Docker Compose
```
# Update system packages
sudo apt update && sudo apt upgrade

# Update Docker images
cd HL-origin/docker-compose/[stack-name]
docker-compose pull
docker-compose up -d

# Clean up unused Docker resources
docker system prune -af
```

#### Via Portainer
1. Go to **Stacks** → Select your stack
2. Click **Editor**
3. Click **Update the stack** (pulls latest images)
4. Or use **Images** section to pull updates manually

### Backup Strategy

```
# Backup configurations
tar -czf backup-configs-$(date +%Y%m%d).tar.gz configs/

# Backup Portainer data
docker run --rm -v portainer_data:/data -v $(pwd):/backup alpine tar czf /backup/portainer-backup-$(date +%Y%m%d).tar.gz -C /data .
```

## 🛠️ Troubleshooting

### Common Issues

1. **Permissions Issues**: Ensure PUID/PGID match your user (check with `id`)
2. **Storage Not Visible**: Check fstab entries and MergerFS mount
3. **SSL Issues**: Verify certificate paths and domain configuration
4. **Service Not Starting**: Check logs in Portainer or via `docker-compose logs servicename`
5. **Portainer Access Issues**: Ensure firewall allows port 9443

### Useful Commands

```
# Check service status
docker ps
docker-compose ps

# View logs
docker logs [container-name]
docker-compose logs -f [service-name]

# Restart specific service
docker restart [container-name]
docker-compose restart [service-name]

# Check system resources
htop
df -h

# Check Portainer container
docker logs portainer
```

### Service Access URLs

After deployment, access services at:

- **Portainer**: `https://server-ip:9443`
- **NPM**: `http://server-ip:81`
- **Homepage**: Configure through NPM with your domain
- **Glances**: Configure through NPM or access directly
- **Media Services**: Configure through NPM with subdomains

## 📞 Support

For issues and questions:
- Check Docker/Portainer logs first
- Verify configuration files and environment variables  
- Ensure all prerequisites are met
- Review firewall and network settings
- Check storage mount points and permissions
