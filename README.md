
# Architecture

For a solid setup, use these containers:

* `evolution-api`
* `postgres`
* `redis`

Evolution’s docs list Postgres/MySQL and Redis as required dependencies for Docker installs. ([doc.evolution-api.com][1])

## Folder structure

For example:

```bash
mkdir evolution-api
cd evolution-api
```

Then:

* Then clone: git clone https://github.com/ChristND242/Evolution-api-docker_compose.git
* cd Evolution-api-docker_compose
* create `.env`



## docker-compose.yml

Use the given **docker-compose.yml** as a starter. I added Postgres and Redis because the docs say they are prerequisites.


## .env

You may use this:

```env
SERVER_TYPE=http
SERVER_PORT=8080
SERVER_URL=https://your-evolution-domain.com

AUTHENTICATION_API_KEY=yourAPI_String

LOG_LEVEL=ERROR,WARN,INFO,LOG,DEBUG,WEBHOOKS
LOG_COLOR=true
LOG_BAILEYS=error

DATABASE_ENABLED=true
DATABASE_PROVIDER=postgresql
DATABASE_CONNECTION_URI=postgresql://evolution:strong_postgres_password@postgres:5432/evolution?schema=public
DATABASE_CONNECTION_CLIENT_NAME=evolution_local

DATABASE_SAVE_DATA_INSTANCE=true
DATABASE_SAVE_DATA_NEW_MESSAGE=true
DATABASE_SAVE_MESSAGE_UPDATE=true
DATABASE_SAVE_DATA_CONTACTS=true
DATABASE_SAVE_DATA_CHATS=true
DATABASE_SAVE_DATA_LABELS=true
DATABASE_SAVE_DATA_HISTORIC=true

CACHE_REDIS_ENABLED=true
CACHE_REDIS_URI=redis://redis:6379/0
CACHE_REDIS_PREFIX_KEY=evolution

WEBHOOK_GLOBAL_ENABLED=false
WEBHOOK_GLOBAL_URL=
WEBHOOK_GLOBAL_WEBHOOK_BY_EVENTS=false

#NODE_OPTIONS=--network-family-autoselection-attempt-timeout=1000
#CONFIG_SESSION_PHONE_VERSION=2.3000.1028450369
```


* `SERVER_URL` is the public URL Evolution uses for internal links and webhook-related return data. 
* `DATABASE_*` controls persistent storage and provider/connection. ([doc.evolution-api.com][2])
* global webhook vars are documented as `WEBHOOK_GLOBAL_URL`, `WEBHOOK_GLOBAL_ENABLED`, and `WEBHOOK_GLOBAL_WEBHOOK_BY_EVENTS`. 

## Start it

```bash
docker compose up -d
```

The official Docker docs use that exact flow. ([doc.evolution-api.com][1])

Check logs:

```bash
docker logs -f evolution_api
```


## Expose it publicly

If you want webhooks to work reliably, Evolution needs a public HTTPS URL. Their Nginx guide shows putting Nginx in front of port 8080 and then issuing an SSL cert with Certbot. ([doc.evolution-api.com][4])

So the rough production flow is:

* point a domain like `evo.yourdomain.com` to your server
* reverse proxy to `127.0.0.1:8080`
* enable SSL

A basic Nginx reverse proxy from the docs is:

```nginx
server {
  server_name evo.yourdomain.com;

  location / {
    proxy_pass http://127.0.0.1:8080;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_cache_bypass $http_upgrade;
  }
}
```



## How to configure webhooks

You have two choices.

### Option A: global webhook from `.env`

This sends enabled global events from all instances to one URL. Evolution documents this with:

```env
WEBHOOK_GLOBAL_URL='https://your-app.com/webhook/evolution'
WEBHOOK_GLOBAL_ENABLED=true
WEBHOOK_GLOBAL_WEBHOOK_BY_EVENTS=false
```

That’s good if you want one endpoint to receive everything. ([doc.evolution-api.com][3])

### Option B: per-instance webhook via API

This is usually better. It lets each instance have its own webhook config.

Evolution’s docs show using `/webhook/instance` with fields like `url`, `webhook_by_events`, and `events`. ([doc.evolution-api.com][3])

Take this as reference:

```bash
curl --request POST \
  --url https://your-evolution-domain.com/webhook/set/my-instance \
  --header 'Content-Type: application/json' \
  --header 'apikey: put-a-long-random-key-here' \
  --data '{
    "enabled": true,
    "url": "https://your-n8n-domain.com/webhook/evolution",
    "webhook_by_events": false,
    "webhook_base64": false,
    "events": [
      "QRCODE_UPDATED",
      "CONNECTION_UPDATE",
      "MESSAGES_UPSERT",
      "MESSAGES_UPDATE",
      "MESSAGES_DELETE",
      "SEND_MESSAGE"
    ]
  }'
```

The exact supported fields and example event list are documented on the Webhooks page. ([doc.evolution-api.com][3])


## Take Aways

### 1. Localhost won’t work for production webhooks unless you tunnel it.

### 2. Wrong `SERVER_URL`:

If this is wrong, callbacks and generated URLs can get weird. Evolution documents `SERVER_URL` as the server’s public address. ([doc.evolution-api.com][2])

### 3. Forgetting Postgres/Redis

Evolution v2 Docker expects them configured. ([doc.evolution-api.com][1])

### 4. No reverse proxy / SSL

Webhook providers and external systems behave much better over HTTPS, and Evolution’s own install guide includes Nginx + Certbot for secure exposure. ([doc.evolution-api.com][4])



[1]: https://doc.evolution-api.com/v2/en/install/docker "Docker - Evolution API Documentation"
[2]: https://doc.evolution-api.com/v2/en/env "Environment Variables - Evolution API Documentation"
[3]: https://doc.evolution-api.com/v2/en/configuration/webhooks "Webhooks - Evolution API Documentation"
[4]: https://doc.evolution-api.com/v2/en/install/nginx "Nginx and SSL - Evolution API Documentation"

ENJOY !!!
