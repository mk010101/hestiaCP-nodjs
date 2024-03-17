# How to Use Node.js in HestiaCP

By default, HestiaCP does not include support for publishing applications developed under Node.js, but you can easily set up your server to work with Node.js applications. Follow these steps to publish an application using HestiaCP either under app.sock or by listening on port 3000.

### Installing Node.js:

`sudo apt install -y curl`

1. Download and import the Nodesource GPG key

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://deb.nodesource.com/gpgkey/nodesource-repo.gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/nodesource.gpg
```

2. Create deb repository

```bash
NODE_MAJOR=20
echo "deb [signed-by=/etc/apt/keyrings/nodesource.gpg] https://deb.nodesource.com/node_$NODE_MAJOR.x nodistro main" | sudo tee /etc/apt/sources.list.d/nodesource.list
```

3. Update and install

```bash
sudo apt-get update
sudo apt-get install nodejs -y
```

HestiaCP’s pre-installed image already comes with Node.js, but you’ll need to install npm to manage your Node.js applications. Execute the following command:

## There are 2 methods:

## Method 1

Creating templates for Nginx using port 3000:
To publish Node.js applications, we’ll use Nginx as a proxy to redirect requests arriving at port 80 to port 3000 where Node.js is running. Perform the following steps to create the necessary files and set the correct permissions:

```bash
touch /usr/local/hestia/data/templates/web/nginx/nodejs3000.sh
touch /usr/local/hestia/data/templates/web/nginx/nodejs3000.tpl
touch /usr/local/hestia/data/templates/web/nginx/nodejs3000.stpl

chmod 755 /usr/local/hestia/data/templates/web/nginx/nodejs3000.sh
chmod 755 /usr/local/hestia/data/templates/web/nginx/nodejs3000.tpl
chmod 755 /usr/local/hestia/data/templates/web/nginx/nodejs3000.stpl
```

Add the following code to the **nodejs3000.sh** file:

```bash
#!/bin/bash
user=$1
domain=$2
ip=$3
home=$4
docroot=$5

mkdir "$home/$user/web/$domain/nodeapp"
chown -R $user:$user "$home/$user/web/$domain/nodeapp"
rm "$home/$user/web/$domain/nodeapp/app.sock"
runuser -l $user -c "pm2 start $home/$user/web/$domain/nodeapp/app.js"
sleep 5
chmod 777 "$home/$user/web/$domain/nodeapp/app.sock"
```

Add the following code to the **nodejs3000.tpl** file:

```bash
server {
listen %ip%:%proxy_port%;
server_name %domain_idn% %alias_idn%;
error_log /var/log/%web_system%/domains/%domain%.error.log error;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location /error/ {
        alias %home%/%user%/web/%domain%/document_errors/;
    }

    location @fallback {
        proxy_pass http://127.0.0.1:3000:/$1;
    }

    location ~ /\.ht {
        return 404;
    }
    location ~ /\.svn/ {
        return 404;
    }
    location ~ /\.git/ {
        return 404;
    }
    location ~ /\.hg/ {
        return 404;
    }
    location ~ /\.bzr/ {
        return 404;
    }

    include %home%/%user%/conf/web/nginx.%domain%.conf*;

}
```

Add the following code to the **nodejs3000.stpl** file:

```bash
server {
listen %ip%:%proxy_port%;
server_name %domain_idn%;
return 301 https://%domain_idn%$request_uri;
}

server {
listen %ip%:%proxy_ssl_port% http2 ssl;
server_name %domain_idn%;
ssl_certificate %ssl_pem%;
ssl_certificate_key %ssl_key%;
error_log /var/log/%web_system%/domains/%domain%.error.log error;
gzip on;
gzip_min_length 1100;
gzip_buffers 4 32k;
gzip_types image/svg+xml svg svgz text/plain application/x-javascript text/xml text/css;
gzip_vary on;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    location /error/ {
        alias %home%/%user%/web/%domain%/document_errors/;
    }

    location @fallback {
        proxy_pass https://127.0.0.1:3000:/$1;
    }

    location ~ /\.ht {
        return 404;
    }
    location ~ /\.svn/ {
        return 404;
    }
    location ~ /\.git/ {
        return 404;
    }
    location ~ /\.hg/ {
        return 404;
    }
    location ~ /\.bzr/ {
        return 404;
    }

    include %home%/%user%/conf/web/s%proxy_system%.%domain%.conf*;

}
```

## Method2

Creating templates for Nginx using the app.sock socket:
Alternatively, you can redirect requests arriving at port 80 to Node.js using a socket named app.sock. Follow these steps to create the necessary files and set the correct permissions:

```bash
touch /usr/local/hestia/data/templates/web/nginx/nodejssock.sh
touch /usr/local/hestia/data/templates/web/nginx/nodejssock.tpl
touch /usr/local/hestia/data/templates/web/nginx/nodejssock.stpl

chmod 755 /usr/local/hestia/data/templates/web/nginx/nodejssock.sh
chmod 755 /usr/local/hestia/data/templates/web/nginx/nodejssock.tpl
chmod 755 /usr/local/hestia/data/templates/web/nginx/nodejssock.stpl
```

Add the following code to the **nodejssock.sh** file:

```bash
#!/bin/bash
user=$1
domain=$2
ip=$3
home=$4
docroot=$5

mkdir "$home/$user/web/$domain/nodeapp"
chown -R $user:$user "$home/$user/web/$domain/nodeapp"
rm "$home/$user/web/$domain/nodeapp/app.sock"
runuser -l $user -c "pm2 start $home/$user/web/$domain/nodeapp/app.js"
sleep 5
chmod 777 "$home/$user/web/$domain/nodeapp/app.sock"

Add the following code to the nodejssock.tpl file:

server {
listen %ip%:%proxy_port%;
server_name %domain_idn% %alias_idn%;
error_log /var/log/%web_system%/domains/%domain%.error.log error;

    location / {
        proxy_pass http://unix:%home%/%user%/web/%domain%/nodeapp/app.sock:$request_uri;
        location ~* ^.+\.(%proxy_extentions%)$ {
            root %docroot%;
            access_log /var/log/%web_system%/domains/%domain%.log combined;
            access_log /var/log/%web_system%/domains/%domain%.bytes bytes;
            expires max;
            try_files $uri @fallback;
        }
    }

    location /error/ {
        alias %home%/%user%/web/%domain%/document_errors/;
    }



    location @fallback {
        proxy_pass http://unix:%home%/%user%/web/%domain%/nodeapp/app.sock:/$1;
    }

    location ~ /\.ht {
        return 404;
    }
    location ~ /\.svn/ {
        return 404;
    }
    location ~ /\.git/ {
        return 404;
    }
    location ~ /\.hg/ {
        return 404;
    }
    location ~ /\.bzr/ {
        return 404;
    }

    include %home%/%user%/conf/web/nginx.%domain%.conf*;

}
```

Add the following code to the **nodejssock.stpl** file:

```bash
server {
listen %ip%:%proxy_port%;
server_name %domain_idn%;
return 301 https://%domain_idn%$request_uri;
}

server {
listen %ip%:%proxy_ssl_port% http2 ssl;
server_name %domain_idn%;
ssl_certificate %ssl_pem%;
ssl_certificate_key %ssl_key%;
error_log /var/log/%web_system%/domains/%domain%.error.log error;
gzip on;
gzip_min_length 1100;
gzip_buffers 4 32k;
gzip_types image/svg+xml svg svgz text/plain application/x-javascript text/xml text/css;
gzip_vary on;

    location / {
        proxy_pass http://unix:%home%/%user%/web/%domain%/nodeapp/app.sock:$request_uri;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_set_header X-NginX-Proxy true;
        location ~* ^.+\.(%proxy_extentions%)$ {
            root %sdocroot%;
            access_log /var/log/%web_system%/domains/%domain%.log combined;
            access_log /var/log/%web_system%/domains/%domain%.bytes bytes;
            expires max;
            try_files $uri @fallback;
            add_header Pragma public;
            add_header Cache-Control "public";
        }
    }

    location /error/ {
        alias %home%/%user%/web/%domain%/document_errors/;
    }

    location @fallback {
        proxy_pass http://unix:%home%/%user%/web/%domain%/nodeapp/app.sock:/$1;
    }

    location ~ /\.ht {
        return 404;
    }
    location ~ /\.svn/ {
        return 404;
    }
    location ~ /\.git/ {
        return 404;
    }
    location ~ /\.hg/ {
        return 404;
    }
    location ~ /\.bzr/ {
        return 404;
    }

    include %home%/%user%/conf/web/s%proxy_system%.%domain%.conf*;

}
```

Using templates in the HestiaCP panel:
To connect to your HestiaCP panel, access https://host_name:8083 with the username “admin” and the password provided in the client panel.

Once inside the Hestia panel, go to **WEB** and click on the domain where you want to use the Node.js template.

In the “NGINX Proxy Support” section, click on the dropdown that shows “default” (the default option) and select either “nodejs3000” or “nodejssock” depending on your project’s requirements.

# pm2

pm2 is node app manager.
https://pm2.keymetrics.io/docs/usage/quick-start/

To start an app: `pm2 start myApp.js`

### Config

pm2.config.js:

```js
module.exports = {
  apps: [
    {
      name: "api 1",
      script: "web/api.my-server.tld/public_html/dist/app.js",
      watch: true,
      ignore_watch: ["node_modules"],
      //args: "limit",
    },
  ],
};
```

Start with config: `pm2 start pm2.config.js`

Watch mode: `pm2 start pm2.config.js --watch`

Auto start: `pm2 startup`

To disable and remove the current startup configuration: `pm2 unstartup`

# PostgreSQL

To login as admin: `sudo su - postgres`

Or

psql -U postgres

As user (when in `psql`): `-d db_name -U user_name`

# Change SSH port

`sudo nano /etc/ssh/sshd_config`

Uncomment #Port and change.
