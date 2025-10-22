#!/bin/bash
            exec > >(tee -a /var/log/user-data.log) 2>&1
            set -Eeuo pipefail

            dnf -y upgrade --refresh --allowerasing
            dnf -y install --allowerasing \
              nginx \
              php php-fpm php-mysqlnd php-json php-mbstring php-xml php-gd php-curl \
              amazon-efs-utils nfs-utils unzip tar rsync

            systemctl enable php-fpm nginx

            mkdir -p /var/www/html
            if ! grep -q " fs-094533122ca7d68cd:/ /var/www/html " /etc/fstab; then
              echo "fs-094533122ca7d68cd:/ /var/www/html efs _netdev,tls,accesspoint=fsap-0c2e2401cb2cc2fd5,noresvport 0 0" >> /etc/fstab
            fi
            for i in {1..6}; do mount -a && break || sleep 5; done
            echo ok >/var/www/html/health || true

            cat >/etc/nginx/conf.d/wordpress.conf <<'NGINX'
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

            sed -i 's/^user = .*/user = nginx/'  /etc/php-fpm.d/www.conf
            sed -i 's/^group = .*/group = nginx/' /etc/php-fpm.d/www.conf
            nginx -t
            systemctl restart php-fpm nginx
            
