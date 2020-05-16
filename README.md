# Nginx serwer z php-fpm
Nginx serwer z php-fpm wirtualnymi hostami i tls.

## Instalacja serwera i php
```bash
# Jako root
su -
# Aktualizacja
sudo apt update
sudo apt upgrade

# Zainstaluj serwer www
sudo apt install -y nginx

# Zainstaluj php (php7.3-fpm)
sudo apt install -y php-fpm php-mysql php-gd php-mbstring php-json php-curl php-mcrypt php-sqlite3 php-intl php-iconv php-xml
```

## Konfiguracja php-fpm
Tworzenie poola i grupy dla strony (odseparowanie php dla domen)

### Tworzenie grupy i użytkownika
```bash
# Użytkownik i grupa php dla domeny
groupadd user_domain_xx
useradd -g user_domain_xx user_domain_xx

# Katalog dla błędów:
mkdir /var/log/php-fpm

# Katalog strony
mkdir -p /var/www/domain.xx

# Zmień uprawnienia po dodaniu plików strony
chown -R www-data:www-data /var/www/domain.xx
chmod -R 775 /var/www/domain.xx
```

### Konfiguracja php-fpm dla domeny
nano /etc/php/7.3/fpm/pool.d/domain.xx
```bash
# Unikalna nazwa
[domain_xx]

; Grupa i użytkownika dla php
user = user_domain_xx
group = user_domain_xx

; Socket
; listen = 127.0.0.1:9000
; Lub unix socket (local only)
listen = /var/run/php/php7.3-fpm-domain-xx-site.sock

; Nginx user and group
listen.owner = www-data
listen.group = www-data

; Ustawienia
php_admin_value[disable_functions] = exec,passthru,shell_exec,system
php_admin_flag[allow_url_fopen] = off

; Błędy
php_flag[display_errors] = on
php_flag[display_startup_errors] = on
php_admin_flag[log_errors] = on
php_admin_value[error_log] = /var/log/fpm-php/domain.xx.log 

; Limit processów
pm = dynamic
pm.max_children = 10 # To będzie (512MB(ram) / 25-50MB(per page) => max. 20)
pm.start_servers = 3
pm.min_spare_servers = 2
pm.max_spare_servers = 4
pm.max_requests = 768
pm.process_idle_timeout = 10s

; Variables
env[HOSTNAME] = $HOSTNAME
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp

; Clients
; listen.allowed_clients = 127.0.0.1
; listen.backlog = -1

; Logi
; request_terminate_timeout = 60s
; request_slowlog_timeout = 10s
; slowlog = /var/log/php-fpm/slowlog-domain-xx.log

; Restart error
; emergency_restart_threshold 10
; emergency_restart_interval 1m
; process_control_timeout 10s
```

### Php-fpm rozmiar
```bash
ps -eo size,pid,user,command --sort -size | awk '{ hr=$1/1024 ; printf("%13.2f Mb ",hr) } { for ( x=4 ; x<=NF ; x++ ) { printf("%s ",$x) } print "" }' | grep php-fpm

pgrep -f php-fpm | xargs -r ps --no-headers -o "rss,cmd" | awk '{ sum+=$1 } END { printf ("%d%s\n", sum/NR/1024,"M") }'
```

### Tworzenie virtual hosta nginx
nano /etc/nginx/sites-available/domain.xx.conf
```bash
# Redirect all to https
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    server_name _;
    ; Dodaj przy generowaniu certyfikatów certbot
    ; root /var/www/domain.xx;
    ; Ukryj przy generowaniu certyfikatów certbot
    return 301 https://$host$request_uri; 
}

server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    
    server_name domain.xx www.domain.xx;
    root /var/www/domain.xx;
    
    ssl_certificate /etc/letsencrypt/live/domain.xx/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/domain.xx/privkey.pem;
    
    ssl_protocols TLSv1.3;
    ssl_prefer_server_ciphers off;
    
    ssl_session_timeout 1d;
    ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions
    ssl_session_tickets off;
    
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/domain.xx/fullchain.pem;
    
    add_header Strict-Transport-Security "max-age=63072000" always;      
    resolver 127.0.0.1;
    
    location / {
	try_files $uri $uri/ /index.php$is_args$args;
    }
    
    location ~ \.php$ {
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors on;
        
        include snippets/fastcgi-php.conf;
		    
        # Unix socket
        fastcgi_pass unix:/var/run/php/php7.3-fpm-domain-xx-site.sock
        
        # Or socket
		    # fastcgi_pass 127.0.0.1:9000;		    
    }
    
    # Error
    error_page 404 errors/404.html;
    access_log /var/log/domain.xx.access.log;
    error_log  /var/log/domain.xx-error.log error;

    # Statyczne pliki
    location ~* \.(html|css|js|jpg|jpeg|gif|png|pdf|txt|md|ico|xml)$ {
        access_log        off;
        log_not_found     off;
        expires           3d;
	add_header Cache-Control "public, no-transform";
    }
    
    location ~ /\.ht {
        access_log off;
        log_not_found off;
	deny  all;
    }
}
```

### Włączenie domeny
```bash
# Enable vhost
sudo ln -s /etc/nginx/sites-available/domain.xx /etc/nginx/sites-enabled/

# Restart nginx
sudo nginx -t
sudo service nginx restart

# Restart php-fpm
sudo service php7.3-fpm restart

# Usuwanie domeny
rm /etc/nginx/sites-enabled/domain.xx
```

### Opcjonalne ustawienia hosta
```conf
# Przekieruj favicon
location = /favicon.ico {
	rewrite . /favicon/favicon.ico;
	log_not_found off;
	access_log off;
}

# Nie loguj
location = /robots.txt {
	allow all;
	log_not_found off;
	access_log off;
}

# Dostęp i przekierowania
location / {
	# Get file or folder or error
	try_files $uri $uri/ =404;

	# Get file or folder or redirect uri to url param in index.php
	try_files $uri $uri/ /index.php?url=$uri&$args;

	# Wordpress
	try_files $uri $uri/ /index.php$is_args$args;
}

# Przeglądarka Cache-Controll
location ~* \.(html|css|js|jpg|jpeg|gif|png|pdf|txt|md|ico|xml)$ {
	access_log        off;
	log_not_found     off;
	expires           3d;
	add_header Cache-Control "public, no-transform";
}

# Ukryj pliki z kropką na początku
location ~ /\. {
	access_log off;
	log_not_found off;
	deny all;
}

# Zablokuj dostep do katalogów
location ~ /(dir1|dir2|dir3) {
	deny all;
	return 404;
}

# Wyłącz php w katalogu
location ~ /uploads/.*\.php$ {
	return 403;
}

# Wyłącz php w katalogach
location ~* /(?:uploads|media|files)/.*\.php$ {
	deny all;
}
```

### Certyfikaty tls
```bash
# Instalacja 
apt -y install certbot

# Generowanie certyfikatu
certbot certonly --standalone -d domain.xx -d www.domain.xx

# Lub
certbot certonly --webroot -w /var/www/html/domain.xx -d www.domain.xx -d domain.xx

# Odnowienie wygasających
certbot renew

# Wymuszenie odnowienia wszystkich
certbot renew --force-renew
```

### Mysql, mariadb
```bash
sudo apt install -y mysql-server
sudo mysql_secure_installation
sudo mysql -u root -p

CREATE DATABASE wordpress;
CREATE USER 'user'@'localhost' IDENTIFIED BY 'toor';
CREATE USER 'user'@'127.0.0.1' IDENTIFIED BY 'toor';
GRANT ALL PRIVILEGES ON wordpress.* TO 'user'@'localhost';
GRANT ALL PRIVILEGES ON wordpress.* TO 'user'@'127.0.0.1';
FLUSH PRIVILEGES;

# Allow create tables
# GRANT ALL PRIVILEGES ON wordpress.* TO 'user'@'localhost' WITH GRANT OPTION;
# GRANT ALL PRIVILEGES ON wordpress.* TO 'user'@'127.0.0.1' WITH GRANT OPTION;
```

#### Links
 - https://www.nginx.com/blog/installing-wordpress-with-nginx-unit/
 - https://www.journaldev.com/26097/php-fpm-nginx
 - https://www.if-not-true-then-false.com/2011/nginx-and-php-fpm-configuration-and-optimizing-tips-and-tricks
 - https://ssl-config.mozilla.org
 - https://www.nginx.com/resources/wiki/start/topics/recipes/wordpress
 - https://wordpress.org/support/article/nginx 
