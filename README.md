# Server Setup

```bash
# Update the package index and upgrade all installed packages
sudo apt update && sudo apt upgrade -y

# Create a new user (replace <username> with your desired username)
adduser <username>

# Add the new user to the sudo group to grant admin privileges
usermod -aG sudo <username>

# Switch to the new user account
su - <username>

# Set up SSH configuration for the new user
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys   # Paste your public key here for key-based authentication
chmod 600 ~/.ssh/authorized_keys

# Disable root login over SSH for enhanced security
sudo nano /etc/ssh/sshd_config   # Change 'PermitRootLogin yes' to 'PermitRootLogin no'
sudo systemctl restart ssh       # Restart SSH to apply changes

# Install btop for system monitoring
sudo apt install btop -y

# (Optional) Update the system's timezone
timedatectl list-timezones | grep Istanbul # List all available timezones and filter for "Istanbul" to find the correct timezone string
sudo timedatectl set-timezone Europe/Istanbul # Set the system timezone to Europe/Istanbul
timedatectl # Display the current date, time, and timezone settings to verify the change

# Set up Zsh and install essentials
sudo apt install -y zsh git curl
chsh -s $(which zsh)  # Change the user's default shell to Zsh. Logout and log back in (exit and reconnect by SSH) to use Zsh.

# Install Oh My Zsh (improves Zsh experience with themes and plugins)
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# Install Zsh plugins for autosuggestions and syntax highlighting
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
nano ~/.zshrc   # Edit the plugins line: plugins=(git) → plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
source ~/.zshrc # Reload the Zsh configuration
```

# GitHub Setup

```bash
# Generate a new SSH key pair (replace the email with yours)
ssh-keygen -t ed25519 -C "your_email@example.com"   # Press Enter through prompts

# Copy the content of your public key to clipboard
cat ~/.ssh/id_ed25519.pub
```

1. Log in to your GitHub account.
2. Go to 'Settings'.
3. Navigate to `SSH and GPG keys`.
4. Click `New SSH Key`.
5. Give it a descriptive name and paste the content you just copied.
6. Test your SSH connection with the following command:

```bash
ssh -T git@github.com
# Hi BooRuleDie! You've successfully authenticated, but GitHub does not provide shell access.
```

# Docker

The Docker installation process can change over time. For the most up-to-date instructions, refer to the [official Docker documentation](https://docs.docker.com/engine/install/ubuntu/).

```bash
# Remove any potentially conflicting Docker-related packages
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add Docker's repository to APT sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# Install Docker Engine and related packages
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# Test your Docker installation with the hello-world container
sudo docker run hello-world

# Add your user to the docker group for non-root usage (replace <username> accordingly)
usermod -aG docker <username>
```

# Lazydocker

**Always inspect the content of scripts downloaded with pipes before executing for security.**

```bash
# Download Lazydocker installation script and execute it
curl https://raw.githubusercontent.com/jesseduffield/lazydocker/master/scripts/install_update_linux.sh | bash

# Move the binary to a directory in your PATH
sudo cp ~/.local/bin/lazydocker /usr/local/bin
```

# Node & npm & pm2

For the most current Node.js version, use [the official Node.js website](https://nodejs.org/en/download/current).
**Avoid installing Node.js via `apt install node` as it is usually outdated.**

```bash
# Download and install nvm (Node Version Manager) to easily manage Node.js versions
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# Load nvm without restarting the shell (for current session)
\. "$HOME/.nvm/nvm.sh"

# Download and install the latest Node.js (replace 24 with your desired version if preferred)
nvm install 24

# Verify Node.js installation
node -v         # Should print something like "v24.4.1"
nvm current     # Should also reflect the installed version

# Check npm (Node Package Manager) version
npm -v          # Should print a current version, e.g. "11.4.2"

# Install pm2 globally (process manager for Node apps)
npm install -g pm2
```

# Cloudflare SSL

To create a Cloudflare Origin Certificate:

- Choose your domain and navigate to `SSL/TLS > Origin Server` in Cloudflare's dashboard.
- Click the blue **Create Certificate** button, leave the default options, and save your **origin certificate** and **private key**.
- You will need these certificate files when setting up your load balancer or reverse proxy server.

For subdomains, you can create `A` records pointing them to any **IPv4** address at your server.

# NGINX

```bash
# Install nginx web server
sudo apt install -y nginx
```

By default, NGINX includes all files under its `sites-enabled` and `conf.d` directories in its main configuration.
All you need to do is create the appropriate config files and reload NGINX as shown below.

**sites-enabled/unleash.booruledie.com**

```nginx
server {
    listen 80;
    server_name unleash.booruledie.com;

    # Redirect HTTP traffic to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name unleash.booruledie.com;

    ssl_certificate     /etc/ssl/certs/unleash-origin.pem;
    ssl_certificate_key /etc/ssl/private/unleash-origin.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    location / {
        proxy_pass http://127.0.0.1:4242; # Unleash dashboard running locally
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**sites-enabled/booruledie.com**

```nginx
server {
    listen 80;
    server_name booruledie.com;

    # Redirect all HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name booruledie.com;

    ssl_certificate     /etc/nginx/ssl/booruledie.com.cert;
    ssl_certificate_key /etc/nginx/ssl/booruledie.com.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    root /home/booruledie/Frontend/dist;
    index index.html;

    # Proxy API requests to the production backend
    location /api/ {
        proxy_pass http://127.0.0.1:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # For React SPAs: always serve index.html for non-static routes
    location / {
        try_files $uri /index.html;
    }

}
```

If you are serving static files from users' home directories, be sure to adjust directory permissions appropriately.
NGINX requires at least **read** and **execute** permissions for all directories in the file path to serve files correctly.

```bash
# Grant NGINX read and execute permissions for the directories and files:
sudo chmod o+rx /home /home/booruledie /home/booruledie/react /home/booruledie/react/build
sudo chmod o+r /home/booruledie/react/build/index.html
```

If you want to avoid adjusting directory permissions, you can place your files in a directory that NGINX already has access to by default, such as `/var/www/html`. Any files you put in this directory can be served by NGINX without extra permission changes:

```bash
# Example: Move your site files to NGINX's default web root
sudo mkdir -p /var/www/html
sudo cp -r <your_site_files>/* /var/www/html/
```

```bash
# Test the NGINX configuration syntax
sudo nginx -t

# Reload NGINX to apply configuration changes
sudo nginx -s reload
```
