# Deploy fullstack app to EC2

This is a step by step process to deploying Node.js with express as backend, mySql as DB and React as front on Ubuntu 20.04 with NGINX.
This process is merge of three different tutorials I found:

1. Deploy fullstack app [video](https://www.youtube.com/watch?v=NjYsXuSBZ5U) and [written](https://github.com/Sanjeev-Thiyagarajan/PERN-STACK-DEPLOYMENT) description by 'Sanjeev-Thiyagarajan'

2. Deploy fullstack app [tutorial](https://towardsdev.com/deploying-a-react-node-mysql-app-to-aws-EC2-2022-1dfc98496acf) by 'Royce Ho'

3. Running mysql [tutorial](https://towardsdatascience.com/running-mysql-databases-on-aws-EC2-a-tutorial-for-beginners-4301faa0c247) by 'Naser Tamimi'

## 1. Launch EC2 instance

Create an AWS account and in the console go to the EC2 Dashboard → Instances → Launch Instances.

Use the following configuration for the instance (those are configurations for free tier):

* AMI - `Ubuntu Server 22.04`.
* Instance Type - `T2.micro`.
* Network:
    * Leave the VPC as the default one for convenience.
    * Create a new security group that allows:
        inbound:
        * SSH(:22) - your IP
        * HTTP(:80) and HTTPS(:433) traffic from anywhere
        * MYSQL/Aurora(:3306) traffic from anywhere
        outbound:
        * all traffic from anywhere
    * Select or create a key pair. Once you create one, a `.pem` file will be download to your computer.

**Click on Launch instance.**

SSH into instance:

Select on your instance and click on 'connect' button.

See instructions for connecting via SSH

Open your local terminal where the `.pem` key file is located and follow instruction for connecting

Once you connected successfully you can continue.

## 2. Install NGINX, Git and Node on your instance

Run all next commands in your instance terminal (via the SSH connection):

### Update/refresh the list of available packages

```
sudo apt update
```

### Install git to clone your repository from github

```
sudo apt install git
```

### Install node.js

```
cd ~
curl -sL https://deb.nodesource.com/setup_14.x | sudo -E bash -
sudo apt-get install -y nodejs
```

[More options for installing Node](https://github.com/nodesource/distributions/blob/master/README.md#debinstall)

### Install NGINX as reverse proxy & web server

NGINX is a service that acts as a web server and a reverse proxy. We opened port 80 for HTTP requests to enter from the web. Given that there is only one incoming port (80) but 2 services (frontend and backend), a reverse proxy acts as an abstraction to direct requests to 2 different ports (3000 for frontend and 8081 for backend in this example).

```
sudo apt install -y NGINX
sudo systemctl enable NGINX
```

NGINX should've started automatic.

Check it by:

```
service NGINX status
```

If NGINX not started, you can run

```
sudo systemctl start NGINX
```

> ### NGINX troubleshooting
> `sudo cat /var/log/NGINX/error.log` to view logs

In the AWS console, under your EC2 instance, click on your public IPV4 DNS address. If you see the NGINX welcome page, you’re on the right track and NGINX is working.

**Be sure you are changing your address to your IPV4 to `http://` and not `https://`**


## 3. Install and Configure MySQL

```
sudo apt install -y mysql-server
```

MySQL should run automatically. You can check it with the following command.

```
sudo systemctl status mysql
```

Log in as root.

```
sudo mysql
```

Set password for your root.

Replace `your_password_here with` a strong password.

```
mysql> ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'your_password_here';
mysql> FLUSH PRIVILEGES;
```

[Reference](https://towardsdatascience.com/running-mysql-databases-on-aws-EC2-a-tutorial-for-beginners-4301faa0c247)

Exit and login again with root new password.

```
mysql> exit
$ sudo mysql -u root -p
```

Enter the new password and wait for mysql command line.

If you can logged in, you can move on.

### Installing MySQL Workbench for Easier Management

MySQL Workbench is a visual tool for database administration. It helps you to do complex database management tasks in a short time without sacrificing any flexibility. We can install this application on our local computer and manage any local or remote databases.

Install mysql workbench on your local computer.

After installing, click on + next to MySQL Connections.

#### Setup new connection:

* Connection Name - whatever you want.
* Connection method - `Standard TCP/IP over SSH`.
* SSH hostname - your EC2 Instance `Public IPv4 DNS` address.
* SSH username - `ubuntu`.
* SSH key file - locate you `.pem` file you downloaded when created your EC2 instance.
* Mysql hostname - `127.0.0.1`.
* Mysql server port - `3306`.
* Username - `root`.

Click OK and your connection should setup.

Open the connection by clicking on it and enter your MySQL root password you setup earlier.

Now you can see your databases and tables in the *schema* tab.

Import your DB exported file and execute it by clicking on the bolt button.

After some seconds you should see your table in the *schema* tab.

Your mysql DB is setup !

In your instance command line you can enter to mysql server and check if your DB was created:

```
$ mysql -u user -p
mysql > SHOW DATABASES;
mysql > USE database_name;
mysql > SHOW TABLES;
```

## 4. Prepare your app to deployment

configurations:

* If you are using typescript, you can convert your app to js by running `npx tsc` in backend folder, and change 'start' script to `node build/app.js`.
* If you are using typescript, change "start" script to `ts-node src/app.ts`. make sure you are installing ts-node.
* Port config - App.listen: `process.env.PORT || 3000` in app file.
* If you are serving static files: `express.static(__dirname + 'your-dirname')`
* mysql connection:
  
```
    mysql = {
        mysqlHost: "localhost",
        mysqlPort: "3306",
        mysqlUser: "root",
        mysqlPassword: "YOUR_PASSWORD",
        mysqlDatabase: "DB_NAME",
    };
```

* server url: `http://YOUR-PUBLIC-IPv4-DNS:3000`

## 5. Clone repository and install dependencies

```
git clone https://github.com/demo/webappdemo.git
```

### Install front-end node dependencies & create build folder

```
cd app/frontend
sudo npm install
sudo npm run build

# install back-end dependencies
cd ../backend/ && sudo npm install

# for building and compile js files from ts files
sudo npx tsc
```

Now we need to point NGINX to the `index.html` file of the frontend, and launch `app.js` for backend.

## 6. Configure NGINX

NGINX is a feature-rich webserver that can serve multiple websites/web-apps on one machine. Each website that NGINX is handling needs to have a separate server block configured for it.

To see all server blocks of NGINX navigate to `/etc/NGINX/sites-available`

```
cd /etc/NGINX/sites-available
```

There should be all server blocks. On first time only one called `default` exist.

```
/etc/NGINX/sites-available$ ls
default
```

The default server block is responsible to handle requests that don't match any other server blocks. Right now if you navigate to your server ip, you will see html page that says NGINX is installed. That is the `default` server block in action.

**Remember that for now we need to use `http://` and not `https://`**

### If you have only one application on your instance:

Copy the frontend build folder over to the directory `/var/www/`.

```
sudo cp -R /home/ubuntu/app/Frontend/build /var/www
```

Configure `default` file:

```
sudo vi /etc/NGINX/sites-available/default
```

Use the following configuration:

```
server {
    listen 80 default_server;
    listen [::]:80 default_server;
    # We want the root folder to point at index.html
    root /var/www/build;
    index index.html index.htm index.NGINX-debian.html;

    server_name _;

    location / {
        # First attempt to serve request as file, then
        # as directory, then fall back to displaying a 404.
        try_files $uri /index.html $uri/ =404;
    }
    
    location /api {
        proxy_pass http://localhost:8081;
    }
}
```

Check the logs to see if there are any issues with NGINX - `sudo cat /var/log/NGINX/error.log`

#### Explanation of the above file

* `80` represents the HTTP port which NGINX listens for web traffic
* `root` sets the directory for requests. In other words, usually your root file (index.html) should be in the folder. In this case, it is in our build folder for React.
* `server_name` does not need to change, as there is only one server block. See [here](https://NGINX.org/en/docs/http/server_names.html) for more info.
* `location /` and `location /api` specifies the configuration based on a request URI. In this case, the latter means that all requests with a prefix of `/api` will go to the Node server.

### If you want to have more than one app

If you have more then one application to run on this instance, you should point NGINX to every one of those app's `index.html` files.

We will need to configure a new server block for every website. To do that let's create a new file in `/etc/NGINX/sites-available/` directory.

Instead of creating a brand new file, copy the default config file and change it.

```
cd /etc/NGINX/sites-available
sudo cp default SERVER-BLOCK-NAME
```

open the new server block file (I used `example.com` as server name for the tutorial) and modify it so it matches below:

```
sudo vi example.com
```

```
server {
        listen 80;
        listen [::]:80;

        # Note that we need to give a full path to the relevant folder.
        root /home/ubuntu/app/example-app/frontend/build;

        index index.html index.htm index.NGINX-debian.html;

        server_name example.com www.example.com;

        location / {
                try_files $uri /index.html;
        }

         location /api {
            proxy_pass http://localhost:3001;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection 'upgrade';
            proxy_set_header Host $host;
            proxy_cache_bypass $http_upgrade;
        }
}
```

#### Let's go over what each line does

The first two lines `listen 80` and `listen [::]:80;` tell NGINX to listen for traffic on port 80 which is the default port for http traffic. Note to removed the `default_server` keyword on these lines. If you want this server block to be the default then keep it in

`root /home/ubuntu/apps/example-app/frontend/build;` tells NGINX the path to the index.html file it will server. Here we passed the path to the build directory in our react app code. This directory has the finalized html/js/css files for the frontend.

`server_name example.com www.example.com;` tells NGINX what domain names it should listen for. Make sure to replace this with your specific domains. If you don't have a domain then you can put the ip address of your ubuntu server.

The configuration block below is needed due to the fact that React is a Singe-Page-App. So if a user directly goes to a url that is not the root url like `https://example.com/sample/1` you will get a 404 cause NGINX has not been configured to handle any path other than the `/`. This config block tells NGINX to redirect everything back to the `/` path so that react can then handle the routing.

```
        location / {
                try_files $uri /index.html;
        }
```

The last section is so that NGINX can handle traffic destined to the backend. Notice the location is for `/api`. So any url with a path of `/api` will automatically follow the instructions associated with this config block. The first line in the config block `proxy_pass http://localhost:3001;` tells NGINX to redirect it to the localhost on port 3001 which is the port that our backend process is running on. This is how traffic gets forwarded to the Node backend. If you are using a different port, make sure to update that in this line.

### Enable the new site

```
sudo ln -s /etc/nginx/sites-available/example.com /etc/nginx/sites-enabled/
systemctl restart nginx
```

Now if we navigate to our EC2 public IP we should see our frontend

**Remember, right now we need to use `http://` and not `https://`**

## 7. Install and Configure PM2

PM2 is an acronym of Process Management Module which is used to run and manage Node applications. It’s an open-source with an in-built load balancer. When a process goes down, PM2 will automatically restart the service and make it live. We never want to run node directly in production. Instead we want to use a process manager like PM2 to handle running our backend server.

```
sudo npm install pm2 -g
```

Point pm2 to the location of the `app.js` file so it can start the app. We can add `--name` flag to give the process a descriptive name

```
pm2 start /home/ubuntu/app-dir/backend/app.js --name app-name
```

Configure PM2 to automatically startup the process after a reboot

```
$ pm2 startup
[PM2] Init System found: systemd
[PM2] To setup the Startup Script, copy/paste the following command:
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

The output above gives you a specific command to run, copy and paste it into the terminal. The command given will be different on your machine depending on the username, so do not copy the output above, instead run the command that is given in your output.

```
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u ubuntu --hp /home/ubuntu
```

Verify that the App is running:

```
pm2 status
```

After verify App is running, save the current list of processes so that the same processes are started during bootup. If the list of processes ever changes in the future, you'll want to do another pm2 save

```
pm2 save
```

## 8. Updating changes and helpful commands

If you make changes in your app files (front or back) you need to handle the update in your instance.

First, pull your repository for update all your changes:

```
git pull https://github.com/example/example.git
```

If you updated files in your frontend, you need to restart NGINX.

```
sudo systemctl restart NGINX
```

**Important: if you copied `build` folder on step 4, you must update the copy in the default PATH too:**

```
cd /var/www
sudo rm -r build
sudo cp -R /home/ubuntu/app/Frontend/build /var/www
sudo systemctl restart NGINX
```

If you updated files in your backend, you need to restart pm2 service.

```
pm2 restart PROCESS-ID-NUMBER
```

#### More pm2 commands that can be helpfull:

Monitoring backend log:

```
pm2 logs
```

Stop pm2 process:

```
pm2 stop process_id
```

Delete pm2 process:

```
pm2 delete process_id
```
