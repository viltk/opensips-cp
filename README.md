# OpenSIPS Control Panel Installation

This guide installs **OpenSIPS Control Panel (OCP) 9.4.0** for managing and monitoring an **OpenSIPS 4.0** multinode deployment.

The recommended architecture uses a dedicated management server for OCP.

```text
                         Management Network
                               │
                    ┌──────────┴──────────┐
                    │ OpenSIPS Control    │
                    │ Panel (OCP)         │
                    │                     │
                    │ Apache + PHP        │
                    │ MariaDB             │ 
                    │ OCP 9.4.0           │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              │                │                │
        ┌─────┴─────┐    ┌─────┴─────┐    ┌─────┴─────┐
        │ Node 1    │    │ Node 2    │    │ Node 3    │
        │ OpenSIPS  │    │ OpenSIPS  │    │ OpenSIPS  │
        │ MI + Monit│    │ MI + Monit│    │ MI + Monit│
        └───────────┘    └───────────┘    └───────────┘
```

OCP is **not part of the SIP call path**. SIP-MESH must continue operating normally if the OCP server is unavailable.

---

## 1. Example Network

This guide uses the following example addresses:

| Host | Address | Purpose |
|---|---|---|
| `os-node1` | `192.168.1.2` | os-node1 |
| `os-node2` | `192.168.1.3` | os-node2 |
| `os-node3` | `192.168.1.4` | os-node3 |
| `os-node4` | `192.168.1.5` | OCP management server |

Replace these addresses with addresses appropriate for your environment.

---

# Part I — Install OCP

## 2. Install dependencies

Run on the OCP server:

```bash
sudo apt update
sudo apt upgrade -y
```

Install Apache, MariaDB, PHP and Git:

```bash
sudo apt install -y \
    apache2 \
    mariadb-server \
    git \
    php \
    libapache2-mod-php \
    php-cli \
    php-mysql \
    php-gd \
    php-curl \
    php-xml \
    php-mbstring \
    php-zip \
    php-apcu
```

Verify the services:

```bash
sudo systemctl status apache2 --no-pager
sudo systemctl status mariadb --no-pager
```

Check PHP:

```bash
php -v
```

---

## 3. Download OpenSIPS Control Panel

Clone OCP into the Apache web directory:

```bash
cd /var/www/html

sudo git clone https://github.com/OpenSIPS/opensips-cp.git
```

Change ownership while configuring the repository:

```bash
sudo chown -R $USER:$USER /var/www/html/opensips-cp
```

Enter the repository:

```bash
cd /var/www/html/opensips-cp
```

Switch to the OCP 9.4.0 branch:

```bash
git checkout 9.4.0
```

Verify:

```bash
git status
```

Expected:

```text
On branch 9.4.0
Your branch is up to date with 'origin/9.4.0'
```

---

## 4. Create the OCP database

Open MariaDB:

```bash
sudo mariadb
```

Create the database:

```sql
CREATE DATABASE opensips
    CHARACTER SET utf8mb4
    COLLATE utf8mb4_unicode_ci;
```

Create a dedicated database user:

```sql
CREATE USER 'opensips'@'localhost'
IDENTIFIED BY 'CHANGE_THIS_DATABASE_PASSWORD';
```

Grant access:

```sql
GRANT ALL PRIVILEGES ON opensips.*
TO 'opensips'@'localhost';

FLUSH PRIVILEGES;

EXIT;
```

Do not use the example password in production.

---

## 5. Import the OCP database schema

Enter the OCP directory:

```bash
cd /var/www/html/opensips-cp
```

Import the supplied MySQL/MariaDB schema:

```bash
mysql -u opensips -p opensips < config/db_schema.mysql
```

Verify:

```bash
mysql -u opensips -p opensips -e "SHOW TABLES;"
```

OCP tables should now be listed.

---

## 6. Configure the OCP database connection

Edit:

```bash
nano /var/www/html/opensips-cp/config/db.inc.php
```

Configure:

```php
$config->db_driver = "mysql";
$config->db_host = "localhost";
$config->db_port = "";

$config->db_user = "opensips";
$config->db_pass = "CHANGE_THIS_DATABASE_PASSWORD";
$config->db_name = "opensips";
```

Check the PHP syntax:

```bash
php -l /var/www/html/opensips-cp/config/db.inc.php
```

Expected:

```text
No syntax errors detected
```

### Test PHP → MariaDB connectivity

Replace the password before running:

```bash
php -r '$db=new mysqli(
    "localhost",
    "opensips",
    "CHANGE_THIS_DATABASE_PASSWORD",
    "opensips"
); if($db->connect_error){
    die("FAILED: ".$db->connect_error.PHP_EOL);
} echo "OCP database connection OK".PHP_EOL;'
```

Expected:

```text
OCP database connection OK
```

---

## 7. Configure Apache

Create:

```bash
sudo nano /etc/apache2/conf-available/opensips-cp.conf
```

Add:

```apache
Alias /cp /var/www/html/opensips-cp/web

<Directory /var/www/html/opensips-cp/web>
    Options FollowSymLinks
    AllowOverride None
    Require all granted
</Directory>

<DirectoryMatch "/var/www/html/opensips-cp/web/tools/.*/.*/(template|custom_actions|lib)/">
    Require all denied
</DirectoryMatch>
```

Enable the configuration:

```bash
sudo a2enconf opensips-cp
```

Validate Apache:

```bash
sudo apache2ctl configtest
```

Expected:

```text
Syntax OK
```

Restart Apache:

```bash
sudo systemctl restart apache2
```

> **Important:** Apache must be restarted or reloaded after enabling the OCP configuration. Otherwise `/cp/` may return `404 Not Found`.

Verify:

```bash
sudo systemctl status apache2 --no-pager
```

---

## 8. Open OCP

Browse to:

```text
http://OCP_SERVER_IP/cp/
```

For example:

```text
http://192.168.1.2/cp/
```

If `/cp/` returns `404`, check:

```bash
ls -l /etc/apache2/conf-enabled/opensips-cp.conf

sudo apache2ctl configtest

sudo systemctl restart apache2
```

Test locally:

```bash
curl -I http://127.0.0.1/cp/
```

For Apache/PHP errors:

```bash
sudo tail -n 100 /var/log/apache2/error.log
```

---

# Part II — Configure OpenSIPS Nodes

OCP communicates with each OpenSIPS node using the OpenSIPS Management Interface (MI).

This example uses:

```text
TCP/8888 → OpenSIPS HTTP MI
```

---

## 9. Install OpenSIPS HTTP modules

Run on **every SIP-MESH node**:

```bash
sudo apt update
sudo apt install -y opensips-http-modules
```

Verify:

```bash
dpkg -l | grep opensips-http
```

Check the modules:

```bash
find /usr/lib -name 'httpd.so' -o -name 'mi_http.so'
```

Both modules should be present.

---

## 10. Enable the OpenSIPS HTTP MI

Add the following to opensips configuration file:
```cfg
####### OCP / Management Interface ########

loadmodule "httpd.so"
modparam("httpd", "ip", "REPLACE_NODE_IP")
modparam("httpd", "port", 8888)

loadmodule "mi_http.so"
modparam("mi_http", "root", "mi")
```

Default configuration is located in
```text
/etc/opensips/opensips.cfg
```

I'm using separate configuration files in location 
```text
/etc/opensips/conf.d/
```


Validate OpenSIPS before restarting:

```bash
sudo opensips -C -f /etc/opensips/opensips.cfg
```

If validation succeeds:

```bash
sudo systemctl restart opensips
```

Verify:

```bash
sudo ss -lntp | grep 8888
```

---

## 11. Test MI connectivity from OCP

Run from the OCP server:

```bash
nc -vz 192.168.1.2 8888
nc -vz 192.168.1.3 8888
nc -vz 192.168.1.4 8888
```

All three connections should succeed.


> **Security:** Do not expose TCP/8888 to untrusted networks. Restrict access to the OCP management host or management network.

---

# Part III — Install Monit

OCP can also use Monit to monitor the operating system and OpenSIPS process.

Monit is separate from the OpenSIPS MI interface:

```text
OCP
 │
 ├── TCP/8888 → OpenSIPS MI
 │
 └── TCP/2812 → Monit
```

---

## 12. Install Monit

On each OpenSIPS node:

```bash
sudo apt update
sudo apt install -y monit
```

---

## 13. Configure Monit

Create:

```bash
sudo nano /etc/monit/conf.d/opensips-ocp
```

Example for Node:

```text
set httpd port 2812
    use address <LOCAL_NODE_IP>
    allow <LOCAL_NETWORK_IP_AND_PREFIX>
    allow <OCP_SERVER_IP> (if server is not in same subnet)
    allow ocpadmin:"CHANGE_THIS_MONIT_PASSWORD"

check process opensips
    matching "/usr/sbin/opensips"
```

For the example environment:

```text
set httpd port 2812
    use address 192.168.1.2
    allow 192.168.1.1/24
    allow 172.168.1.5  # opensips-cp IP address
    allow ocpadmin:"MyWeryStrongPassword.123!"

check process opensips
    matching "/usr/sbin/opensips"
```


**Security:** Use a strong password and keep it to yourself!

---

## 14. Validate Monit

Check:

```bash
sudo monit -t
```

Expected:

```text
Control file syntax OK
```

Restart and enable:

```bash
sudo systemctl restart monit
sudo systemctl enable monit
```

Check:

```bash
sudo systemctl status monit --no-pager
```

Verify TCP/2812:

```bash
sudo ss -lntp | grep 2812
```

---

## 15. Test Monit

Test locally:

```bash
curl -v \
    -u 'ocpadmin:CHANGE_THIS_MONIT_PASSWORD' \
    http://192.168.1.2:2812/
```

Then test from the OCP server:

```bash
curl -v \
    -u 'ocpadmin:CHANGE_THIS_MONIT_PASSWORD' \
    http://192.168.1.2:2812/
```

A successful request should return the Monit web interface.

Check connectivity to all nodes:

```bash
nc -vz 192.168.1.2 2812
nc -vz 192.168.1.3 2812
nc -vz 192.168.1.4 2812
```

---

# Part IV — Add SIP-MESH Nodes to OCP

## 16. Add first node

In OCP, create a system with name you like:

```text
My-Awesome-System
```

Then add the first box:

```text
Box Name:
First -BOX

MI connector:
json:192.168.1.2:8888/mi

Monit connector:
192.168.1.2:2812

Monit username:
ocpadmin

Password:
<MONIT_PASSWORD>

Confirm Password:
<MONIT_PASSWORD>

Monit SSL:
Disabled

System Monitor charting:
Enabled

System name:
My-Awesome-System


Description:
My-Awesome-System BOX description you like.
```

---

# Security Recommendations

OCP is an administrative interface and should not be exposed directly to the public Internet.

```

At minimum:

- restrict OCP access to the network;
- restrict OpenSIPS TCP/8888 to the OCP server;
- restrict Monit TCP/2812 to the OCP server;
- use unique strong passwords;
- do not commit passwords to Git;
- use HTTPS for OCP in production;
- consider TLS for management interfaces;
- keep OCP outside the SIP signaling path.

---

# Troubleshooting

## `httpd.so` or `mi_http.so` not found

Example:

```text
failed to load module 'httpd.so' - not found
```

Install:

```bash
sudo apt install opensips-http-modules
```

---

## `/cp/` returns 404

Make sure the Apache configuration is enabled:

```bash
sudo a2enconf opensips-cp
sudo apache2ctl configtest
sudo systemctl restart apache2
```

Then:

```bash
curl -I http://127.0.0.1/cp/
```

---

## MariaDB access denied

Test directly:

```bash
mysql -u opensips -p opensips
```

If necessary, reset the password:

```bash
sudo mariadb
```

```sql
ALTER USER 'opensips'@'localhost'
IDENTIFIED BY 'NEW_DATABASE_PASSWORD';

FLUSH PRIVILEGES;
```

Update the same password in:

```text
/var/www/html/opensips-cp/config/db.inc.php
```

---

## Monit `Connection reset by peer`

Check access rules:

```bash
sudo grep -R -A10 -B2 "set httpd" /etc/monit/
```

Validate:

```bash
sudo monit -t
```

Check the listener:

```bash
sudo ss -lntp | grep 2812
```

Check logs:

```bash
sudo journalctl -u monit -n 50 --no-pager
```

Remember that Monit may reject a client whose source IP is not permitted by an `allow` rule.

---

# Final Architecture

After installation:

```text
                       OCP Management Server
                    Apache + PHP + MariaDB
                              │
              ┌───────────────┼───────────────┐
              │               │               │
              │               │               │
         OpenSIPS 4.0    OpenSIPS 4.0    OpenSIPS 4.0
              │               │               │
        MI :8888         MI :8888         MI :8888
        Monit :2812      Monit :2812      Monit :2812
              │               │               │
              └──────── Monitored-nodes ──────┘
```

OCP provides centralized **management and observability**, but it is not a dependency of the distributed SIP-MESH call-routing architecture.