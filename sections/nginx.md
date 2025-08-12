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

### Basic Authentication

```bash
# install apache2-utils (to get htpasswd tool)
sudo apt install apache2-utils

# username, it'll ask you a password; enter it twice
sudo htpasswd -c /etc/nginx/.htpasswd <username>
```

```nginx
server {
    listen 80;
    server_name admin.example.com;

    location / {
        auth_basic "Restricted Access";
        auth_basic_user_file /etc/nginx/.htpasswd;

        proxy_pass http://localhost:3000;  # Or your backend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Check for a custom header value

```nginx
server {
    listen 80;
    server_name custom-header.example.com;

    location / {
        # X-Custom-Header: ExpectedValue
        if ($http_x_custom_header != "ExpectedValue") {
            return 403;
        }
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```