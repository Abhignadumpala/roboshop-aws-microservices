# User

User is a microservice that handles login, registration and order history in RoboShop. Passwords are stored as bcrypt hashes, never as plain text.

> **Developer has chosen Node.js. Check with the developer which version is needed. This setup requires Node.js >= 24.**

---

## Install Node.js

**You can list modules by using `dnf module list nodejs`**

```shell
dnf module disable nodejs -y
dnf module enable nodejs:24 -y
dnf install nodejs -y
```

---

## Create Application User

```shell
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
```

---

## Download the Application

```shell
mkdir /app
curl -L -o /tmp/user.zip https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/user-v4.zip
cd /app
unzip /tmp/user.zip
```

---

## Install Dependencies

```shell
cd /app
npm install
```

---

## Configure SystemD Service

### Why do we create a service file?

Without a service file, you would have to start the app by hand, and these problems happen:

| Problem when running it by hand | What the service file does instead |
|---------------------------------|------------------------------------|
| Close the terminal and the app **dies** | Runs in the **background** |
| Server reboots and the app is **gone** | `enable` starts it **at every boot** |
| App crashes and stays **down** | `Restart=on-failure` brings it **back in 5 seconds** |
| Runs as **root** (unsafe) | `User=roboshop` runs it as a normal user |
| Logs only on screen | Logs go to `journalctl -u user` |

By adding the **environment** address values, we are telling user the address of **MongoDB and Redis**, so it can connect to them.

```shell
vim /etc/systemd/system/user.service
```

```ini
[Unit]
Description=User Service
After=network-online.target

[Service]
User=roboshop
Environment=REDIS_URL="redis://<REDIS-SERVER-IPADDRESS>:6379"
Environment=MONGO_URL="mongodb://<MONGODB-SERVER-IPADDRESS>:27017/users"
ExecStart=/bin/node /app/server.js
SyslogIdentifier=user
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Replace `<REDIS-SERVER-IPADDRESS>` and `<MONGODB-SERVER-IPADDRESS>` with the private IPs of those servers.**

**Each line of the file**

| Line | Meaning |
|------|---------|
| `Description=User Service` | Name shown in `systemctl status` |
| `After=network-online.target` | Start only **after the network is up** (user needs MongoDB and Redis over the network) |
| `User=roboshop` | Run as the `roboshop` user, not root |
| `Environment=REDIS_URL=...` | Where Redis is (guest id counter) |
| `Environment=MONGO_URL=...` | Where MongoDB is, and the database name (`users`), for users and orders |
| `ExecStart=/bin/node /app/server.js` | The command that starts the Node.js app |
| `SyslogIdentifier=user` | Name shown in the logs (`journalctl -u user`) |
| `Restart=on-failure` + `RestartSec=5` | If it crashes, restart after 5 seconds |
| `WantedBy=multi-user.target` | Used by `enable`, to start at boot (creates the symlink in `multi-user.target.wants/`) |

Load and start the service:

```shell
systemctl daemon-reload
systemctl enable user
systemctl start user
```

---

## Verification

```shell
systemctl status user
journalctl -u user -f
curl http://localhost:8080/health
```

`/health` returns `{"app":"OK","mongo":true,"redis":true}`. If either database is down you get `503`, and the log shows which one.

Try a login with the demo user (loaded by catalogue's master data):

```shell
curl -i -X POST http://localhost:8080/login -H 'Content-Type: application/json' -d '{"name":"roboshop","password":"RoboShop@1"}'
```

`200` means it works. A wrong password gives `401`. The password is never written to the log.

---

## API

| Method | Path | Notes |
|--------|------|-------|
| GET | `/health` | 200, or 503 if MongoDB or Redis is down |
| GET | `/uniqueid` | a new guest id from Redis |
| POST | `/register` | `{name, email, password}` → 201, 400 missing fields, 409 name taken |
| POST | `/login` | `{name, password}` → 200, 401 wrong login |
| GET | `/check/<name>` | 200 or 404, used by payment |
| POST | `/order/<name>` | saves an order, used by payment |
| GET | `/history/<name>` | order history |
| GET | `/users` | all users, without passwords |
| GET | `/metrics` | Prometheus metrics |

---

## Security Group

Allow **inbound TCP 8080** from `roboshop-frontend` and `roboshop-payment`.
