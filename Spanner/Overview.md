# Locally Hosting [Spanner](https://spanner.eagombar.uk)
I decided to document the process of setting up my website frontend and backend servers on my Proxmox server. Hopefully writing this will make it easier to do again and also serve as some notes incase anything goes wrong.

## Environment
I am runnign Fedora Server in a VM on Proxmox. Originally, I was going to use a Linux Container (LXC) as this would require less resources, however for some reason SSH was not working properly and I could not figure out why. I shall come back to it at some point.
I chose Fedora as the OS as I have the most experience with it, however I regret not using Ubuntu as I had some additional issues with NGINX which would not be present on Ubuntu. There is also just generally better support for it.


## Configuring Firewall Rules
Some firewall rules need to be configured to allow TCP traffic on ports 80, 443, and 8080. Fedora uses firewallD and this can be acheived with the commands below.

- Open a port `firewall-cmd --add-port=<port>/tcp --permanent`

- Reload firewall to implement changes `firewall-cmd --reload`

FirewallD also comes with some common services which can be used to quickly configure the firewall. These services can be listed with `firewall-cmd --get-services` and implemented using `firewall-cmd --add-service=https --permanent`

Proxmox also comes with its own firewall which can be configured using the GUI. The same ports must be opened there too. Proxmox also offers 'macros' which are similar to firewallD's services.

## Setting up the Backend

Looking back, I should have built the Go server on my local machine and then copied the binary to the server. However instead, I chose to install Go on the VM and build it there.

I need to clone SpannerBackend on the "SingleUserBoltDb" branch

`git clone -b <branchName> <url>`

Go was installed and the project was built with `go build .`

For the server to function, it needs the envinment variables. Can copy the .env file to the server from my host machine using the scp command line tool

`scp <file> <hostname>@<ip>:<destination path>`

### Testing
The server can now be tested by running the binary and accessing it at `http://<serverIP>:8080/`

To test it, I ran `curl -X GET 192.168.50.57:8080/api/profile/info`

This returns:

`{"displayname":"Ed Agombar","followercount":27,"imageurl":"https://i.scdn.co/image/ab67757000003b8229bb9cb2112130441b4ad53e"}`

Looks good!

### Running as a Process
Now that this works, the server needs to run in the background, and on startup of the server. It should also restart if the server crashes. To do this, I followed [this](https://medium.com/@monirz/deploy-golang-app-in-5-minutes-ff354954fa8e) tutorial, however I came across some differences when using Fedora so I shall document my process here.

- Install Supervisor `dnf install supervisor`

- Add Supervisor to the system usergroup `groupadd --system supervisor`

- Create Supervisor config file `vim /etc/supervisord.d/spanner.ini`

- Here is the `spanner.ini` config file for the backend:

    ```
    #/etc/supervisor/conf.d/myapp.conf

    [program:spannerbackend]
    directory=/home/spanner/SpannerBackend
    command=/home/spanner/SpannerBackend/main
    autostart=true
    autorestart=true
    stderr_logfile=/var/log/myapp.err
    stdout_logfile=/var/log/myapp.log
    environment=GOPATH="/root/gocode"
    ```

 - Supervisor could then be restarted to load the config `supervisorctl reload`

 - The status of Supervisor can be checked with `supervisorctl status` and if everything is working properly, it should show something like this:

        spannerbackend                   RUNNING   pid 5344, uptime 3 days, 8:29:59

For some reason, on my system I had to manually start supervisord. To fix this, and to ensure that supervisord starts auitomatically on reboot, I used:

- `systemctl enable supervisord.service` which ensures that systemd starts supervisord on boot. 

- `systemctl start supervisord.service` which starts the service now.

## Setting up the Frontend

To host the frontend, I decided to use NGINX. I chose this over Apache due to NGINX being slightly more modern, and I had more of an interest to learn about it.

To setup NGINX, I followed [this](https://www.youtube.com/watch?v=KFwFDZpEzXY) tutorial which shows how to deploy a React app using NGINX and linux. Again, this tutorial was conducted on Ubuntu, so I shall document the differnences and challenges I came across.

#### Differences:

1) The NGINX usergroup is called nginx on Fedora, as opposed to www-data

2) NGINX on Fedora does not include the `sites-available` and `sites-enabled` folders, however this can be implemented manually fairly simply. Create the folders, and then in nginx.conf, include `/etc/nginx/sites-enabled/*;` in the `http{}` block.

#### Challenges:

After following the tutorial, my website was unnaccesible at the IP address of the server. NGINX was running correctly as I would get an NGINX page showing 403 permission denied.
I checked all the read and write permissions of the server files, as well as the parent directories, however  I had no success in fixing the permission denied error.
Eventually, after much debugging, I found a forum with a similar issue. Fedora ships with SElinux, which was interfering with NGINX and preventing it from accessing the server files. To get around this issue, I disabled SELinux by editing the config file at `/etc/selinux/` and setting `SELINUX=disabled`.

After this, website could be accessed by entering the IP address of the virtual machine into the browser.

## Domain Name Setup

So that the website can be accessed outside my local network, I need to setup port forwarding on the router. I opened up port 80, 443, and 8080 and configured them to point to the virtual machine.

As I do not have access to a static IP address, I used [DuckDNS](https://www.duckdns.org/) which is a free dynamic DNS service. It works by installing a small piece of software on your local server which will routinely transmit the IP of the router to DuckDNS. DuckDNS will assign you a domain that points to that IP address. This means that it is simple to access the home network remotely, even its IP address changes. 

I own the domain eagombar.uk so I add a CNAME record to my domain provider to point spanner.eagombar.uk to the DuckDNS domain.

## SSL With Lets Encrypt
Although the website works, it used HTTP which modern browsers complain about. To obtain HTTPS, I need a SSL certificate which luckily can now be obtained for free and very easily using [Lets Encrypt](https://letsencrypt.org/). I just followed the steps provided on the website to set up the certificate.

Although the HTTPS was now available on the frontend of the website, it could no longer communicate with the backend, which was still using HTTP as this is blocked by the browser.
I implemented SSL certificates on the backend in Go, however when it came time to test it, I realised that as both frontend and backend services are running on the same VM, they would be sharing port 443 which would not be possible. Instead, I setup a reverse proxy in NGINX which would recieve all traffic, and depending on the desitination, it will be either sent as a HTTP request to port 8080 for the backend, or sent to the frontend server. This meant that the SSL Go code could be removed.

The full NGINX configuration can be seen below:

```
server {
    server_name spanner.eagombar.uk;
    root /var/www/spanner.eagombar.uk;
    index index.html;

    error_log /var/log/nginx/spanner_error.log;
    access_log /var/log/nginx/spanner_access.log;

    listen [::]:443 ssl ipv6only=on; # managed by Certbot
    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/spanner.eagombar.uk/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/spanner.eagombar.uk/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    # Proxy backend API requests
    location /api {
        proxy_pass http://localhost:8080;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection 'upgrade';
        proxy_set_header Host $host;
        proxy_cache_bypass $http_upgrade;
    }

    # Serve static files
    location / {
        try_files $uri $uri/ =404;
    }
}

server {
    if ($host = spanner.eagombar.uk) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

    listen 80;
    listen [::]:80 ipv6only=on;

    server_name spanner.eagombar.uk;
    return 404; # managed by Certbot
}
```

## Sidenote: SSH keygen stuff
Usefule notes for setting up passwordless SSH again.

1) Run `ssh-keygen -t ecdsa` on local machine 
2) Name it as localmachine_to_remote_id_ecdsa
3) `scp` copy the .pub file to the remote in the ~/.ssh/ dir
4) Logon to remote and append the file to autorized_keys which is in .ssh
5) On the local machine, add to config file:

    ```
    Host spanner
        HostName spanner.eagombar.uk
        User spanner
        IdentityFile ~/.ssh/chip_to_spanner_id_ecdsa
    ```
