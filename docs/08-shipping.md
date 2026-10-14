# Shipping

Shipping works out how far a package has to travel and what it costs to ship it. It looks up the city in MySQL, measures the distance from the warehouse, and adds the shipping cost to the cart.

Shipping is written in Java with Spring Boot 4. **Maven** is the Java build tool: it downloads the libraries the application needs and packages everything into one `.jar` file.

> **Developer has chosen Java and Maven. Check with the developer which versions are needed. This setup requires Java >= 25 and Maven >= 3.9.**

---

## Java Basics

**Why Java?**

- Famous programming language, mostly used at **enterprise level** (banks, big companies).
- Can run **many tasks at the same time** (multi-threading). Node.js runs my code on a single thread.
- Uses **more CPU and memory** than Node.js - that's why the shipping server needs more RAM.

| | Java |
|---|---|
| Extension | `.java` |
| Build tool | Maven |
| Build file | `pom.xml` → name, version, description, dependencies, etc. |
| Install dependencies | `mvn package` (like `npm install` in Node.js) |
| Dependencies stored in | `~/.m2/repository` |

### Where dependencies are stored

| Node.js | Maven (Java) |
|---|---|
| Each app has its **own** `node_modules` folder | **One common** folder for all apps: `~/.m2/repository` |
| `app-1/node_modules`<br>`app-2/node_modules`<br>`app-3/node_modules` | `~/.m2/repository/http-request-1.0.0`<br>`~/.m2/repository/http-request-2.0.0` |

**How can one common folder keep 2 versions of the same dependency? → GAV**

- Maven names every component with **GAV** = **G**roup-id, **A**rtifact-id, **V**ersion.
- GAV makes each component unique in the universe.
- So `http-request-1.0.0` and `http-request-2.0.0` sit side by side without clashing.

### Compilation

**Compilation** = checking the source code is correct and converting it.

```
Java:     source code (.java) → compile → bytecode → binary language → runs
Node.js:  source code (.js)  → no compile step → compiled and run at runtime
```

- **Java** → must compile **before** running; only then I get bytecode.
- **Node.js** → no compile step; it compiles while running.

### Maven Lifecycle

**Build tool** = packages the source code into an **artifact** (for Java, a `.jar` file).

The Maven lifecycle is a **sequential process execution** to build the project with different **phases** - the phases run one after another, in this order:

```
clean → install dependencies → compile → test → package → install
```

| # | Phase | What happens |
|---|---|---|
| 1 | Clean | Delete the old build (`target/` folder) |
| 2 | Install dependencies | Download them into `~/.m2/repository` |
| 3 | Compile | Check the source code and compile it |
| 4 | Test | Run the test cases |
| 5 | Package | Convert to bytecode and pack into one `.jar` → this is **packaging** |
| 6 | Install | Copy the `.jar` into `~/.m2/repository`, so other projects on the machine can use it |

- If one phase fails (e.g. a test fails), the next phases don't run.
- Running a phase runs every phase before it too.
- Here I use `mvn clean package` → runs phases 1-5 → artifact lands in `target/shipping.jar`.

### JDK vs JRE

| | Full form | What it has |
|---|---|---|
| **JRE** | Java Runtime Environment | Only what's needed to **run** Java |
| **JDK** | Java Development Kit | JRE + extra tools and packages for **development** (e.g. `javac` compiler) |

- JRE is a subset of JDK → **JRE < JDK**
- **JDK = JRE + extra tools for development**
- While developing / building (like `mvn package` here) → need the **JDK** (that's why we install `java-25-openjdk-devel`).

More on how Java compares with Node.js, Python and Go: [Day 11 - Backend Deployment](https://github.com/Abhignadumpala/devops-notes/blob/main/day-11-backend-deployment-languages/README.md)

---

## Install Java and Maven

**You can list modules by using `dnf module list maven`**

```shell
dnf module enable maven:3.9 -y
dnf install maven java-25-openjdk-devel -y
```

`dnf install maven` also brings an older Java (21), and Maven uses it by default. This application needs Java 25, so make Java 25 the default **once**:

```shell
alternatives --config java
```

It lists the installed Java versions. Type the number of the line with `java-25-openjdk` and press Enter. Now the `java` command points to Java 25.

**Why `JAVA_HOME`?** Maven should know **which Java** to use and **where it's installed**.

- `JAVA_HOME` = the Java 25 folder, e.g. `/usr/lib/jvm/java-25-openjdk-...`
- I save it in `/etc/profile.d/java.sh` so it's set at every login.
- `echo $JAVA_HOME` → prints the folder, so I know it's set.

Maven needs `JAVA_HOME`, the Java 25 folder. The folder name contains the full version, so take it from the `java` command instead of typing it:

```shell
echo "export JAVA_HOME=$(dirname $(dirname $(readlink -f /usr/bin/java)))" > /etc/profile.d/java.sh
source /etc/profile.d/java.sh
echo $JAVA_HOME
```

- `readlink -f /usr/bin/java` gives the real file, for example `/usr/lib/jvm/java-25-openjdk-25.0.1.0.8-1.el9.x86_64/bin/java`. The two `dirname` remove `/bin/java` from the end.
- Files in `/etc/profile.d/` run at every login, so you never have to type it again, even after a reboot.

Verify:

```shell
java -version
mvn -v
```

Both should show version `25`.

---

## Create Application User

```shell
useradd --system --home /app --shell /sbin/nologin --comment "roboshop system user" roboshop
```

---

## Download the Application

```shell
mkdir /app
curl -L -o /tmp/shipping.zip https://raw.githubusercontent.com/daws-92s/roboshop-documentation/refs/heads/main/artifacts/shipping-v4.zip
cd /app
unzip /tmp/shipping.zip
```

---

## Build the Application

```shell
cd /app
mvn clean package
mv target/shipping.jar shipping.jar
```

**What `mvn clean package` does:**

| Part | Meaning |
|---|---|
| `clean` | A Maven phase - deletes the old build |
| `package` | Validate → install dependencies → compile → test → package, all in one |

Result → `target/shipping.jar`

The first build takes a few minutes, because Maven downloads all the libraries.

---

## Load Master Data

Load the data **before** starting the service. This includes all countries and their cities with their locations. The developer provides the schema and the app user in the `db` folder.

Install the MySQL client:

```shell
dnf install mysql -y
```

Load the cities. This creates the `cities` database and its tables:

```shell
mysql -h <MYSQL-SERVER-IPADDRESS> -uroot -pRoboShop@1 < /app/db/master-data.sql
```

Create the app user. The application logs in as `shipping`, not as `root`, and it can only use the `cities` database:

```shell
mysql -h <MYSQL-SERVER-IPADDRESS> -uroot -pRoboShop@1 < /app/db/app-user.sql
```

The import is large (about 60 MB) and takes a minute. It is safe to run again: it drops and recreates the tables.

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
| Logs only on screen | Logs go to `journalctl -u shipping` |

By adding the **environment** address values, we are telling shipping the address of **MySQL and cart**, so it can connect to them.

```shell
vim /etc/systemd/system/shipping.service
```

```ini
[Unit]
Description=Shipping Service
After=network-online.target

[Service]
User=roboshop
Environment=CART_ENDPOINT=<CART-SERVER-IPADDRESS>:8080
Environment=DB_HOST=<MYSQL-SERVER-IPADDRESS>
Environment=DB_USER=shipping
Environment=DB_PASSWORD=RoboShop@1
ExecStart=/bin/java -jar /app/shipping.jar
SyslogIdentifier=shipping
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

**Each line of the file**

| Line | Meaning |
|------|---------|
| `Description=Shipping Service` | Name shown in `systemctl status` |
| `After=network-online.target` | Start only **after the network is up** (shipping needs MySQL and cart over the network) |
| `User=roboshop` | Run as the `roboshop` user, not root |
| `Environment=CART_ENDPOINT=...` | Where cart is (`name:port`), to add the shipping cost to the cart |
| `Environment=DB_HOST=...` | Where MySQL is (cities and shipping cost data) |
| `Environment=DB_USER=shipping` | MySQL user the app logs in as |
| `Environment=DB_PASSWORD=...` | That user's password |
| `ExecStart=/bin/java -jar /app/shipping.jar` | The command that starts the Java app |
| `SyslogIdentifier=shipping` | Name shown in the logs (`journalctl -u shipping`) |
| `Restart=on-failure` + `RestartSec=5` | If it crashes, restart after 5 seconds |
| `WantedBy=multi-user.target` | Used by `enable`, to start at boot (creates the symlink in `multi-user.target.wants/`) |

> **Replace `<CART-SERVER-IPADDRESS>` and `<MYSQL-SERVER-IPADDRESS>` with the private IPs of those servers.**

Load and start the service:

```shell
systemctl daemon-reload
systemctl enable shipping
systemctl start shipping
```

---

## Verification

Spring Boot takes 10-20 seconds to start. Watch the log until you see `Started ShippingApplication`:

```shell
systemctl status shipping
journalctl -u shipping -f
```

```shell
curl http://localhost:8080/actuator/health
curl http://localhost:8080/count
curl http://localhost:8080/codes
```

```text
/actuator/health → app alive + MySQL connected
/count           → can actually READ data (number of cities)
/codes           → the real feature works (country list on the website)
```

`/actuator/health` returns `{"status":"UP"}` and includes the database status. `/count` returns the number of cities.

| Problem | Meaning |
|---------|---------|
| Log shows `Access denied for user 'shipping'` | `app-user.sql` was not loaded |
| Log shows `Communications link failure` | Shipping cannot reach MySQL: check the IP and port 3306 in the security group |
| Log shows `Cannot find any provider supporting RSA/ECB/OAEPWithSHA-1AndMGF1Padding` | Old `shipping-v4.zip` that connects to MySQL without TLS. Download the zip again and rebuild |
| `503` from `/codes` | MySQL went down while shipping was running |
| `502` from `/confirm/...` | Shipping cannot reach Cart |

---

## API

| Method | Path | Notes |
|--------|------|-------|
| GET | `/health` | `OK` |
| GET | `/codes` | list of countries |
| GET | `/match/<code>/<text>` | city search, needs at least 3 letters, otherwise 400 |
| GET | `/calc/<cityId>` | `{distance, cost}`, 404 for an unknown city |
| POST | `/confirm/<id>` | adds shipping to the cart |
| GET | `/count` | number of cities |
| GET | `/memory` / `/free` | memory leak demo: each call to `/memory` keeps 25 MB more |
| GET | `/actuator/health`, `/actuator/prometheus` | Spring Boot health and metrics |

---

## Security Group

Allow **inbound TCP 8080** from `roboshop-frontend` only.

---

## Summary - Java

| Topic | In short |
|---|---|
| Extension | `.java` |
| Build tool | Maven |
| Build file | `pom.xml` |
| GAV | Group-id, Artifact-id, Version |
| JDK | For **development** - heavy |
| JRE | For **running** the bytecode - lightweight |
| JDK vs JRE | JDK = JRE + development tools |
| Maven lifecycle | Phases run one after another: clean → dependencies → compile → test → package → install |
| Dependencies stored in | `~/.m2/repository` |
| `mvn clean package` | Creates the `target/` folder → `.jar` file inside |
| `.jar` | **J**ava **AR**chive file → holds the bytecode |

---

## Interview Questions - Java

**1. What is Maven?**

The build tool for Java. It downloads the dependencies, compiles the code, runs the tests and packages the app into a `.jar`.

**2. What is `pom.xml`?**

Maven's build file. It has the project's name, version, description and dependencies - like `package.json` in Node.js.

**3. Where does Maven store dependencies?**

In one common folder for all projects: `~/.m2/repository`. Node.js instead keeps a `node_modules` folder inside each app.

**4. What is GAV and why is it needed?**

Group-id, Artifact-id, Version. It makes every component unique, so 2 versions of the same dependency can sit together in `~/.m2/repository` without clashing.

**5. What is the Maven lifecycle?**

A sequential process with phases that run one after another: clean → install dependencies → compile → test → package → install. If one phase fails, the next ones don't run.

**6. What does `mvn clean package` do?**

`clean` deletes the old build. `package` validates, installs dependencies, compiles, runs tests and packages. Result: `target/shipping.jar`.

**7. What is a `.jar` file?**

Java ARchive - one file that holds the compiled bytecode of the app. Run it with `java -jar app.jar`.

**8. JDK vs JRE?**

JRE only runs Java bytecode (lightweight). JDK = JRE + development tools like the `javac` compiler (heavy). Building needs the JDK; only running needs the JRE.

**9. Is Java compiled or interpreted? How is it different from Node.js?**

Java is compiled before it runs: source code → bytecode → binary language. Node.js has no compile step - it compiles at runtime.

**10. Why do we set `JAVA_HOME`?**

So Maven knows which Java to use and where it's installed. Saved in `/etc/profile.d/java.sh` so it stays after every login and reboot.

**11. Many Java versions are installed. How do you choose the default?**

`alternatives --config java` → pick the version number. Check with `java -version` and `mvn -v`.

**12. Why is Java popular in enterprises, and what's the downside?**

It's mature and can run many tasks at the same time (multi-threading). Downside: it uses more CPU and memory than Node.js, so the server needs more RAM.

**13. Spring Boot shipping service started, but `curl` fails right away. Why?**

Spring Boot takes 10-20 seconds to start. Watch `journalctl -u shipping -f` until you see `Started ShippingApplication`, then test.
