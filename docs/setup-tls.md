# Setup TLS

> [!NOTE]
> The configurations for setup TLS are removed due to the fact that the TLS is
> not required for user who want to test the `flexisip` server.


> [!IMPORTANT]
> Currently this demo only setup TLS for the `flexisip-account-manager` server.
> For the user to register the account with the softphone clients using TLS, you
> have to configure the `flexisip.conf` server with the TLS.

The demo is configured with the self-signed certificate for the `nginx` server
using [`mkcert`](https://github.com/FiloSottile/mkcert).

1. Install `mkcert`:

```bash
sudo apt install libnss3-tools
```

2. Configure `mkcert`:

```bash
mkcert -install
```

3. Generate the certificate:

```bash
mkcert -cert-file ./ubuntu23-10/letsencrypt/live/192.168.1.17.nip.io/fullchain.pem -key-file ./ubuntu23-10/letsencrypt/live/192.168.1.17.nip.io/privkey.pem "*.192.168.1.17.nip.io";
```

This command will generate the certificate for the wildcard domain
`*.192.168.1.17.nip.io`.

4. Update the `nginx` configuration file:

```conf
resolver 127.0.0.11 valid=15s;

server {
  listen 80;
  listen [::]:80;

  server_name *.192.168.1.17.nip.io;

  location /.well-known/acme-challenge/ {
    root /var/www/certbot;
  }

  location / {
    return 301 https://$host$request_uri;
  }
}

server {
  listen 443 ssl;
  listen [::]:443 ssl http2;

  server_name *.192.168.1.17.nip.io;
  server_tokens off;

  ssl_certificate /etc/letsencrypt/live/192.168.1.17.nip.io/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/192.168.1.17.nip.io/privkey.pem;

  index index.php index.html;
  root /var/www/html/flexiapi/public;

  set $upstream phpmyadmin-fpm:9000;
  set $upstream_host php-fpm-laravel:9000;

  location ~ \.php$ {
    try_files      $uri = 404;
    fastcgi_split_path_info ^(.+\.php)(/.+)$;
    fastcgi_pass   $upstream_host;
    fastcgi_index index.php;
    include        fastcgi_params;
    fastcgi_param  SCRIPT_FILENAME $document_root$fastcgi_script_name;
    fastcgi_param PATH_INFO $fastcgi_path_info;
  }

  location / {
    try_files $uri $uri/ /index.php?$query_string;
    gzip_static on;
  }

  location ^~ /phpmyadmin {
    alias /var/www/html/phpmyadmin;
    index index.php;
    location ~ \.php$ {
      try_files      $uri = 404;
      include        fastcgi_params;
      fastcgi_split_path_info ^\/phpmyadmin\/(.+\.php)(.*)$;
      fastcgi_param  SCRIPT_FILENAME $fastcgi_script_name;
      fastcgi_pass   $upstream;
    }
  }
}
```
5. Update `flexisip.conf`:

> [!NOTE]
> This configuration is not tested yet.


```conf
[global]
aliases=192.168.1.17.nip.io
transports=sip:* sips:*
tls-certificates-file=/etc/letsencrypt/live/192.168.1.17.nip.io/fullchain.pem
tls-certificates-private-key=/etc/letsencrypt/live/192.168.1.17.nip.io/privkey.pem
[module::Authentication]
enabled=true
auth-domains=account.192.168.1.17.nip.io
available-algorithms=SHA-256
db-implementation=soci
soci-backend=mysql
soci-connection-string=db=flexisip user=mysql password=mysql host=db.192.168.1.17.nip.io
soci-password-request=select password, algorithm from passwords join accounts on passwords.id = accounts.id where accounts.username = :id and accounts.domain = :domain
[module::Registrar]
reg-domains=account.192.168.1.17.nip.io
```
