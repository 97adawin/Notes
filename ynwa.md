# Admin EC2 — Manual Provisioning Guide (Amazon Linux 2023)

This guide shows exactly how to set up the **Admin EC2** *manually* (no CloudFormation/user‑data) to prepare WordPress on **EFS** and connect it to **RDS** — matching the build you have running.

> Works on **Amazon Linux 2023**. Run everything in an **SSM Session Manager** shell (recommended) or SSH.

---

## 0) Prerequisites / Security Groups

Before you start, confirm these rules exist:

- **EFSSG**: inbound **TCP 2049** from **AdminSG** (and WebSG for later).
- **RDSSG**: inbound **TCP 3306** from **AdminSG** (and WebSG for later).
- EFS has **mount targets** in both of your public subnets (state **available**).

---

## 1) Set your variables (paste and edit)

```bash
# ---- Fill these with your real values ----
EFS_ID="fs-xxxxxxxxxxxxxxxxx"
EFS_AP="fsap-xxxxxxxxxxxxxxxxx"
DB_ENDPOINT="your-db.xxxxxx.eu-west-1.rds.amazonaws.com"
DB_NAME="wordpress"
DB_USER="wpuser"
DB_PASS="your-strong-password"
SITE_URL="http://placeholder.local"   # change to ALB URL later
SITE_TITLE="Viking Mead"
WP_ADMIN_USER="wpadmin"
WP_ADMIN_PASS="1337muchstronger1929"
WP_ADMIN_EMAIL="jasonburne@example.com"
# ------------------------------------------
```

> Tip: If you’ll store `DB_PASS` in SSM Parameter Store later, see §8.

---

## 2) Install packages (Nginx, PHP, EFS tools)

Amazon Linux 2023 ships with `curl-minimal`. Use `--allowerasing` to avoid conflicts.

```bash
sudo dnf -y upgrade --refresh --allowerasing
sudo dnf -y install --allowerasing \
  nginx \
  php php-fpm php-mysqlnd php-json php-mbstring php-xml php-gd php-curl \
  amazon-efs-utils nfs-utils unzip tar rsync
sudo systemctl enable php-fpm nginx
sudo systemctl start  php-fpm nginx
```

---

## 3) Mount EFS at `/var/www/html`

```bash
sudo mkdir -p /var/www/html
# add fstab entry (with access point)
if ! grep -q " ${EFS_ID}:/ /var/www/html " /etc/fstab; then
  echo "${EFS_ID}:/ /var/www/html efs _netdev,tls,accesspoint=${EFS_AP},noresvport 0 0" | sudo tee -a /etc/fstab
fi
# attempt mount a few times
for i in {1..6}; do sudo mount -a && break || sleep 5; done

# health file on EFS
echo ok | sudo tee /var/www/html/health >/dev/null || true
```

**Check**

```bash
mount | grep efs
sudo tail -n 50 /var/log/amazon/efs/mount.log
```

---

## 4) Configure Nginx + PHP-FPM (socket)

On AL2023, php-fpm listens on a **UNIX socket** (`/run/php-fpm/www.sock`).

```bash
# site config
sudo tee /etc/nginx/conf.d/wordpress.conf >/dev/null <<'NGINX'
server {
  listen 80 default_server;
  server_name _;
  root /var/www/html;

  location = /health { return 200 'ok'; add_header Content-Type text/plain; }

  index index.php index.html;
  location / { try_files $uri $uri/ /index.php?$args; }
  location ~ \.php$ {
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass unix:/run/php-fpm/www.sock;
  }
}
NGINX

# run php-fpm as nginx user (matches file ownership later)
sudo sed -i 's/^user = .*/user = nginx/'  /etc/php-fpm.d/www.conf
sudo sed -i 's/^group = .*/group = nginx/' /etc/php-fpm.d/www.conf

sudo nginx -t && sudo systemctl restart php-fpm nginx
```

---

## 5) Put WordPress files on EFS

```bash
if [ ! -f /var/www/html/wp-settings.php ]; then
  cd /tmp
  curl -fsSL -o wp.tgz https://wordpress.org/latest.tar.gz
  tar xzf wp.tgz
  sudo rsync -a wordpress/ /var/www/html/
fi

# permissions (EFS AP may override; ignore failures)
sudo chown -R nginx:nginx /var/www/html || true
sudo find /var/www/html -type d -exec chmod 775 {} \; || true
sudo find /var/www/html -type f -exec chmod 664 {} \; || true
```

**Check**

```bash
test -f /var/www/html/wp-settings.php && echo "WP files present"
curl -s localhost/health
```

---

## 6) Install WP-CLI and DB client, create DB

```bash
# wp-cli
if ! command -v wp >/dev/null 2>&1; then
  sudo curl -fsSL -o /usr/local/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar
  sudo chmod +x /usr/local/bin/wp
fi

# MySQL client + ensure DB exists
sudo dnf -y install mariadb105 || sudo dnf -y install mariadb
mysql -h "$DB_ENDPOINT" -u "$DB_USER" -p"$DB_PASS" \
  -e "CREATE DATABASE IF NOT EXISTS \`$DB_NAME\` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"
```

---

## 7) Create `wp-config.php` and run the install (once)

```bash
# create wp-config (idempotent)
sudo -u nginx wp config create \
  --path=/var/www/html \
  --dbname="$DB_NAME" --dbuser="$DB_USER" --dbpass="$DB_PASS" --dbhost="$DB_ENDPOINT" \
  --skip-check --force

# useful constants
sudo -u nginx wp config set WP_HOME    "$SITE_URL" --type=constant --path=/var/www/html
sudo -u nginx wp config set WP_SITEURL "$SITE_URL" --type=constant --path=/var/www/html
sudo -u nginx wp config set FS_METHOD  "direct"    --type=constant --path=/var/www/html

# install WordPress if not installed
sudo -u nginx wp core is-installed --path=/var/www/html || sudo -u nginx wp core install \
  --url="$SITE_URL" --title="$SITE_TITLE" \
  --admin_user="$WP_ADMIN_USER" --admin_password="$WP_ADMIN_PASS" --admin_email="$WP_ADMIN_EMAIL" \
  --skip-email --path=/var/www/html

# verify DB connection through wp-config
sudo -u nginx wp db check --path=/var/www/html
```

> If `wp config create` complains about permissions due to EFS AP, re-run as root with `--allow-root`, then chown the file:  
> `sudo wp config create --allow-root ... ; sudo chown nginx:nginx /var/www/html/wp-config.php || true`

---

## 8) (Optional) Pull DB password from SSM Parameter Store

Create the parameter from your laptop/CLI:

```bash
aws ssm put-parameter \
  --name "/wp-school/db-password" \
  --type SecureString \
  --value "your-strong-password" \
  --overwrite --region eu-west-1
```

Instance role must have `ssm:GetParameter` access. Then on the admin box:

```bash
sudo dnf -y install awscli || true
DB_PASS="$(aws ssm get-parameter --name '/wp-school/db-password' --with-decryption --region eu-west-1 --query Parameter.Value --output text)"
```

Use that `DB_PASS` variable for the steps in §6–§7.

---

## 9) After ALB + ASG are online — set the public URL

```bash
# From any instance (they all share EFS)
ALB_DNS="http://<your-alb-dns>"
sudo -u nginx wp config set WP_HOME    "$ALB_DNS" --type=constant --path=/var/www/html
sudo -u nginx wp config set WP_SITEURL "$ALB_DNS" --type=constant --path=/var/www/html
sudo -u nginx wp option update home    "$ALB_DNS" --path=/var/www/html
sudo -u nginx wp option update siteurl "$ALB_DNS" --path=/var/www/html

# pretty permalinks
sudo -u nginx wp rewrite structure '/%postname%/' --hard --path=/var/www/html
sudo -u nginx wp rewrite flush --hard --path=/var/www/html
```

---

## 10) Quick verification & smoke tests

```bash
# services
systemctl is-active nginx php-fpm

# EFS
mount | grep efs

# WordPress status
sudo -u nginx wp core is-installed --path=/var/www/html && echo "WP installed"
sudo -u nginx wp db check --path=/var/www/html

# content + uploads
sudo -u nginx wp post create --post_title="Skål from Admin" --post_status=publish --path=/var/www/html
echo hello | sudo tee /var/www/html/wp-content/uploads/probe.txt >/dev/null
```

Fetch via browser: `http://<ALB-DNS>/wp-content/uploads/probe.txt` → should print `hello`.

---

## 11) Troubleshooting Cheatsheet

- **502 from ALB** → ensure Nginx uses PHP-FPM **socket**: `fastcgi_pass unix:/run/php-fpm/www.sock;`
- **EFS mount fails** → check `/var/log/amazon/efs/mount.log`, SG 2049 from AdminSG, mount targets in both subnets.
- **DB connect fails** → SG 3306 from AdminSG (and WebSG), verify `timeout 5 bash -lc ":</dev/tcp/$DB_ENDPOINT/3306"`
- **User-data didn’t run** → Only runs on first boot. For manual method (this guide), you don’t need user-data.

---

## 12) All-in-one script (optional)

If you prefer to paste once, use this idempotent script **after filling variables in §1**:

```bash
sudo bash -lc '
set -Eeuo pipefail

dnf -y upgrade --refresh --allowerasing
dnf -y install --allowerasing \
  nginx php php-fpm php-mysqlnd php-json php-mbstring php-xml php-gd php-curl \
  amazon-efs-utils nfs-utils unzip tar rsync

systemctl enable php-fpm nginx
systemctl start  php-fpm nginx

mkdir -p /var/www/html
grep -q " ${EFS_ID}:/ /var/www/html " /etc/fstab || echo "${EFS_ID}:/ /var/www/html efs _netdev,tls,accesspoint=${EFS_AP},noresvport 0 0" >> /etc/fstab
for i in {1..6}; do mount -a && break || sleep 5; done
echo ok >/var/www/html/health || true

cat >/etc/nginx/conf.d/wordpress.conf <<'"'"'NGINX'"'"'
server {
  listen 80 default_server;
  server_name _;
  root /var/www/html;
  location = /health { return 200 '"'"'ok'"'"'; add_header Content-Type text/plain; }
  index index.php index.html;
  location / { try_files $uri $uri/ /index.php?$args; }
  location ~ \.php$ {
    include fastcgi_params;
    fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_pass unix:/run/php-fpm/www.sock;
  }
}
NGINX

sed -i "s/^user = .*/user = nginx/"  /etc/php-fpm.d/www.conf
sed -i "s/^group = .*/group = nginx/" /etc/php-fpm.d/www.conf
nginx -t && systemctl restart php-fpm nginx

# WordPress files
if [ ! -f /var/www/html/wp-settings.php ]; then
  cd /tmp && curl -fsSL -o wp.tgz https://wordpress.org/latest.tar.gz
  tar xzf wp.tgz && rsync -a wordpress/ /var/www/html/
fi

# wp-cli + DB client + ensure DB
[ -x /usr/local/bin/wp ] || { curl -fsSL -o /usr/local/bin/wp https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar && chmod +x /usr/local/bin/wp; }
dnf -y install mariadb105 || dnf -y install mariadb
mysql -h "$DB_ENDPOINT" -u "$DB_USER" -p"$DB_PASS" -e "CREATE DATABASE IF NOT EXISTS \`$DB_NAME\` DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;"

# wp-config + install
sudo -u nginx wp config create \
  --path=/var/www/html \
  --dbname="$DB_NAME" --dbuser="$DB_USER" --dbpass="$DB_PASS" --dbhost="$DB_ENDPOINT" \
  --skip-check --force
sudo -u nginx wp config set WP_HOME    "$SITE_URL" --type=constant --path=/var/www/html
sudo -u nginx wp config set WP_SITEURL "$SITE_URL" --type=constant --path=/var/www/html
sudo -u nginx wp config set FS_METHOD  "direct"    --type=constant --path=/var/www/html
sudo -u nginx wp core is-installed --path=/var/www/html || sudo -u nginx wp core install \
  --url="$SITE_URL" --title="$SITE_TITLE" \
  --admin_user="$WP_ADMIN_USER" --admin_password="$WP_ADMIN_PASS" --admin_email="$WP_ADMIN_EMAIL" \
  --skip-email --path=/var/www/html

echo "Admin EC2 manual provisioning complete."
'
```

---

**You’re done.** This Admin EC2 can now be stopped/terminated; the web ASG will serve WordPress from **RDS + EFS**.
