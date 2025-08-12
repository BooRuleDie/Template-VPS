# OpenWebUI + LiteLLM Setup

**OpenWebUI + LiteLLM** = self-hosted AI chat interface + unified backend API for any LLM provider or local model. OpenWebUI expects an **OpenAI-compatible API** to work; LiteLLM provides exactly that while routing requests to any supported model.

**Features:**

- **Multi-user & sub-user management** – control access for teams or individuals.
- **Conversation history & prompt library** – save chats and reusable prompts.
- **Model switching** – easily swap between cloud and local models without reconfiguring the UI.
- **Centralized API key control** – manage provider keys in one place.
- **Request & usage logs** – track all queries for auditing.
- **Cost tracking & rate limiting** – monitor and control expenses.
- **Failover between models** – automatically switch if a model is down.

### OpenWebUI

![OpenWebUI](../images/openwebui.png)

### LiteLLM

![LiteLLM](../images/litellm.png)

First setup the .env file in the same directory just like that:

```bash
# OpenWebUI
MASTER_KEY=<MASTER_KEY>
OPENAI_API_BASE_URL=https://api.openai.com/v1

# LiteLLM
LITELLM_MASTER_KEY=<LITELLM_MASTER_KEY>
ANTHROPIC_API_KEY=<API_KEY>
OPENAI_API_KEY=<API_KEY>
```

Then you can use this docker compose file to start both applications:

```yml
services:
    openwebui:
        image: ghcr.io/open-webui/open-webui:main
        container_name: open-webui
        ports:
            - "127.0.0.1:3000:8080"
        volumes:
            - open-webui:/app/backend/data
        env_file:
            - .env
        restart: always
        depends_on:
            - litellm
        networks:
            - app-network

    litellm:
        image: ghcr.io/berriai/litellm-database:main-latest
        container_name: litellm
        ports:
            - "127.0.0.1:4000:4000"
        env_file:
            - .env
        restart: always
        environment:
            DATABASE_URL: "postgresql://llmproxy:dbpassword9090@litellm_db:5432/litellm"
            STORE_MODEL_IN_DB: "True"
        depends_on:
            litellm_db:
                condition: service_healthy
        networks:
            - app-network

    litellm_db:
        image: postgres:17.4
        container_name: postgres-litellm
        restart: always
        environment:
            POSTGRES_DB: litellm
            POSTGRES_USER: llmproxy
            POSTGRES_PASSWORD: dbpassword9090
        volumes:
            - litellm-postgres:/var/lib/postgresql/data
        healthcheck:
            test: ["CMD-SHELL", "pg_isready -d litellm -U llmproxy"]
            interval: 1s
            timeout: 5s
            retries: 10
        networks:
            - app-network

volumes:
    open-webui:
    litellm-postgres:

networks:
    app-network:
        driver: bridge
```

### LiteLLM Setup

1. Open the LiteLLM web interface and expand the sidebar menu.
2. Click on **Models & Endpoints**, then hit **Add Model**.
3. In the “Add Model” form:
    - **Model Name**: A friendly label (e.g., `gpt-4`).
    - **Provider**: Select your provider (OpenAI, Anthropic, or Local).
    - **Endpoint URL**: The URL where LiteLLM should forward requests.
    - (Optional) Set any extra parameters like default temperature or token limits.
4. Click **Save** to register the new model.
5. Now navigate to the **Virtual Keys** section in the sidebar.
6. Click **Create API Key**, enter a descriptive name, and check the model(s) you just configured.

### OpenWebUI Config

Next, configure OpenWebUI to connect to LiteLLM:

1. Open OpenWebUI in your browser and click the profile icon in the bottom-left corner.
2. Select **Settings** → **Admin Settings** → **Connections**, then click **Add Connection**.
3. In the dialog, enter:
    - **Name**: LiteLLM
    - **Base URL**: http://litellm:4000
      (use the Docker service name so OpenWebUI can resolve the LiteLLM container)
    - **API Key**: paste the **API Key** you generated above
4. Click **Save** to register the connection.

You should now see LiteLLM listed under **Connections**, ready to serve requests.

![Connection Setup](../images/openwebui-connection.png)

### Nginx Setup

**/etc/nginx/sites-enabled/litellm.yourdomain.com**
```nginx
server {
    listen 80;
    server_name litellm.yourdomain.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name litellm.yourdomain.com;

    # Restrict access to authorized users only.
    # Remove or comment out this block if you want to allow open access without requiring a custom header.
    if ($http_x_custom_secret != "some-custom-value-here") {
        return 403 "Access Denied";
    }

    ssl_certificate     /etc/nginx/ssl/yourdomain.com.cert;
    ssl_certificate_key /etc/nginx/ssl/yourdomain.com.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    # redirect requests to the OpenWebUI docker container
    location / {
        proxy_pass http://127.0.0.1:4000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```

**/etc/nginx/sites-enabled/openwebui.yourdomain.com**
```nginx
server {
    listen 80;
    server_name openwebui.yourdomain.com;

    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name openwebui.yourdomain.com;

    # Restrict access to authorized users only.
    # Remove or comment out this block if you want to allow open access without requiring a custom header.
    if ($http_x_custom_secret != "some-custom-value-here") {
        return 403 "Access Denied";
    }

    ssl_certificate     /etc/nginx/ssl/yourdomain.com.cert;
    ssl_certificate_key /etc/nginx/ssl/yourdomain.com.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers on;

    # redirect requests to the OpenWebUI docker container
    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }
}
```
