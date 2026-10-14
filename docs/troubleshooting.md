# RoboShop Troubleshooting

[← Back to the setup guide](README.md) · [Project overview](../README.md)

Problems I faced while building RoboShop, and how I fixed them.

| # | Problem | Server |
|---|---------|--------|
| 1 | [User shows `"redis":false`](#1-user-shows-redisfalse) | redis, user |

---

## 1. User shows `"redis":false`

### Why does the user server reach Redis?

The user service stores **2 kinds of data in 2 places**:

| Data | Where | Why there |
|------|-------|-----------|
| Accounts, passwords (hashed), orders | **MongoDB** | Must be saved permanently |
| **Guest id counter** | **Redis** | Needs to be very fast |

**Guest id:** when someone opens RoboShop **without logging in**, they still need an id so their cart can be saved. User asks Redis for "the next number", and Redis counts up: 1, 2, 3...

```
Visitor opens website ──► user server ──► Redis: "next guest id?" ──► "42"
                                          (cart is saved under id 42)
```

Counting in RAM is super fast, and two visitors never get the same number.

### What was happening

```
user server ──► DNS ✅ ──► SG ✅ ──► bind 0.0.0.0 ✅ ──► protected-mode yes ❌
                                                         "No password and you're from
                                                          outside? Go away." → closes connection
```

- The user log (`journalctl -u user`) showed `Socket closed unexpectedly`, retrying every 2 seconds
- `curl http://localhost:8080/health` showed `{"app":"OK","mongo":true,"redis":false}`

| protected-mode | Who can connect from other servers |
|----------------|------------------------------------|
| `yes` | Only clients **with a password** (no password set = nobody from outside) |
| `no` | Any server, no password needed (the SG decides who gets in) |

We didn't set a Redis password, and protected mode was still `yes`, so Redis refused the user server.

### What I did, step by step

| Step | What | Result |
|------|------|--------|
| 1 | Read the user log | `Socket closed unexpectedly`, so the connection **opens but gets closed** |
| 2 | Checked the SG and DNS | Both fine, so it's not the network |
| 3 | Checked the Redis config | `protected-mode yes` ← found it |
| 4 | Set it to `no` + restart redis | Redis now accepts other servers |
| 5 | Restart user | User connects again → `"redis":true` |

```shell
# on redis
grep -E "^bind|^protected-mode" /etc/redis/redis.conf
sed -i 's/^protected-mode yes/protected-mode no/' /etc/redis/redis.conf
systemctl restart redis
redis-cli config get protected-mode      # "no" = what Redis is actually running with

# on user
systemctl restart user
curl http://localhost:8080/health        # "redis":true
```

> Use `systemctl restart` after changing a config file. `start` does nothing if the service is already running.

### How to tell connection errors apart

| If the problem is... | The error looks like |
|----------------------|----------------------|
| SG blocking | timeout |
| Service not listening | connection refused |
| Protected mode | connection opens, then gets closed |

> **Lesson:** I changed `bind` during setup but missed `protected-mode`. For Redis, always check both.
