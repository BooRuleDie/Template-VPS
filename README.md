# Server Setup

```bash
# update & upgrade system
sudo apt update && sudo apt upgrade -y

# create user
adduser <username>

# add it to sudo group
usermod -aG sudo <username>

# switch to user
su - <username>

# setup ssh config
mkdir -p ~/.ssh
chmod 700 ~/.ssh
nano ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys

# disable root login
sudo nano /etc/ssh/sshd_config # PermitRootLogin yes -> no
sudo systemctl restart ssh

# setup zsh
sudo apt install -y zsh git curl
chsh -s $(which zsh) # Logout and log back in (exit and then SSH again) to start using Zsh.

# install ohmyzsh
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"

# install zsh plugins (auto suggestions, syntax highlighting)
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
nano ~/.zshrc # plugins=(git) -> plugins=(git zsh-autosuggestions zsh-syntax-highlighting)
source ~/.zshrc
```

# GitHub Setup

```bash
# generate a public private key
ssh-keygen -t ed25519 -C "your_email@example.com" # just enter couple times

# copy the content of your public key
cat ~/.ssh/id_ed25519.pub
```

1. Open your GitHub account
2. Hit settings
3. Open `SSH and GPG keys`
4. Click `New SSH Key`
5. Give it a reasonable name and paste the copied content.
6. Test your connection with following command:

```bash
ssh -T git@github.com
# Hi BooRuleDie! You've successfully authenticated, but GitHub does not provide shell access.
```

# Docker

The way you install docker can be changed by time, it's always better to run the commands on the [official documentation](https://docs.docker.com/engine/install/ubuntu/).

```bash
# uninstall all conflicting packages
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done

# add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update

# install docker packages
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# test docker 
sudo docker run hello-world

# add yourself to the docker group
usermod -aG docker <username>
```

# Lazydocker

You should always check the pipe content beforehand:
```bash
# download the binary
curl https://raw.githubusercontent.com/jesseduffield/lazydocker/master/scripts/install_update_linux.sh | bash

# move it under any PATH path
sudo cp ~/.local/bin/lazydocker /usr/local/bin
```

# Node & npm & pm2

Using [this website](https://nodejs.org/en/download/current) is a better option to set it up as it'll be up to date always. Avoid using `apt install node` as it'll be an outdated version.

```bash
# download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# in lieu of restarting the shell
\. "$HOME/.nvm/nvm.sh"

# download and install Node.js:
nvm install 24

# verify the Node.js version:
node -v # should print "v24.4.1".
nvm current # should print "v24.4.1".

# verify npm version:
npm -v # Should print "11.4.2".

# install pm2 globally
npm install -g pm2
```

# Cloudflare SSL

In order to create cloudflare certificate, pick a domain and go under the `SSL/TLS > Origin server`. In that page you'll see a blue **Create Certificate** button. Hit it, leave everything as default and save your **origin certificate** and **private key**. You'll need those keys when setting up your load balancer or reverse proxy.

For subdomains you can create `A` records pointing to any **IPv4** address.

# NGINX

```bash
# install nginx
sudo apt install -y nginx
```

By default all files under `sites-enabled` and `conf.d` directories are included to the NGINX config. All you need to do is create the right config file and update the NGINX service like that:

**sites-enabled/unleash.booruledie.com**
```nginx
server {
    listen 80;
    server_name unleash.booruledie.com;

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
```
server {
    listen 80;
    server_name booruledie.com;

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

    # Proxy API requests to prod backend
    location /api/ {
        proxy_pass http://127.0.0.1:3002;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # React app: always serve index.html for SPA routes
    location / {
        try_files $uri /index.html;
    }

}
```

If you locate your static files under users' home directories make sure you give the enough permissions for each directory. NGINX expects you to give at least **read** and **execute** for each directory in the destination file & path.

```bash
# test the syntax
sudo nginx -t 

# reload nginx
sudo nginx -s reload
```

