# Cart

Cart is a microservice that keeps each shopper's cart in Redis. When an item is added, Cart asks Catalogue for the current price and stock, so a cart never holds more than what is in stock.

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
curl -L -o /tmp/cart.zip https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/cart-v4.zip
cd /app
unzip /tmp/cart.zip
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
| Logs only on screen | Logs go to `journalctl -u cart` |

By adding the **environment** address values, we are telling cart the address of **Redis and catalogue**, so it can connect to them.

```shell
vim /etc/systemd/system/cart.service
```

```ini
[Unit]
Description=Cart Service
After=network-online.target

[Service]
User=roboshop
Environment=REDIS_HOST=<REDIS-SERVER-IPADDRESS>
Environment=CATALOGUE_HOST=<CATALOGUE-SERVER-IPADDRESS>
Environment=CATALOGUE_PORT=8080
ExecStart=/bin/node /app/server.js
SyslogIdentifier=cart
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

> **Replace `<REDIS-SERVER-IPADDRESS>` and `<CATALOGUE-SERVER-IPADDRESS>` with the private IPs of those servers.**

**Each line of the file**

| Line | Meaning |
|------|---------|
| `Description=Cart Service` | Name shown in `systemctl status` |
| `After=network-online.target` | Start only **after the network is up** (cart needs Redis and catalogue over the network) |
| `User=roboshop` | Run as the `roboshop` user, not root |
| `Environment=REDIS_HOST=...` | Where Redis is (to save carts) |
| `Environment=CATALOGUE_HOST=...` | Where catalogue is (to check price and stock) |
| `Environment=CATALOGUE_PORT=8080` | Catalogue's port |
| `ExecStart=/bin/node /app/server.js` | The command that starts the Node.js app |
| `SyslogIdentifier=cart` | Name shown in the logs (`journalctl -u cart`) |
| `Restart=on-failure` + `RestartSec=5` | If it crashes, restart after 5 seconds |
| `WantedBy=multi-user.target` | Used by `enable`, to start at boot (creates the symlink in `multi-user.target.wants/`) |

Optional: `CART_TTL_SECONDS` (default `3600`) is how long an unchanged cart is kept in Redis.

Load and start the service:

```shell
systemctl daemon-reload
systemctl enable cart
systemctl start cart
```

---

## Verification

```shell
systemctl status cart
journalctl -u cart -f
curl http://localhost:8080/health
```

`/health` returns `{"app":"OK","redis":true}`.

Add a product to a test cart. This also checks the connection to Catalogue:

```shell
curl -i -X POST http://localhost:8080/add/test/RMC/1
curl http://localhost:8080/cart/test
```

| Response | Meaning |
|----------|---------|
| `200` | Works |
| `404` | Product not found |
| `409` | Not enough stock |
| `502` | Cart cannot reach Catalogue (check the IP, the Catalogue service and the security group) |
| `504` | Catalogue did not answer within 5 seconds |
| `503` | Redis is down |

---

## API

| Method | Path | Notes |
|--------|------|-------|
| GET | `/health` | 200, or 503 if Redis is down |
| GET | `/cart/<id>` | the cart, 404 if empty |
| DELETE | `/cart/<id>` | empty the cart, used by payment |
| POST | `/add/<id>/<sku>/<qty>` | add items |
| POST | `/update/<id>/<sku>/<qty>` | change quantity, `0` removes the item |
| GET | `/rename/<from>/<to>` | move the guest cart to the user at login |
| POST | `/shipping/<id>` | add the shipping cost, used by shipping |
| GET | `/metrics` | Prometheus metrics |

---

## Security Group

Allow **inbound TCP 8080** from `roboshop-frontend`, `roboshop-shipping` and `roboshop-payment`.
