# How to deploy Next.js Application on Hostinger VPS

# Intro

hello and welcome to my coding tutorial to build and deploy my nextjs e-commerce website on Linux Ubuntu operating system using Hostinger VPS server.

In this tutorial we will learn how to:

- buy and configure VPS server and website domain on hostinger
- clone my main amazona project from GitHub
- connect to the hostinger Linux server
- build and deploy automatically applications on every git push
- and at the end I teach you how to encrypt the website using let's encrypt SSL certificate

## Hostinger VPS

Let us start by buying hostinger VPS and domain. first of all I'm going to thank hostinger for sponsoring this video. I already used their service for hosting my website and web applications like codingwithbasir.com.
In this tutorial we are going to build and deploy a full-functional ecommerce website on hostinger VSP server.

Lets see benefits of using VPS services with Hostinger

- High performance
- KVM virtualization
- Full root access
- Scalability
- Enhanced security
- One-click applications
- VPS AI assistant
- Free automatic backups and real-time snapshots

## Buy VPS Server

- select vps kvm 2
- apply coupon code
- make payment

## Buy Domain

- buy a domain
- add A, AA record for vps ip address

## Deploy Application

I suppose the vps `ip address` is `5.182.18.65` and `domain name` is `nextjs-ecommerce.com`. you need to change them based on your vps ip address and domain name.

1. connect to vps server

   ```shell
    ssh root@5.182.18.65
   ```

2. install node

   ```shell
   apt-get update
   curl -fsSL https://deb.nodesource.com/setup_20.x -o nodesource_setup.sh
   sudo -E bash nodesource_setup.sh
   sudo apt-get install -y nodejs
   node -v

   ```

3. create git repo

   ```shell
    mkdir -p ~/apps/nextjs-ecommerce/repo
    mkdir -p ~/apps/nextjs-ecommerce/dest
    cd ~/apps/nextjs-ecommerce/repo
    git --bare init
   ```

4. automate deployment on every git push

   ```shell
   nano hooks/post-receive
   ```

   ```txt
    #!/bin/bash -l
    echo 'post-receive: Triggered.'
    cd ~/apps/nextjs-ecommerce/dest/
    git --git-dir=/root/apps/nextjs-ecommerce/repo --work-tree=/root/apps/nextjs-ecommerce/dest/ checkout main -f
    pnpm install
    pnpm run build
    pm2 restart nextjs-ecommerce
   ```

5. make post-receive executable

   ```shell
   chmod ug+x hooks/post-receive
   ```

6. npm install -g pnpm
7. create postgres database

   - create database at https://vercel.com/docs/storage/vercel-postgres
   - copy POSTGRES_URL

   ```shell
   cd ../dest
   nano .env
   ```

   ```txt
    NEXT_PUBLIC_SERVER_URL=http://nextjs-ecommerce.com
    NEXT_PUBLIC_APP_NAME=Next ECommerce App
    NEXT_PUBLIC_APP_DESCRIPTION=An ECommerce App built with Next.js, Postgres, Shadcn

    # Database (vercel or neon db url)
    POSTGRES_URL=your_postgres_url

    # NextAuth ($ npx auth secret)
    AUTH_SECRET=fFLDeFDO7rzsZ/TdxINaG2VUogiPEZX/LgTl2FAoReY=
    AUTH_TRUST_HOST=true

    # API Keys
    RESEND_API_KEY=123
   ```

8. [On Dev Machine] clone repo

   ```shell
    git clone https://github.com/basir/next-pg-shadcn-ecommerce
    cd next-pg-shadcn-ecommerce
    git remote add hostinger ssh://root@5.182.18.65:/root/apps/nextjs-ecommerce/repo/
   ```

9. [On Dev Machine] add dummy changes, commit and push

   ```shell
   git add . && git commit -m "commit message" && git push hostinger
   ```

10. config pm2

```shell
 sudo npm install -g pm2
 cd /root/apps/nextjs-ecommerce/dest/
 npx drizzle-kit push
 npx tsx ./db/seed
 sudo pm2 start npm --name <name> -- start
 pm2 save
 pm2 startup
```

11. [On Dev Machine] add dummy changes, commit and push

    ```shell
    git add . && git commit -m "commit message" && git push hostinger
    ```

12. open http://5.182.18.65:3000
13. config apache

    ```shell
    sudo apt install apache2
    sudo a2enmod proxy proxy_http rewrire headers expires
    sudo nano /etc/apache2/sites-available/nextjs-ecommerce.com.conf
    ```

```txt
<VirtualHost *:80>
    ServerName admin.cartex.pk
    Redirect permanent / https://admin.cartex.pk/
</VirtualHost>

<VirtualHost *:443>
    ServerName admin.cartex.pk

    SSLEngine on
    SSLCertificateFile /etc/letsencrypt/live/cartex.pk/fullchain.pem
    SSLCertificateKeyFile /etc/letsencrypt/live/cartex.pk/privkey.pem

    ProxyPreserveHost On
    ProxyPass / http://127.0.0.1:3001/
    ProxyPassReverse / http://127.0.0.1:3001/

    ErrorLog /var/log/apache2/admin.cartex.pk-error.log
    CustomLog /var/log/apache2/admin.cartex.pk-access.log combined
</VirtualHost>
```

    ```shell
    sudo a2dissite 000-default.conf
    sudo a2ensite nextjs-ecommerce.com.conf
    sudo service apache2 restart
    ```

14. open http://nextjs-ecommerce.com
15. secure website

    ```shell
    sudo apt install certbot python3-certbot-apache
    sudo certbot -d nextjs-ecommerce.com -d www.nextjs-ecommerce.com -m basir.jafarzadeh@gmail.com --apache --agree-tos  --no-eff-email --redirect
    sudo certbot renew --dry-run
    ```

16. update http to https in .env file

    ```shell
    cd /root/apps/nextjs-ecommerce/dest/
    nano .env
    ```

    ```txt
    NEXT_PUBLIC_SERVER_URL=https://nextjs-ecommerce.com
    ```
17. open https://nextjs-ecommerce.com
