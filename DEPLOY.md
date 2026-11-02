# Deploying the API on your own server

This repository is the REST API only (Express + MongoDB). The Vue front end is a
separate project. The steps below run the API and its database on one Linux server
with Docker Compose, behind a reverse proxy that provides HTTPS.

## 1. Server prerequisites

- A Linux server (1 vCPU / 1 GB RAM is enough to start) with Docker and the Compose plugin:
  ```console
  curl -fsSL https://get.docker.com | sh
  ```
- A DNS record for the API, e.g. `api.example.com` → the server's IP.
- Ports 80 and 443 open in the firewall. Do not open 3001 or 27017.

## 2. Get the code

```console
git clone https://github.com/ashcoyote2004-creator/youtube-clone-nodejs-api.git
cd youtube-clone-nodejs-api
```

## 3. Configure the environment

The app reads `config/.env` (it is gitignored, so it never comes with the clone):

```console
cp config/.env.example config/.env
openssl rand -hex 32   # use this output as JWT_SECRET
```

Fill in `config/.env`:

```ENV
NODE_ENV=production
PORT=3001

# Same credentials as the two MONGO_INITDB_* values below.
MONGO_URI=mongodb://mongo-user:CHANGE-ME@mongodb:27017/vue-tube?authSource=admin

MONGO_INITDB_ROOT_USERNAME=mongo-user
MONGO_INITDB_ROOT_PASSWORD=CHANGE-ME

JWT_SECRET=REPLACE-WITH-THE-GENERATED-VALUE
JWT_EXPIRE=30d
JWT_COOKIE_EXPIRE=30

FILE_UPLOAD_PATH=./public/uploads
MAX_FILE_UPLOAD=1000000

# Only needed for the password-reset email flow.
SMTP_HOST=smtp.mailtrap.io
SMTP_PORT=2525
SMTP_EMAIL=
SMTP_PASSWORD=
FROM_EMAIL=noreply@example.com
FROM_NAME=VueTube
```

Notes:

- `MONGO_INITDB_ROOT_USERNAME` and `MONGO_INITDB_ROOT_PASSWORD` are read by the
  database container on first start only. Changing them later does not change an
  existing database user.
- `JWT_SECRET` must be a real random value; auth tokens are signed with it.

## 4. Start the stack

```console
docker compose -f docker-compose.prod.yml up -d --build
docker compose -f docker-compose.prod.yml ps
curl -s http://127.0.0.1:3001/api/v1/videos/public
```

A JSON body with `"success":true` means the API and the database are up.

## 5. Seed the categories

The API has no categories until you seed them (or create them as an admin user):

```console
docker compose -f docker-compose.prod.yml exec api node seeder -i
```

`node seeder -i` imports `_data/categories.json`; `node seeder -d` deletes all
categories. Register a user first, then promote it to `admin` in the database if you
want to manage categories through the API.

## 6. HTTPS reverse proxy

With Caddy on the same server (`apt install caddy`), add `/etc/caddy/Caddyfile`:

```console
api.example.com {
	reverse_proxy 127.0.0.1:3001
}
```

```console
sudo systemctl reload caddy
curl -s https://api.example.com/api/v1/videos/public
```

Caddy obtains and renews the certificate automatically. With nginx, proxy
`https://api.example.com` to `http://127.0.0.1:3001` and use certbot for the
certificate.

## 7. Point the front end at the API

The API allows cross-origin requests (`cors()` in `app.js` is unrestricted), so the
Vue front end can call `https://api.example.com/api/v1/...` directly. Set its API base
URL accordingly and deploy the front end separately.

## 8. Data, uploads and updates

- Videos and thumbnails are written to the `uploads` volume mounted at
  `/app/public/uploads`; the database lives in the `data` volume.
- Uploaded files are served by the API at `/uploads/...`, so they have no CDN in
  front of them.
- Back up the database with:
  ```console
  docker compose -f docker-compose.prod.yml exec -T mongodb mongodump --archive > backup-$(date +%F).archive
  ```
- Deploy a new version:
  ```console
  git pull
  docker compose -f docker-compose.prod.yml up -d --build
  ```
- The API's rate limiter is commented out in `app.js`, so nothing throttles requests
  yet; enable it before exposing the API publicly at scale.
- `mongo-express` from the development `docker-compose.yml` is intentionally not part
  of this stack.
