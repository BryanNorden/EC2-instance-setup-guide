# Steps-to-setup-EC2-instance-on-AWS
Steps to setup an EC2 instance on AWS with an Nginx proxy server in front. 

This is a brief guide to be used as reference. More detailed info about creating an instance is provided by Amazon at https://docs.aws.amazon.com/ec2/

## Intial Setup

1. Create and launch your EC2 instance on AWS (Need AWS account - free 1 year trial)
   - Swift servers work well with Ubuntu 18.04

1. Create a security group if you do not have one.

1. Open up port 80 for HTTP requests to `0.0.0.0/0` and `::/0`

1. Open up port 443 for HTTPS requests to `0.0.0.0/0` and `::/0`

1. Open up your server port (8080 is typlical for an api server) to `0.0.0.0/0`

1. Attach your security group to your instance(s)

1. Optional - add database if needed

1. create a ssh key 
   - I like to create a ssh host shortcut in the `~/.ssh/config`

1. Mark your ssh private key as read-only otherwise AWS will refuse the connection
   - `chmod 600 ~/.ssh/key` or `chmod 600 /path/to/your/ssh/key` 
   
1. ssh into instance and update system
   - `apt-get update`
   - `apt-get upgrade`

## Vapor Server Setup

1. ssh into instance if not already in

1. Switch to root user `sudo su -` (needed for updating and installing)

1. Add the signing key from the Vapor APT repository. This allows apt-get to verify the packages it downloads from the repository.
   - `wget -q https://repo.vapor.codes/apt/keyring.gpg -O- | apt-key add -`
   
1. Add the correct APT repository for your Ubuntu version.
   - `echo "deb https://repo.vapor.codes/apt $(lsb_release -sc) main" | tee /etc/apt/sources.list.d/vapor.list`
   
1. Update apt `apt-get update`

1. Install Swift and the SSL C bindings. `apt-get install swift ctls`

1. Verify setup and view Swift version `swift --version`

1. Add your instance ssh to your github if not there already

1. Exit from root user `exit`

1. Clone your project and cd to project directory 

1. Create a build `swift build -c release`

1. Test the build runs `./.build/release/Run`

### Setup Supervisor 
This will keep the server running when you exit ssh and will restart it when it crashes
   - Supervisor documentation: http://supervisord.org

1. Switch to root user `sudo su -`

1. Should already be installed, but if not install `apt-get install supervisor`

1. Create a new supervisor config with your server name `vim /etc/supervisor/conf.d/configName.conf`
   ```
   [program: ProjectName]
   command=/home/ubuntu/projectDirectory/.build/release/Run serve
   directory=/home/ubuntu/projectDirectory/
   autostart=true
   autorestart=true
   user=ubuntu
   stdout_logfile=/var/log/supervisor/ProjectName-out.log
   stderr_logfile=/var/log/supervisor/ProjectName-err.log
   ```
   
1. Reread supervisor configs `supervisorctl reread`

1. Updated supervisor with new/updated config `supervisorctl update`

### Setup Nginx 
Simple proxy server that will sit in front of your server to handle routing traffic to the correct places before touching your core server. This will also be the place where you will enable SSL for HTTPS traffic and handle load balancing
   - NGINX documentation: https://nginx.org/en/docs/

1. Switch to root user `sudo su -` if not a root user

1. Install Nginx `apt-get install nginx`

1. Create new Nginx configuration file `vim /etc/nginx/conf.d/projectName.conf`
   ```
   server { 
      ## Server name is iP or domain name(s)
      server_name example.com www.example.com;
    
      root /home/ubuntu/ProjectDirectory/Public; 
      try_files $uri @proxy;

      location @proxy {
         proxy_pass http://127.0.0.1:8080;
      
         proxy_set_header Host $host;
         proxy_set_header X-Real-IP $remote_addr;
         proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; 
      
         proxy_connect_timeout 3s;
         proxy_read_timeout 10s;
      } 
   }
   ```
   
1. Start nginx `systemctl start nginx`
   - can also restart and stop `systemctl restart nginx` or `systemctl stop nginx`
   
#### Add HTTPS to your nginx server
Documentation found at https://docs.nginx.com/nginx/admin-guide/security-controls/terminating-ssl-http/ but I will be adding it via Let's Encrypt SSL/TLS https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/

1. Switch to root user `sudo su -`

1. Create the certbot repository `add-apt-repository ppa:certbot/certbot`

1. Update and install certbot
   - `apt-get update`
   - `apt-get install python-certbot-nginx`

1. Generate certs with certbot with your domain `sudo certbot --nginx -d example.com -d www.example.com`
   - Follow the instrutions, but it should show something like this if successful 
   ```
   Congratulations! You have successfully enabled https://example.com and https://www.example.com 

   -------------------------------------------------------------------------------------
   IMPORTANT NOTES: 

   Congratulations! Your certificate and chain have been saved at: 
   /etc/letsencrypt/live/example.com/fullchain.pem 
   Your key file has been saved at: 
   /etc/letsencrypt/live/example.com//privkey.pem
   Your cert will expire on 2017-12-12.
   ```
   
1. It should have updated your nginx config file /etc/nginx/conf.d/projectName.conf

1. These certbot certificates expire after 90 days so we need to add a cron job to renew and reload within 30 days. So open crontab file
   - `crontab -e`
   
1. Add the command to run daily. This will run it every day at noon. The command checks to see if the certificate on the server will expire within the next 30 days, and renews it if so. The --quiet directive tells certbot not to generate output. 
   - `0 12 * * * /usr/bin/certbot renew --quiet`
