# SearXNG Docker

A production-ready, self-hosted [SearXNG](https://docs.searxng.org/) deployment using Docker Compose. Provides a private, API-enabled metasearch engine with no tracking and no ads.

## What is SearXNG?

SearXNG is a free, privacy-respecting metasearch engine that aggregates results from 70+ search engines (Google, Bing, DuckDuckGo, etc.) without tracking users. This deployment exposes a **JSON API** suitable for use as a search tool in AI agents, RAG pipelines, or any application that needs web search.

## Architecture

```
┌─────────────────────────────────────┐
│  SearXNG (Port 8182)                │
│  - Web UI for manual search         │
│  - JSON API for programmatic use    │
│  - Aggregates 70+ search engines    │
└────────────┬────────────────────────┘
             │
             ▼
┌─────────────────────────────────────┐
│  Valkey/Redis (Internal)            │
│  - Rate limiting cache              │
│  - Result caching                   │
│  - Auto-saves every 30 seconds      │
└─────────────────────────────────────┘
```

## Prerequisites

- **Docker** (v20.10+)
- **Docker Compose** (v2.0+)
- A Linux server (tested on Ubuntu 22.04)

Verify installation:

```bash
docker --version          # Docker version 20.10+
docker compose version    # Docker Compose version v2.0+
```

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/bhargav-latent/searxng-docker.git
cd searxng-docker
```

### 2. Generate a Secret Key

Replace the default secret key in the settings file:

```bash
sed -i "s|CHANGE_ME_GENERATE_WITH_openssl_rand_-hex_32|$(openssl rand -hex 32)|g" searxng/settings.yml
```

### 3. Configure the Base URL

Edit `docker-compose.yml` and set `SEARXNG_BASE_URL` to your server's IP or domain:

```yaml
environment:
  - SEARXNG_BASE_URL=http://<YOUR_SERVER_IP>:8182/
```

### 4. Start the Services

```bash
docker compose up -d
```

### 5. Verify

```bash
# Check containers are running
docker compose ps

# Test the health endpoint
curl http://localhost:8182/healthz

# Test a search query
curl "http://localhost:8182/search?q=hello+world&format=json"
```

You should see JSON search results. The web UI is also available at `http://<YOUR_SERVER_IP>:8182`.

## Configuration

### docker-compose.yml

| Setting | Default | Description |
|---------|---------|-------------|
| `ports` | `8182:8080` | Host port mapped to SearXNG. Change `8182` to use a different port |
| `SEARXNG_BASE_URL` | `http://10.26.1.129:8182/` | Public URL of your instance. Must match your server IP/domain |
| `UWSGI_WORKERS` | `4` | Number of uWSGI worker processes |
| `UWSGI_THREADS` | `4` | Number of threads per worker |

### searxng/settings.yml

```yaml
use_default_settings: true     # Inherit all default SearXNG settings
server:
  secret_key: "<random-hex>"   # REQUIRED: unique key for your instance
  limiter: false               # Rate limiting (set true for public instances)
  image_proxy: true            # Proxy images through SearXNG for privacy
search:
  formats:
    - json                     # Enable JSON API (required for programmatic use)
    - html                     # Enable web UI
```

#### Key Settings to Consider

| Setting | Default | When to Change |
|---------|---------|----------------|
| `secret_key` | pre-generated | **Always** — generate a unique key per deployment |
| `limiter` | `false` | Set to `true` if exposing to the public internet |
| `image_proxy` | `true` | Set to `false` to reduce bandwidth if you don't need image proxying |
| `formats: json` | enabled | **Do not remove** — required for API access |

## API Usage

### Search Endpoint

```
GET /search?q=<query>&format=json
```

#### Parameters

| Parameter | Required | Description |
|-----------|----------|-------------|
| `q` | Yes | Search query string |
| `format` | Yes | Response format: `json` or `html` |
| `categories` | No | Comma-separated: `general`, `images`, `videos`, `news`, `science`, `files`, `it`, `social media` |
| `engines` | No | Comma-separated engine names: `google`, `duckduckgo`, `wikipedia`, etc. |
| `language` | No | Language code: `en`, `fr`, `de`, etc. |
| `pageno` | No | Page number (default: `1`) |

#### Example Requests

**Basic search:**
```bash
curl "http://localhost:8182/search?q=docker+compose+tutorial&format=json"
```

**Search specific category:**
```bash
curl "http://localhost:8182/search?q=sunset&categories=images&format=json"
```

**Search with specific engines:**
```bash
curl "http://localhost:8182/search?q=python+tutorial&engines=google,duckduckgo&format=json"
```

**News search:**
```bash
curl "http://localhost:8182/search?q=AI+news&categories=news&format=json"
```

#### Response Structure

```json
{
  "query": "hello world",
  "results": [
    {
      "title": "Result Title",
      "url": "https://example.com",
      "content": "Description/snippet of the result...",
      "engine": "duckduckgo",
      "score": 1.0,
      "category": "general",
      "publishedDate": null
    }
  ],
  "suggestions": ["related query 1", "related query 2"],
  "unresponsive_engines": []
}
```

### Health Check

```bash
curl http://localhost:8182/healthz
# Returns HTTP 200 if healthy
```

### Using with Python

```python
import requests

def search(query, categories="general"):
    response = requests.get(
        "http://localhost:8182/search",
        params={
            "q": query,
            "format": "json",
            "categories": categories,
        },
    )
    response.raise_for_status()
    return response.json()["results"]

results = search("what is Docker?")
for r in results:
    print(f"{r['title']}: {r['url']}")
```

### Using as an AI Agent Tool (LangChain)

```python
from langchain_core.tools import tool
import requests

@tool
def internet_search(query: str) -> str:
    """Search the internet for current information."""
    response = requests.get(
        "http://localhost:8182/search",
        params={"q": query, "format": "json"},
        timeout=10,
    )
    response.raise_for_status()
    results = response.json().get("results", [])[:5]
    return "\n\n".join(
        f"**{r['title']}**\n{r['url']}\n{r['content']}"
        for r in results
    )
```

## Deployment Guide

### Production Deployment on a VM

#### Step 1: Install Docker

```bash
# Ubuntu/Debian
sudo apt update
sudo apt install -y docker.io docker-compose-v2
sudo systemctl enable --now docker

# Add your user to the docker group (log out and back in after)
sudo usermod -aG docker $USER
```

#### Step 2: Clone and Configure

```bash
git clone https://github.com/bhargav-latent/searxng-docker.git
cd searxng-docker

# Generate a unique secret key
sed -i "s|CHANGE_ME_GENERATE_WITH_openssl_rand_-hex_32|$(openssl rand -hex 32)|g" searxng/settings.yml

# Set your server's IP/domain
SERVER_IP=$(hostname -I | awk '{print $1}')
sed -i "s|SEARXNG_BASE_URL=.*|SEARXNG_BASE_URL=http://${SERVER_IP}:8182/|" docker-compose.yml
```

#### Step 3: Start Services

```bash
docker compose up -d
```

#### Step 4: Verify

```bash
# Check both containers are running
docker compose ps

# Expected output:
# NAME       IMAGE                            STATUS
# redis      docker.io/valkey/valkey:8-alpine  Up
# searxng    docker.io/searxng/searxng:latest  Up

# Test the API
curl "http://localhost:8182/search?q=test&format=json" | python3 -m json.tool | head -20
```

### Firewall Configuration

If your server has a firewall, open port 8182:

```bash
# UFW (Ubuntu)
sudo ufw allow 8182/tcp

# firewalld (CentOS/RHEL)
sudo firewall-cmd --permanent --add-port=8182/tcp
sudo firewall-cmd --reload

# GCP
gcloud compute firewall-rules create allow-searxng \
  --allow tcp:8182 \
  --source-ranges 0.0.0.0/0 \
  --description "Allow SearXNG"
```

### Using a Custom Port

Edit `docker-compose.yml`:

```yaml
ports:
  - "YOUR_PORT:8080"   # Change left side only
```

Update the base URL to match:

```yaml
environment:
  - SEARXNG_BASE_URL=http://YOUR_IP:YOUR_PORT/
```

Then restart:

```bash
docker compose down && docker compose up -d
```

### Enabling Rate Limiting (Public Instances)

If exposing SearXNG to the public internet, enable rate limiting:

```yaml
# searxng/settings.yml
server:
  limiter: true
```

Then restart:

```bash
docker compose restart searxng
```

### Putting Behind a Reverse Proxy (Nginx)

For HTTPS and domain-based access:

```nginx
server {
    listen 443 ssl;
    server_name search.yourdomain.com;

    ssl_certificate     /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:8182;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Update `docker-compose.yml`:

```yaml
environment:
  - SEARXNG_BASE_URL=https://search.yourdomain.com/
```

## Operations

### View Logs

```bash
# All services
docker compose logs -f

# SearXNG only
docker compose logs -f searxng

# Redis only
docker compose logs -f redis
```

### Restart Services

```bash
docker compose restart
```

### Update to Latest SearXNG

```bash
docker compose pull
docker compose down
docker compose up -d
```

### Stop Services

```bash
docker compose down
```

### Full Reset (Delete All Data)

```bash
docker compose down -v   # -v removes volumes (cached data)
docker compose up -d
```

## Troubleshooting

### Container not starting

```bash
# Check logs for errors
docker compose logs searxng

# Common fix: permission issue on settings
chmod 644 searxng/settings.yml
docker compose restart searxng
```

### Port already in use

```bash
# Find what's using the port
sudo lsof -i :8182

# Kill the process or change the port in docker-compose.yml
```

### JSON API returns HTML

Ensure `json` is listed in `searxng/settings.yml`:

```yaml
search:
  formats:
    - json
    - html
```

### No search results

```bash
# Check if engines are responding
curl -s "http://localhost:8182/search?q=test&format=json" | python3 -c "
import sys, json
d = json.load(sys.stdin)
print(f'Results: {len(d.get(\"results\", []))}')
print(f'Unresponsive engines: {d.get(\"unresponsive_engines\", [])}')
"
```

If all engines are unresponsive, check DNS resolution from inside the container:

```bash
docker exec searxng nslookup google.com
```

### High memory usage

Reduce workers and threads in `docker-compose.yml`:

```yaml
environment:
  - UWSGI_WORKERS=2
  - UWSGI_THREADS=2
```

## Security Notes

- **Secret key**: Always generate a unique secret key per deployment. Never use the default.
- **Rate limiting**: Enable `limiter: true` if the instance is publicly accessible.
- **Network**: Both containers run in an isolated Docker network. Only SearXNG's port is exposed to the host.
- **Capabilities**: Containers drop all Linux capabilities except the minimum required (CHOWN, SETGID, SETUID).
- **Logging**: Log rotation is configured (1MB max, 1 file) to prevent disk exhaustion.

## License

SearXNG is licensed under [AGPL-3.0](https://github.com/searxng/searxng/blob/master/LICENSE). This deployment configuration is for internal use.
