# Cloudflare SSL

To create a Cloudflare Origin Certificate:

- Choose your domain and navigate to `SSL/TLS > Origin Server` in Cloudflare's dashboard.
- Click the blue **Create Certificate** button, leave the default options, and save your **origin certificate** and **private key**.
- You will need these certificate files when setting up your load balancer or reverse proxy server.

For subdomains, you can create `A` records pointing them to any **IPv4** address at your server.