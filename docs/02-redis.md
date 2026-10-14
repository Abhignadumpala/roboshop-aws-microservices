# Redis

Redis (REmote DIctionary Server) is an open-source, in-memory data store used as a database and cache. It is very fast because it keeps data in RAM. It uses a simple key-value model but supports complex data types.

RoboShop uses Redis for:

| Used by | What it stores |
|---------|----------------|
| Cart | Every cart, as one key per user (expires after 1 hour without changes) |
| User | A counter that gives each guest visitor a unique id (`anonymous-1`, `anonymous-2`, ...) |

> **Check with the developer for the exact version required. This setup uses Redis 7.**
>
> Redis 7 is the newest Redis on RHEL 9. Newer RHEL releases also offer **Valkey 8**, an open-source fork of Redis that works with the same clients.

---

## Install

By default Redis 6 is enabled. Enable version 7 and install it:

**You can list modules by using `dnf module list redis`**

```shell
dnf module disable redis -y
dnf module enable redis:7 -y
dnf install redis -y
```

---

## Configure

By default Redis listens only on `localhost (127.0.0.1)`. User and Cart run on other servers, so Redis must listen on all addresses.

Edit the config file:

```shell
vim /etc/redis/redis.conf
```

- Update `bind 127.0.0.1 -::1` to `bind 0.0.0.0 -::1`
- Update `protected-mode yes` to `protected-mode no`

Or do the same with one command:

```shell
sed -i -e 's/^bind 127.0.0.1/bind 0.0.0.0/' -e 's/^protected-mode yes/protected-mode no/' /etc/redis/redis.conf
```

### Why both changes? (two locks)

Redis has **two separate locks**. Both must be opened for other servers to connect:

```
user / cart server ──► [ Lock 1: bind ] ──► [ Lock 2: protected-mode ] ──► Redis data
```

| Lock | Question it asks | Default | We set |
|------|------------------|---------|--------|
| **bind** | Which door am I listening on? | `127.0.0.1` = only the inside door | `0.0.0.0` = all doors |
| **protected-mode** | No password set and someone from outside? Then refuse them | `yes` | `no` |

If only `bind` is changed, the door is open but the guard is still standing there, sending everyone away.

**Real-life example:** you unlocked your front door (bind), but a security guard stands inside and says "no visitors without a pass" (protected mode). Visitors come through the door and get sent right back.

**Why does Redis have this guard?** By default Redis has no password. Thousands of Redis servers on the internet got hacked because people opened them without one. So Redis added protected mode: "if there's no password, don't talk to strangers."

**Is it safe to turn it off?** Here, yes, because the security group does the guard's job:

```
Internet ──✗── SG blocks 6379 ──► only roboshop-user and roboshop-cart get in
```

| Setup | Who protects Redis |
|-------|--------------------|
| Our practice | SG (network level), protected-mode off |
| Real companies | SG **plus** a Redis password (`requirepass`), protected-mode stays on |

**Compare with MongoDB:** MongoDB has only one lock (`bindIp`), so changing `bindIp` was enough. Redis needs both.

> **Interview one-liner:** Redis protected mode refuses remote connections when no password is set. To let app servers connect, either set a password or disable protected mode and rely on the security group.

Start & enable the Redis service:

```shell
systemctl enable redis
systemctl start redis
```

---

## Verification

```shell
systemctl status redis
ss -lntp | grep 6379
redis-cli ping
```

`redis-cli ping` should answer `PONG`.

> User shows `"redis":false`? See [Troubleshooting #1](troubleshooting.md#1-user-shows-redisfalse).

After you add something to the cart on the website, you can see it in Redis:

```shell
redis-cli keys '*'
redis-cli get <user-id>
```
