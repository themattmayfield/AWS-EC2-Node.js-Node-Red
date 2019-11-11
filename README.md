# AWS-EC2-Node.js-Node-Red
Creating and managing a Node.js server on AWS 

### What will be discussed? How to: 
* Start an AWS server
* SSH into the server
* Install Node.js
* Create a public HTTP endpoint that responds with a static message
* Serve HTTP traffic on the standard port, 80
* Keep the Node.js process running
* Deploy code into the server
* Serve some HTML
* Install Node-Red on AWS EC2 instance


## Start an AWS server
1. Log in to the AWS EC2 console

2. Click ‘Launch Instance’

3. In the list of Quick Start AMIs, select Ubuntu Server

4. Select the Instance Type - t2.micro is a good starting point

5. Click Next: Configure Instance, click Next: Add Storage, click Next: Add Tags

6. Click Next: Configure Security Group.
- A security group is a config for your server, telling it which ports it should expose to which IP addresses for certain types of traffic. 
- Name the group something meaningful
- To run our app we are going to need SSH access, which by default is on port 22 and uses the TCP protocol. Amazon adds this in for us by default.
- Since we would like to also serve an app we need to expose a HTTP port publicly, by default this is port 80

7. Click Add Rule and select the type as HTTP, the default settings for this will use TCP as the protocol and expose port 80 to all IPs.

8. To launch the instance, click Review and Launch, then click Launch. 
- You will be prompted to setup an SSH key which will give you access to the instance. 

9. Choose create a new key pair, and name the key something meaningful. 

10. Click Download Key Pair. 
- This should download a .pem file which can be used to SSH into the server. 
- Keep this file safe because anyone can connect to your server using it, if you lose the file you will need to generate a new one.

11. Click Launch Instance. Click View Instances.

## SSH into the server

The correct place to put your .pem file is in your .ssh folder, in your user directory. The .ssh folder is a hidden folder, to open it in finder open terminal and execute the open command.

```
# The open command will open the
# given path using the default system
# application for the file type.
$ open ~/.ssh
```

Once the .pem file is in your ssh folder, use chmod to set permissions so that it can be used as a key.

```
$ chmod 400 ~/.ssh/whatever-your-key-name-is.pem
```

To SSH we need to have a username, an address and a key. The address is available when we click on our instance in the EC2 instances dashboard.

To connect, the SSH command should look something like

```
# Fill in your specific details for this to work.
$ ssh -i ~/.ssh/whatever-your-key-name-is.pem ubuntu@your-public-DNS
```
This will connect you to your instance, just type yes when prompted so that you can add your instance as a known host.

## Install Node.js

Once in an SSH session the first thing to do is get Node.js. NVM (Node Version Manager) is a pretty great way to install Node.js and allows you to easily switch versions if required.

To install NVM just run this command.
```
$ curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.32.1/install.sh | bash
```

Then to get nvm working, run 
```
$ source ~/.bashrc
```

Find out the latest version of node. Then install by running
```
$ nvm install 7
```
To check node is ready to go just echo the version.
```
$ node --version
```

## Create a public HTTP endpoint that responds with a static message

Now make a public URL anyone can request from the browser to get a response from the server.

1. Make a directory for the server and cd into it.
```
$ mkdir server
$ cd server
```
2. Now you are in your server directory, you need to npm init
```
$ npm init
```
3. Install express and add it to package.json.
```
$ npm install express --save-dev
```
- You should have a node_modules directory and package.json
4. Add some code to run the server
```
$ nano index.js
```

```javascript
var express = require('express');
var app = express();
app.get('/', function (req, res) {
  res.send('Hello World!');
});
app.listen(3000, function () {
  console.log('Example app listening on port 3000!');
});
```
5. Save and exit
6. Open up port 3000 so we can test our server.
- Leave your server running and go to the Security Groups tab in the EC2 console. 
- Right click the security group you setup and click edit inbound rules. 
- Click Add Rule. This time we are going to use a custom TCP rule on port 3000, open to anywhere.
- Click save
7. Visit your public DNS URL with port 3000 and you should see the Hello World! response.
8. To leave the server running when we log out
- Press ctrl+z to pause the process (this only works when your server is running, node index.js)
- You can see that the job number for node index.js is 1 (as noted by [1]+). To run that in the background, use the bg command.
```
$ bg %1
```
- Logout
```
$ exit
```

You should still be able to access your URL and see the “HEY!” response.

## Serve HTTP traffic on the standard port, 80

1. SSH into the server if you are not.
2. Run the command
```
sudo apt-get install nginx
```
apt-get runs nginx automatically after install so you should now have it running on port 80, check by entering your public DNS URL into a browser.

We need to configure nginx to route port 80 traffic to port 3000.

3. Let’s first remove the default config from sites-enabled, we will leave it in sites-available for reference.
```
sudo rm /etc/nginx/sites-enabled/default
```
4. Create a config file in sites-available and name it whatever you like.
```
sudo nano /etc/nginx/sites-available/your-custom-name
```
5. The config to use
```
server {
  listen 80;
  server_name your-custom-name;
  location / {
    proxy_set_header  X-Real-IP  $remote_addr;
    proxy_set_header  Host       $http_host;
    proxy_pass        http://127.0.0.1:3000;
  }
}
```
This will forward all HTTP traffic from port 80 to port 3000.

6. Link the config file in sites enabled (this will make it seem like the file is actually copied insites-enabled).
```
sudo ln -s /etc/nginx/sites-available/tutorial /etc/nginx/sites-enabled/tutorial
```
7. Restart nginx for the new config to take effect.
```
sudo service nginx restart
```
8. See if node application is running
```
# list background jobs
jobs
```
If your application shows up then no need to run it again. If it is not running then you need to start it.
```
node tutorial/index.js
```
Once the server is running, press ctrl+z, then resume it as a background task.
```
bg %1
```
Now visit your server’s public DNS URL, using port 80.

## Keep the Node.js process running

Before moving forward, stop your running node process
```
# Nukes all Node processes
killall -9 node
```
To keep these processes running we are going to use a great NPM package called PM2. While in an SSH session, install PM2 globally.
```
npm i -g pm2
```
To start your server, simply use pm2 to execute index.js.
```
pm2 start tutorial/index.js
```
To make sure that your PM2 restarts when your server restarts
```
pm2 startup
```
This will print out a line of code you need to run depending on the server you are using. 
Run the code it outputs.
Finally, save the current running processes so they are run when PM2 restarts.
```
pm2 save
```
You can log out/in to SSH, even restart your server and it will continue to run on port 80.
To list all processes use
```
pm2 ls
```
Your process will have a pretty generic name, something like index, which would be hard to differentiate if you had a few other microservices running on the server too. Let’s stop the process, remove it and start it back up with a better name.
```
# Use the number listed in pm2 ls
# to stop the daemon
pm2 stop index
# Remove it from the list
pm2 delete index
# Start it again, but give it a
# catchy name
pm2 start your-custom-name/index.js --name “Your-Custom-Name”
```

## Deploy code into the server
Instead of writing code in an SSH session, let’s push the code to a git repo in Github, SSH into the server and pull in the new code.

1. Go to Github, login and create a new repository named what you like.
2. Make a new directory wherever you like to put your code projects locally
```
mkdir your-new-Dir
cd your-new-Dir
```
3. Now set up your origin, make an empty commit and push it up, setting your upstream branch as master.
```
git init
git commit --allow-empty -m "my first commit, *yay*"
# Use your repo's origin URL here
git remote add origin git@github.com:WS/git-name-origin.git
git push -u origin master
```
4. Run:
```
npm init
```
5. Create an index.js file
```
var express = require('express');
var app = express();
app.get('/', function (req, res) {
  res.send('Hello World!');
});
app.listen(3000, function () {
  console.log('Example app listening on port 3000!');
});
```
6. NPM install express
```
npm install express --save
```
Add a .gitignore file so that we don’t check in the node_modules directory. .DS_Store files always get added to directories by OSX, they contain folder meta data. We want to ignore these too.
```
node_modules
.DS_Store
```
Now add all your code and push it up
```
git add .
git commit -m "Ze server."
git push
```
Now we need to pull the code into the server.
We need to SSH into the server, generate a SSH private/public key pair and then add it as a deployment key in source control (i.e. Github). Only when the server is allowed access to the remote repo will it be able to clone the code and pull down changes.
7. SSH into your server and generate the key pair.
```
# When prompted, use the default name.
# No need for a pass phrase.
ssh-keygen -t rsa
```
8. Show the contents of the file
```
cat ~/.ssh/id_rsa.pub
```
Select the key’s contents and copy it into Github. Deploy keys are added in a section called Deploy keys in the settings for your repo.

9. Paste your key and call it something meaningful.
Whenever you are logged in over SSH, you want the keys to be added so that they are used to authenticate with Github. To do this, add these lines to the top of your ~/.bashrc file.
```
# Start the SSH agent
eval `ssh-agent -s`
# Add the SSH key
ssh-add
```
This will make sure you use the keys whenever you log on to the server. 
10. To run the code without logging out, execute the .bashrc file
```
source ~/.bashrc
```
11. Now we can clone the repo! Remove any previous code on the server and in the user directory, clone the repo
```
# You should use your own git URL.
git clone git@github.com:WS/git-name-origin.git
```
We want to completely avoid ever using SSH. For deployment, we are going to use PM2 in order for us to do the git cloning on the server.
12. Before using PM2, remove the code you just pulled in from git into your server.
```
rm -rf ~/tutorial-pt-2
```
13. While you are still in the SSH session, ensure that there are no processes still running on PM2, if there are then remove them.
```pm2 ls
# Only do this if a task is still running
pm2 delete your-custome-name
```
14. In your local version of the project, install PM2 globally
```
npm i -g pm2
```
Now we need to add a config file PM2 can read so that it knows how to deploy.

15. The config file should be named ecosystem.config.js and should look like this
```
module.exports = {
  apps: [{
    name: 'your-custom-name',
    script: './index.js'
  }],
  deploy: {
    production: {
      user: 'ubuntu',
      host: 'your-aws-public-dns',
      key: '~/.ssh/your-key.pem',
      ref: 'origin/master',
      repo: 'git@github.com:ws/your-custom-name.git',
      path: '/home/ubuntu/your-custom-name',
      'post-deploy': 'npm install && pm2 startOrRestart ecosystem.config.js'
    }
  }
}
```
16. Once the file is saved, setup the directories on the remote
```
pm2 deploy ecosystem.config.js production setup
```
17. Once setup, commit and push your changes to Github so that when it clones it gets your ecosystem.config.js file, which is going to be used to start your app using PM2 on the server.
```
git add .
git commit -m "Setup PM2"
git push
```
18. Now you can run the deploy command
```
pm2 deploy ecosystem.config.js production
```
This should come up with an error, which will be that npm was not found.
19. SSH into your server and open up the ~/.bashrc file. The code that excludes non-interactive sessions is near the top.
```
# If not running interactively, don't do anything
case $- in
  *i*) ;;
  *) return;;
esac
```
20. Move the NVM code above this code, so it always executes. Find the following lines and move them above the case statement
```
export NVM_DIR="/home/ubuntu/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"  # This loads nvm
```
21. Save and exit.
22. Back on your local terminal, try running the PM2 deploy again
```
pm2 deploy ecosystem.config.js production
```
It should work this time! And your server should still be running when you check in a browser.
Using a global PM2 on the server and a global PM2 on the client is a bit messy. It would be better if our code used the local version of the PM2 package. To do this, add a deploy and restart script to your package.json
```
...
  "main": "index.js",
  "scripts": {
    "restart": "pm2 startOrRestart ecosystem.config.js",
    "deploy": "pm2 deploy ecosystem.config.js production",
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "repository": {
...
```
23. Install PM2 locally and save, using --save-dev
```
npm i pm2 --save-dev
```
24. Before deploying, commit all your changes and push to git.
25. When you run npm deploy, it will now use the local version of PM2.
```
npm run-script deploy
```
26. Make sure the app restarts when the server restarts, SSH back into your server and run
```
pm2 save
```
The only PM2 process running should be your deployed server, so PM2 will ensure that is always kept running.

## Serve some HTML
1. Remove the route handler for / and set up a static directory that express will use to serve static files.
```
var express = require('express')
var app = express()
app.use(express.static('public'))
app.listen(3000, () => console.log('Server running on port 3000'))
```
By calling express.static with “public”, static files will be served from the public directory. If we add an index.html in /public, then this will be automatically served at / by express (if we left a custom / handler, then that would override this functionality).
2. Save a HTML file in public/index.html
```html
<!DOCTYPE html>
<html>
  <head>
    <title>Erm hey.</title>
  </head>
  <body>
    <p>Something</p>
  </body>
</html>
```
3. Commit these changes (perhaps you could check the server still works by running node index.js locally and visiting http://localhost:3000).
4. Deploy the changes
```
npm run-script deploy
```
Your server will now respond with your static HTML page.


## Install Node-Red on AWS EC2 instance
1. From AWS EC2 console got to ‘Configure Security Group’ tab, add a new ‘Custom TCP Rule’ for port 1880 and save.
2. SSH into your server from terminal
3. Run
```
sudo npm install -g node-red
```
4. Test your instance by running 
```
node-red
```
5. Once started, you can access the editor at http://<your-instance-ip>:1880/.
6. To get Node-RED to start automatically whenever your instance is restarted, you can use pm2:
```
pm2 start `which node-red` -- -v
pm2 save
pm2 startup
```
