# Hosting Spanner Backend and Frontend Server Notes

Running Fedora Server in a VM on Proxmox.

Clone SpannerBackend using
git clone -b SingleUserBoltDb <url>
to get the correct branch

Need environment variables. Can copy them to the server using:
scp .\app.env spanner@192.168.50.57:~/SpannerBackend
