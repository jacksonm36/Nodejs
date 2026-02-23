# Debian 13 setup: MongoDB + Node.js + Payload app

This guide installs:

- Node.js (LTS, 22.x)
- MongoDB Community Edition (8.0)
- A new Payload app

## 0) Prerequisites

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg build-essential
```

---

## 1) Install Node.js (22.x LTS)

```bash
sudo install -d -m 0755 /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | \
  sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_22.x nodistro main" | \
  sudo tee /etc/apt/sources.list.d/nodesource.list
sudo apt-get update
sudo apt-get install -y nodejs

node -v
npm -v
```

---

## 2) Install MongoDB (native package method)

As of this writing, MongoDB's Debian install doc targets Debian 12 ("bookworm").
On Debian 13, many users run MongoDB 8.0 using the bookworm repo below.

```bash
curl -fsSL https://www.mongodb.org/static/pgp/server-8.0.asc | \
  sudo gpg --dearmor -o /usr/share/keyrings/mongodb-server-8.0.gpg
echo "deb [ signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/debian bookworm/mongodb-org/8.0 main" | \
  sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list
sudo apt-get update
sudo apt-get install -y mongodb-org

sudo systemctl enable --now mongod
sudo systemctl status mongod --no-pager
```

Test MongoDB:

```bash
mongosh --eval 'db.runCommand({ ping: 1 })'
```

---

## 3) Docker fallback for MongoDB (if native package fails)

If `mongodb-org` does not install cleanly on your Debian 13 machine, use Docker:

```bash
sudo apt-get install -y docker.io
sudo systemctl enable --now docker
sudo docker run -d \
  --name mongodb \
  -p 27017:27017 \
  -v mongo-data:/data/db \
  mongo:8
```

Mongo URL:

```text
mongodb://127.0.0.1:27017/payload-app
```

---

## 4) Create and run a Payload app

```bash
npx create-payload-app@latest -n payload-app -t blank --use-npm
cd payload-app
```

When prompted for a database, choose **MongoDB**.

Create `.env`:

```bash
cat > .env << 'EOF'
DATABASE_URI=mongodb://127.0.0.1:27017/payload-app
PAYLOAD_SECRET=replace-with-a-long-random-string
NEXT_PUBLIC_SERVER_URL=http://localhost:3000
EOF
```

Tip: generate a secret quickly:

```bash
openssl rand -base64 32
```

Start the app:

```bash
npm run dev
```

Then open:

- App: `http://localhost:3000`
- Admin: `http://localhost:3000/admin`

---

## 5) If your generated app is not on MongoDB

Install adapter:

```bash
npm install @payloadcms/db-mongodb
```

In your Payload config, use:

```ts
import { mongooseAdapter } from '@payloadcms/db-mongodb'

db: mongooseAdapter({
  url: process.env.DATABASE_URI || '',
})
```