# Firewall

A properly configured firewall is essential for server security. After your web server is running, you should restrict inbound traffic to only the ports required by your applications—typically port 80 (HTTP) and port 443 (HTTPS). Port 20 is usually used for FTP data transfers and is not needed unless you specifically run an FTP server.

Allow all outbound traffic unless you have special requirements to restrict outgoing connections.

If you are using Cloudflare as a reverse proxy, you can further enhance security by only allowing inbound traffic to these ports from Cloudflare's IP ranges (CIDRs). This ensures that users can only access your web applications through Cloudflare, preventing attackers from bypassing Cloudflare's protection and directly reaching your server.

```bash
# Enable UFW and default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# sudo ufw allow 22/tcp
sudo ufw allow 22/tcp

# Allow HTTP/HTTPS only from Cloudflare
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do sudo ufw allow from $ip to any port 80,443 proto tcp; done # IPv4
for ip in $(curl -s https://www.cloudflare.com/ips-v6); do sudo ufw allow from $ip to any port 80,443 proto tcp; done # IPv6

# Enable UFW
sudo ufw enable

# Check Status
sudo ufw status verbose
```

The final output should be looking like that:

```bash
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), deny (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
80,443/tcp                 ALLOW IN    173.245.48.0/20
80,443/tcp                 ALLOW IN    103.21.244.0/22
80,443/tcp                 ALLOW IN    103.22.200.0/22
80,443/tcp                 ALLOW IN    103.31.4.0/22
80,443/tcp                 ALLOW IN    141.101.64.0/18
80,443/tcp                 ALLOW IN    108.162.192.0/18
80,443/tcp                 ALLOW IN    190.93.240.0/20
80,443/tcp                 ALLOW IN    188.114.96.0/20
80,443/tcp                 ALLOW IN    197.234.240.0/22
80,443/tcp                 ALLOW IN    198.41.128.0/17
80,443/tcp                 ALLOW IN    162.158.0.0/15
80,443/tcp                 ALLOW IN    104.16.0.0/13
80,443/tcp                 ALLOW IN    104.24.0.0/14
80,443/tcp                 ALLOW IN    172.64.0.0/13
80,443/tcp                 ALLOW IN    131.0.72.0/22
22/tcp (v6)                ALLOW IN    Anywhere (v6)
80,443/tcp                 ALLOW IN    2400:cb00::/32
80,443/tcp                 ALLOW IN    2606:4700::/32
80,443/tcp                 ALLOW IN    2803:f800::/32
80,443/tcp                 ALLOW IN    2405:b500::/32
80,443/tcp                 ALLOW IN    2405:8100::/32
80,443/tcp                 ALLOW IN    2a06:98c0::/29
80,443/tcp                 ALLOW IN    2c0f:f248::/32
```

Cloudflare updates these CIDR ranges several times a year. Instead of manually checking and updating them, you can automate the process by creating a cron job to update the rules regularly:

```bash
# Create the script in /usr/local/bin (a common system-wide location for custom executables)
# This path is chosen because /usr/local/bin is intended for user-supplied scripts and binaries that are not managed by the system's package manager, making it an ideal place for utilities like this that should be accessible to all users and available to cron without issues.
sudo nano /usr/local/bin/update-cloudflare-ufw.sh

#### File Content Start ####
#!/bin/bash
set -e

add_rule_if_missing() {
  local ip=$1
  local port="80,443"
  local proto="tcp"
  local comment="Cloudflare"

  # Check if the exact rule already exists
  if ! sudo ufw status verbose | grep -qP "ALLOW IN.*From\s+$ip.*Ports\s+$port.*Proto\s+$proto"; then
    sudo ufw allow from "$ip" to any port $port proto $proto comment "$comment"
  fi
}

echo "Syncing Cloudflare IP rules to UFW..."

# IPv4
for ip in $(curl -s https://www.cloudflare.com/ips-v4); do
  add_rule_if_missing "$ip"
done

# IPv6
for ip in $(curl -s https://www.cloudflare.com/ips-v6); do
  add_rule_if_missing "$ip"
done

echo "Done syncing Cloudflare IP rules at $(date)"
#### File Content End ####

# Make it executable
sudo chmod +x /usr/local/bin/update-cloudflare-ufw.sh

# Run it once manually
sudo /usr/local/bin/update-cloudflare-ufw.sh

# Then check
sudo ufw status

# Open crontab
sudo crontab -e

# Add the cronjob that runs daily at 3 AM
0 3 * * * /usr/local/bin/update-cloudflare-ufw.sh >/dev/null 2>&1
```