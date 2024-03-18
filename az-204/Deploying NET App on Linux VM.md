- Login into linux VM from console, use PuTTY on Windows or just use Mac console directly
- Install the runtime on linux VM using bash commands (refer .NET installation instructions for linux VM)
- Publish .NET WebAPP to a folder and SFTP it to /homr/linuxuser/
- Go to publish folder on linux VM console and run "dotnet webapp.dll"
- This will run the webapp on Kestrel Web Server on port 5000
- verify using curl command on linux console

NGINX Web Server
- install nginx using "sudo apt-get install nginx"
- Now if you open the IP address in web browser it will display "Welcome to nginx" page.
- sudo service nginx stop
- go to /etc/nginx/sites-available
- there make a change in "default" file
- change permissions first "sudo chmod 777 default"
- make a change in this file (look for location word) to redirect to http://localhost:5000 (The Kestrel Web Server where we deployed)
- save the file and change permissions back to "sido chmod 644 default"
- sudo service nginx start
- go to browser and access the page by IP address, it will show our .NET app
