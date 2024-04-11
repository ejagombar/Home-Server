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

        #/etc/supervisor/conf.d/myapp.conf

        [program:spannerbackend]
        directory=/home/spanner/SpannerBackend
        command=/home/spanner/SpannerBackend/main
        autostart=true
        autorestart=true
        stderr_logfile=/var/log/myapp.err
        stdout_logfile=/var/log/myapp.log
        environment=GOPATH="/root/gocode"

 - Supervisor could then be restarted to load the config `supervisorctl reload`

 - The status of Supervisor can be checked with `supervisorctl status` and if everything is working properly, it should show something like this:

        spannerbackend                   RUNNING   pid 5344, uptime 3 days, 8:29:59

For some reason, on my system I had to manually start supervisord. To fix this, and to ensure that supervisord starts auitomatically on reboot, I used:

- `systemctl enable supervisord.service` which ensures that systemd starts supervisord on boot. 

- `systemctl start supervisord.service` which starts the service now.

## Setting up the Frontend

after much debugging, i had to disable selinux
sudo vim /etc/selinux/
and changing the config file to disabled


## ssh-keygen stuff

1) ssh-keygen -t ecdsa on local machine 
2) name it as localmachine_to_remote_id_ecdsa
3) scp copy the .pub file to the remote in the ~/.ssh/ dir
4) logon to remote and append the file to autorized_keys which is in .ssh
5) on loca machine, add to config file: example below

Host spanner
    HostName spanner.eagombar.uk
    User spanner
    IdentityFile ~/.ssh/chip_to_spanner_id_ecdsa




# Lets Encrypt for SSL
pretty simple to setup - just follow the instructions inline and everything worked


# SSL Backend
Have been implementing SSL for the backend, however I have just realised that port 443 is already being used by the frontend for HTTPS so I cant use it for the backend too. I think I need a proxy or something to fix this 
https://discourse.haproxy.org/t/binding-port-443-to-both-http-and-tcp/5613/3
https://www.reddit.com/r/node/comments/13tyiew/hosting_frontend_and_backend_on_same_domain/


