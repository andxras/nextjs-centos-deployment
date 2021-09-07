## Despliegue de [Nextjs](https://nextjs.org/) en un servidor CentOS

instrucciones directas para el despliegue de nextjs (o aplicaciones similares) en un servidor centos8.

![diseño](https://i.ibb.co/5TK3wmR/preview.png)

## Setup

### configurar acceso y algunas medidas de seguridad

#### I — instalar dependencias necesarias

```sh
dnf install git vim curl wget net-tools nginx firewalld
dnf module enable nodejs:14 -y # LSV
dnf module install nodejs:14 -y
```

#### II — habilitar actualizaciones automáticas del sistema

```sh
dnf update -y
dnf install epel-release
dnf upgrade
dnf install snapd -y
systemctl enable --now snapd.socket
ln -s /var/lib/snapd/snap /snap
dnf install dnf-automatic -y
systemctl enable --now dnf-automatic.timer
systemctl list-timers

```

#### III — crear sudo user y configurar ssh

```sh
# remote >>
adduser andxras
passwd andxras # set a new password for this user
usermod -aG wheel andxras # add username to wheel group (sudo access)
su andxras
mkdir ~/.ssh
chmod 700 ~/.ssh # rwx for current user
touch ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys # rw for current user
```

```sh
# host >>
ssh-keygen -t rsa -b 4096 -f ~/.ssh/mykey -C "nobody@domain.tld"
ssh-copy-id -i ~/.ssh/mykey andxras@host # or manually paste your pub key into ~/.ssh/authorized_keys
ssh -i ~/.ssh/mykey andxras@<remote> # test conn
```

##### editar /etc/ssh/sshd_config

```sh
PermitRootLogin no
PasswordAuthentication no
Port 33420
```

```sh
# remote >>
userdel -r centos
sshd -t
sudo semanage port -a -t ssh_port_t -p tcp 33420 # for selinux to allow ssh on this port
systemctl reload sshd
```

##### IV — configurar fail2ban para ssh

##### archivo de configuración [jail-sshd.conf](https://github.com/andxras/deployment-nextjs-centos/blob/main/jail-sshd.conf)

```sh
sudo dnf install fail2ban
sudo systemctl start fail2ban
sudo systemctl enable fail2ban
sudo curl https://raw.githubusercontent.com/andxras/deployment-nextjs-centos/main/jail-sshd.conf > /etc/fail2ban/jail.d/jail-sshd.conf
sudo systemctl restart fail2ban
```

#### V — configurar firewall usando firewalld

```
sudo systemctl start firewalld
sudo firewall-cmd --add-service=http --zone=public --permanent
sudo firewall-cmd --add-service=https --zone=public --permanent
sudo firewall-cmd --add-port=33420/tcp --zone=public --permanent # sshd running on this port
sudo firewall-cmd --reload
sudo firewall-cmd --list-all # check if rules are correct
sudo systemctl enable firewalld
```

### reverse proxy, load balancing y módulo gzip
    
#### I — descargar código y configurar PM2

##### para clonar un repo privado

```sh
# /var/app
ssh-keygen -t ed25519 -C "nobody@domain.tld" -f ~/.ssh/foo
# then manually insert .pub key on the github account in settings > SSH and GPG keys > SSH keys
ssh-agent bash -c 'ssh-add ~/.ssh/foo; git clone git@github.com:username/repo_name.git'
```

```sh
  # /var/app
  git clone https://github.com/username/repo_name.git
  sudo npm install -g pm2
  npm run build
  PORT=4100 pm2 start npm --name "replicante-4100" -- start
  PORT=4200 pm2 start npm --name "replicante-4200" -- start
  pm2 save
  pm2 startup systemd       # this generates and configures a startup script
                            # to launch PM2 and its managed processes on server boot
                            
  ausearch -c 'systemd' --raw | audit2allow -M my-systemd # run this command (as root) and follow instructions to solve pm2
                                                          # issues on centos environments due to selinux

  sudo systemctl start pm2-andxras # we should get the same error
  ausearch -c 'systemd' --raw | audit2allow -M my-systemd # execute again to fix issues
  sudo systemctl start pm2-andxras # now it should be working
```
  
#### II — configurar reverse proxy

```sh
sudo systemctl start nginx
sudo systemctl enable nginx
```

##### Crear un archivo de configuración en /etc/nginx/conf.d/*.conf (siendo root)

```
# e.g. domain.tld.conf

server {
    listen 80;
    listen [::]:80;
    server_name domain.tld www.domain.tld; # e.g. google.com www.google.com
    
    proxy_set_header Host $http_host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    
    location / {
        proxy_pass http://localhost:4100;
    }
    
    # static content for nextjs
    location /_next/static/ {
        alias /var/app/repo_name/.next/static/;
    }
}
```

##### Reinicia el servidor de nginx

```sh
  sudo nginx -t                 # check any syntax errors
  sudo systemctl reload nginx  # restart nginx
  sudo setsebool -P httpd_can_network_connect on  # enable http which is disabled by default on selinux
```

#### III — habilitar load balancing en nginx con gzip y soporte para websockets

##### Modifica el archivo de configuración en /etc/nginx/conf.d/domain.tld.conf (siendo root)

```

upstream domain {
    ip_hash;
    server localhost:4100;
    server localhost:4200;
}

server {

    server_name domain.tld www.domain.tld;
    
    gzip on;
    gzip_disable "msie6";

    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_buffers 16 8k;
    gzip_http_version 1.1;
    gzip_min_length 256;
    gzip_types
      application/javascript
      application/json
      font/eot
      font/otf
      font/ttf
      image/svg+xml
      text/css
      text/javascript
      text/plain
      text/xml;

    ...
    
    location / {
        proxy_pass http://domain;
    }
    
    location /_next/static/ {
        alias /var/app/repo_name/.next/static/;
    }
    
    # websockets support (uncomment only when your application uses websockets)
    # location /socket.io {
    #     proxy_http_version 1.1;
    #     proxy_set_header Upgrade $http_grade;
    #     proxy_set_header Connection "upgrade";
    # 
    #     proxy_pass http://domain/socket.io;
    # }

    ...
}

```

#### IV — ocultar la versión del servidor de nginx

##### Editar el archivo /etc/nginx/nginx.conf (siendo root)

```
http {
    ...
    server_tokens off;
    ...
}
```

##### Reinicia el servidor de nginx y verifica que no se muestre la versión

```sh
nginx -t # check any syntax errors
systemctl reload nginx
curl --head domain.tld
```

  
### certificado SSL con let's encrypt & certbot (auto renovable)

https://certbot.eff.org/lets-encrypt/centosrhel8-nginx

```sh
sudo snap install core
sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo certbot --nginx -d domain.tld,www.domain.tld -n -m <myemail> --agree-tos # requests a new ssl certificate
sudo certbot renew --dry-run # test automatic renewal
```



