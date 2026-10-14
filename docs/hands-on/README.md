# RoboShop - Hands-On: My Lab Run (2026-10-14)

[← Back to the setup guide](../README.md) · [Project overview](../../README.md)

I built the full RoboShop app on AWS: **11 servers**, 4 databases + 6 backend services in 4 languages + an Nginx frontend, all connected by **DNS names** on my own domain `abhignadevops.store`.

> My email is blacked out in the screenshots. The servers are deleted after the lab, so the IPs in the screenshots don't exist anymore.

![RoboShop databases](../images/roboshop-databases.svg)

## Table of Contents

1. [My servers](#1-my-servers)
2. [MongoDB](#2-mongodb)
3. [Redis](#3-redis)
4. [MySQL](#4-mysql)
5. [RabbitMQ](#5-rabbitmq)
6. [DNS records for the databases](#6-dns-records-for-the-databases)
7. [Catalogue (Node.js)](#7-catalogue-nodejs)
8. [User (Node.js)](#8-user-nodejs)
9. [Cart (Node.js)](#9-cart-nodejs)
10. [Shipping (Java)](#10-shipping-java)
11. [Payment (Python)](#11-payment-python)
12. [Dispatch (Go)](#12-dispatch-go)
13. [Frontend (Nginx)](#13-frontend-nginx)
14. [Domain points to the frontend](#14-domain-points-to-the-frontend)
15. [Place a real order](#15-place-a-real-order)
16. [System status page](#16-system-status-page)
17. [Test: dispatch down, order waits in the queue](#17-test-dispatch-down-order-waits-in-the-queue)
18. [Summary](#18-summary)

---

## 1. My servers

All 11: t3.micro, AMI `Redhat-9-DevOps-Practice`, us-east-1, each with its own security group.

| Server | Private IP | DNS name | Runs |
|--------|------------|----------|------|
| mongodb | 172.31.13.80 | `mongodb.abhignadevops.store` | MongoDB 7.0 |
| redis | 172.31.2.177 | `redis.abhignadevops.store` | Redis 7 |
| mysql | 172.31.4.67 | `mysql.abhignadevops.store` | MySQL 8.4.11 |
| rabbitmq | 172.31.7.243 | `rabbitmq.abhignadevops.store` | RabbitMQ 4.3.6 |
| catalogue | 172.31.8.66 | `catalogue.abhignadevops.store` | Node.js 24 |
| user | 172.31.12.7 | `user.abhignadevops.store` | Node.js 24 |
| cart | 172.31.11.36 | `cart.abhignadevops.store` | Node.js 24 |
| shipping | 172.31.15.175 | `shipping.abhignadevops.store` | Java 25 (Spring Boot) |
| payment | 172.31.8.73 | `payment.abhignadevops.store` | Python 3.14 + gunicorn |
| dispatch | 172.31.0.64 | - (nobody connects to it) | Go 1.26 |
| frontend | 172.31.14.156 | `abhignadevops.store` (public IP) | Nginx 1.26 |

Order: databases first → backend services → frontend. I launched and set up **one server at a time**.

---

## 2. MongoDB

Changed `bindIp` from `127.0.0.1` to `0.0.0.0` in `/etc/mongod.conf`, so catalogue and user (other servers) can connect.

![mongod.conf bindIp](./images/mongodb-01-bindip.png)

Restarted → `ss -lntp` shows `0.0.0.0:27017`.

![MongoDB running on 0.0.0.0:27017](./images/mongodb-02-running-port.png)

---

## 3. Redis

Edited `/etc/redis/redis.conf` (changed `bind` to `0.0.0.0`), then enabled + started Redis.

![Redis config and start](./images/redis-01-config-start.png)

`0.0.0.0:6379` + `redis-cli ping` → `PONG`.

![Redis port and ping](./images/redis-02-port-ping.png)

> I missed `protected-mode`: it was still `yes`. I found it later when user showed `"redis":false`. See [Troubleshooting #1](../troubleshooting.md#1-user-shows-redisfalse).

---

## 4. MySQL

`systemctl enable` creates the symlink in `multi-user.target.wants/` → starts at every boot.

![MySQL enable symlink](./images/mysql-01-enable-symlink.png)

Set the root password → `mysql -uroot` without password is denied (expected) → version `8.4.11` → listening on `*:3306`.

![MySQL root password and version](./images/mysql-02-root-password-version.png)

---

## 5. RabbitMQ

Started RabbitMQ and created the app user `roboshop`.

![RabbitMQ start and add user](./images/rabbitmq-01-start-add-user.png)

Running, ports `5672` (apps), `15672` (web UI), `25672` (clustering).

![RabbitMQ status and ports](./images/rabbitmq-02-status-ports.png)

Web UI at `http://<rabbitmq-public-ip>:15672`, logged in as `roboshop`.

![RabbitMQ web UI](./images/rabbitmq-03-web-ui.png)

---

## 6. DNS records for the databases

Deleted the old expense-project records, then added one **A record per server with its private IP** (TTL 30).

![Database records in Route 53](./images/dns-01-database-records.png)

After every server: **Create record** → name + private IP.

![Create record](./images/dns-02-create-record.png)

---

## 7. Catalogue (Node.js)

Node.js 24 → app user `roboshop` → download + unzip into `/app`.

![Catalogue download](./images/catalogue-01-node-user-download.png)

`npm install` → `node_modules` folder created.

![npm install](./images/catalogue-02-npm-install.png)

Service file with the MongoDB **DNS name** instead of an IP.

![catalogue.service](./images/catalogue-03-service-file.png)

Loaded the 11 products into MongoDB with `mongosh` (only once).

![Load master data](./images/catalogue-04-load-master-data.png)

Count = `11`, service running, log shows `MongoDB connected`.

![Count and status](./images/catalogue-05-count-status.png)

`/health` → `{"app":"OK","mongo":true}` and `/products` → all robots.

![Health and products](./images/catalogue-06-health-products.png)

Looked inside MongoDB: `show dbs` → `use catalogue` → `show collections` → `db.products.find()`.

![mongosh](./images/catalogue-07-mongosh.png)

---

## 8. User (Node.js)

Same steps as catalogue. Service file needs **2** addresses: Redis and MongoDB.

![user.service](./images/user-01-service-file.png)

Started → the log kept showing **Redis errors**.

![User start with Redis errors](./images/user-02-start-redis-errors.png)

`/health` → `"redis":false`, log `Socket closed unexpectedly`.

![redis false](./images/user-03-redis-false.png)

**Fix:** on redis, `protected-mode no` + `systemctl restart redis`, then restarted user → `Redis connected` + `MongoDB connected`. Full story: [Troubleshooting #1](../troubleshooting.md#1-user-shows-redisfalse).

![Redis connected](./images/user-04-redis-connected.png)

Login with the demo user → `HTTP/1.1 200 OK`.

![Login 200](./images/user-05-login-200.png)

---

## 9. Cart (Node.js)

Service file needs Redis (save carts) and catalogue (price and stock).

![cart.service](./images/cart-01-service-file.png)

Started → `Redis connected` → `/health` → `"redis":true`.

![Cart start and health](./images/cart-02-start-health.png)

Added 1 Robotic Mining Cyborg to a test cart → `200`, total 42. This proves cart reaches **both** Redis and catalogue.

![Add to cart](./images/cart-03-add-to-cart.png)

---

## 10. Shipping (Java)

Java 25 as default (`alternatives --config java`), `JAVA_HOME` in `/etc/profile.d/java.sh`, Maven uses Java 25.

![Java and Maven](./images/shipping-01-java-maven.png)

Downloaded the source and built it with `mvn clean package`.

![Download and build](./images/shipping-02-download-build.png)

`BUILD SUCCESS` → moved `target/shipping.jar` to `/app/shipping.jar`.

![Build success](./images/shipping-03-build-success.png)

Loaded **948,833 cities** into MySQL, created the app user `shipping` (it only sees the `cities` database).

![Load cities and app user](./images/shipping-04-load-cities-app-user.png)

Service file with the cart and MySQL DNS names.

![shipping.service](./images/shipping-05-service-file.png)

Started → Spring Boot log → connected to MySQL.

![Shipping started](./images/shipping-06-started.png)

```text
/actuator/health → app alive + MySQL connected   → "status":"UP"
/count           → can actually READ data         → 948833
/codes           → the real feature works         → 25 countries
```

---

## 11. Payment (Python)

Python 3.14 → virtual env `/app/.venv` → `pip install -r requirements.txt`.

![Python venv](./images/payment-01-python-venv.png)

Service file needs cart, user and RabbitMQ.

![payment.service](./images/payment-02-service-file.png)

gunicorn listening on `0.0.0.0:8080` → `/health` → `"rabbitmq":true`.

![Payment running and health](./images/payment-03-running-health.png)

---

## 12. Dispatch (Go)

Go → download → `go build -o dispatch .` → one binary `/app/dispatch`.

![Go build](./images/dispatch-01-go-build.png)

Only needs RabbitMQ.

![dispatch.service](./images/dispatch-02-service-file.png)

Log: `connected to RabbitMQ, waiting for orders`. No web port, so no `curl` here.

![Dispatch connected](./images/dispatch-03-connected.png)

---

## 13. Frontend (Nginx)

Nginx 1.26 → removed the default page → unzipped the website into `/usr/share/nginx/html`.

![Nginx and website files](./images/frontend-01-nginx-deploy.png)

Reverse proxy in `/etc/nginx/default.d/roboshop.conf`: every `/api/<service>/` goes to the service by **DNS name**.

![roboshop.conf proxy_pass](./images/frontend-02-proxy-conf.png)

`nginx -t` → syntax ok → restart → running.

![nginx -t and restart](./images/frontend-03-nginx-t-restart.png)

Website on the frontend public IP.

![Website by IP](./images/frontend-04-website-by-ip.png)

---

## 14. Domain points to the frontend

Edited the root A record `abhignadevops.store` → **frontend public IP** (only the frontend gets a public IP record).

![Root record to frontend](./images/dns-03-root-to-frontend.png)

All 13 records: NS, SOA, root A, `www` CNAME and 9 private A records.

![All records](./images/dns-04-all-records.png)

---

## 15. Place a real order

Created my own account.

![Create account](./images/web-01-create-account.png)

All 11 products load (catalogue → MongoDB).

![All products](./images/web-02-all-products.png)

Added Stan + Watson to the cart (cart → Redis).

![Cart](./images/web-03-cart.png)

Shipping to Portugal, Lamego → 1,597 km → €79.85 (shipping → MySQL cities).

![Shipping cost](./images/web-04-shipping-cost.png)

Review and pay (payment).

![Review and pay](./images/web-05-review-and-pay.png)

Order confirmed (payment → user saves order → RabbitMQ → dispatch).

![Order confirmed](./images/web-06-order-confirmed.png)

My order history: 3 orders.

![Order history](./images/web-07-order-history.png)

```text
frontend → user → catalogue → cart → shipping → payment → user → cart → RabbitMQ → dispatch
```

---

## 16. System status page

`http://abhignadevops.store/status` calls every `/health` URL every 10 seconds → **6/6 healthy**.

![System status](./images/web-08-system-status.png)

---

## 17. Test: dispatch down, order waits in the queue

1. `systemctl stop dispatch`
2. Placed an order → payment still worked
3. RabbitMQ → `orders` queue → **Ready = 1** (the order is waiting)

![Order waiting in the queue](./images/queue-01-order-waiting.png)

4. `systemctl start dispatch` → it took the waiting order **the same second** → `order received` → `order dispatched` with a tracking id

![Dispatch catches up](./images/queue-02-dispatch-catches-up.png)

5. Queue back to **0**. No order lost.

![Queue empty](./images/queue-03-queue-empty.png)

> **Why a queue:** if dispatch is down, orders wait safely in RabbitMQ and the customer doesn't notice. Dispatch catches up when it's back.

---

## 18. Summary

| What | Result |
|------|--------|
| Servers | 11 × t3.micro (needed a vCPU quota raise 16 → 32) |
| Databases | MongoDB, Redis, MySQL, RabbitMQ |
| Languages | Node.js, Java, Python, Go |
| Connections | DNS names in Route 53, not IPs |
| Security | 11 security groups, only the needed ports between servers |
| Data | 11 products, 948,833 cities |
| Test | 4 real orders end to end, 1 waited in the queue while dispatch was down |
| Problem I fixed | User `"redis":false` → Redis `protected-mode` → [Troubleshooting #1](../troubleshooting.md#1-user-shows-redisfalse) |
