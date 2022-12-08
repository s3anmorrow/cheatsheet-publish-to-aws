**Publishing MERN Stack Web App to AWS**  
**[From Scratch]**  
**Cheat Sheet**  
**PROG3017 Full Stack Programming**

Publishing MERN Stack web apps is complicated. They require two servers (Express / MongoDB) running on top of a host server. As a result, they need to be heavily configured. 

In this course, we will be exploring Amazon Web Services (AWS) for publishing. AWS is a collection of remote computing services, or web services, offered by Amazon to handle tasks related to cloud computing, storage, databases, etc. The setup is complicated – but luckily, we have a secret weapon…Docker!

With Docker, we can spin up all required containers (and thus servers) on an AWS EC2 instance without having to spend time on configuration. 

This cheat sheet will walk you through the process.

*Note the approach taken below is very manual and not the most efficient deployment (due to account limitations). A more automated approach is using Amazon’s ECR (Elastic Container Registry) which is a service specifically for deploying docker driven web apps.*

1) Login to AWS EC2 Dashboard
    - Each student has already received an invite from AWS Academy
    - Login to AWS Academy and open the “2022-2023 AWS Learning Space” course
    - Click the “Modules” link in the side navigation bar
    - Click the “Learner Lab - Foundational Services” link
    - Click the “Start Lab” play button to start up your AWS lab – it may take some time to startup. Note that you only have a certain amount of time to play with AWS before your session expires. Your data is not lost when the session runs out, but it will shut down all your EC2 instances.
    - Once the lab is started you can click the “AWS” link to go to the “AWS Management Console”

2) Launch EC2 Instance (Amazon Elastic Compute Cloud)
    - EC2 provides a computing service for handling computational tasks. In our case, we’ll be using EC2 to run the servers which will host our web application. In other words, EC2 is a virtual machine running your server, etc. (spins it up)
    - AWS refers to these virtual machines as "instances"
    - On the console home page, click “View All Services” link and click “EC2”
    - Click "Launch Instance" button from EC2 Dashboard to take you to the EC2 setup page

3) Setup the EC2 Instance by filling out the individual sections on the page
    - Give the instance a name
    - Select an AMI (Amazon Machine image) to base the EC2 instance on
        - While there are prebuilt MERN stack AMIs, it is good practice to set one up from scratch using our docker containers. Select “Amazon Linux” (free tier eligible) from the Quick Start menu
    - Create your Key/Pair
        - to launch your instance, you need to create an EC2 Key Pair
        - Basically, key pairs enable you to login to EC2 instances without a username / password. Also known as an SSH key
        - Click the "Create a new key pair" link
        - give it a name like "PROG3017-AWS-Key" and leave all else default
        - Click the “Create Key Pair” button to download the .pem file with the same name. Don’t lose this file! It’s only generated once and you’ll need it to access your EC2 instance.
    - Click the “Launch Instance” orange button to spin up the EC2 instance – wait for it to start running before continuing

4) Connecting to the EC2 Instance with SSH and key pair
    - click on the running EC2 instance and click the connect button for detailed instructions on how to connect to your server via SSH
    - open a terminal (or windows bash / or putty) in same folder as the .pem key pair file and run commands:  
        ```
        chmod 400 PROG3017-AWS-Key.pem
        ```
        ```
        ssh -i "PROG3017-AWS-Key.pem" ec2-user@[public DNS of EC2 instance]
        ```
    - the SSH command above can be found by clicking link for EC2 instance and clicking the connect button. Look for SSH client and copy the provided example
    - access is now granted and you will see a flashing cursor – you can now run commands on your instance on AWS!

5) Thanks to docker, we don’t need to manually install all the dependencies for our app, but we do need to install Docker and Git:
    - Will use YUM which is a packaging tool much like NPM – but first make sure it is updated. Run command:
    sudo yum update -y
    - Install docker with commands:
        ```
        sudo amazon-linux-extras install docker
        ```
        ```
        sudo usermod -aG docker $USER
        ```
        ```
        newgrp docker
        ```
    - Install docker-compose with commands:
        ```    
        sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose
        ```
        ```
        sudo chmod +x /usr/local/bin/docker-compose
        ```
    - Install git:
        ```
        sudo yum install git
        ```

6) There is some configuration that needs to be done to the web app before deploying to the EC2 instance. Luckily, with docker this can all be tested locally.
    - During development we have three containers spinning up: 
        - Container for the Client-Side Web App server (React)
        - Container for the Web API (Express server with Node.js) 
        - Container for the MongoDB server

    - For production we need the Express server to serve up the client-side web app and thus drop our container count to two. Luckily, we have already explored this in a previous lesson and the required production Docker files are already included in the boilerplate project folder. 

    - However, docker-compose-prod.yml does need to be tweaked:
        - Uncomment the two commented MongoDB sections
        - Under the Mongo service, set the name of the database to be the same as defined in Server.js
    environment:
            ```
            MONGO_INITDB_DATABASE: [Change this to DB name!]
            ```
    - Our React app needs to send out an AJAX request to the express server running on the same server – not an entirely new server like during development. As a result, we need to change all URLs in our React web app to be relative. For example, modify App.tsx to send request to a relative URL of AWS server:
        ```
        // const RETRIEVE_SCRIPT:string = "http://localhost/get";
        const RETRIEVE_SCRIPT:string = '/get';
        ```
    - Start up docker containers locally with command (while in project folder root):
    ```
    docker-compose -f docker-compose-prod.yml up --build
    ```
    - Test the web app by hitting http://localhost

7) Copy project folder to EC2 instance
    - Create a private repo of your web app on Github and push your web app to that repo. Don’t forget to delete the .git folder and reinitialize a new repo with the remote set to your own GitHub account
    - In order to clone this webapp in the EC2 instance you need to have a Personal Access Token (PAT). Github no longer supports passwords for cloning private repos
    - Login to github.com and go to Settings / Developer Settings / Personal Access Tokens / and press “Generate New Token” button. Save token string in a text file for safe keeping
    - From the SSH terminal, clone your web app into EC2 instance with command:
    git clone <HTTPS of GitHub Repo>
    You will have to enter in your Username / PAT (when it asks for password)
    If any changes are made, use “git pull” to update this project folder
    - Move into the project folder with cd command

8) Start up docker service with command:
    ```
    sudo service docker start
    ```
    Setup docker to always fire up when the instance starts with command:
    ```
    sudo chkconfig docker on
    ```

9) Setup Security Group and Startup Express Server (Node.js)
    - because our express server uses port 80 and the client side is also served from it, we need to open up the port
    - in AWS EC2 dashboard - click on instance / security tab – click on the link to go directly to security group settings
    - click Edit Inbound Rules / Add Rules button and select “Custom TCP” from the dropdown with the following settings:
        HTTP / 80 / 0.0.0.0/0
    - save rules

10) Build and spin up docker containers with command:
    ```
    docker-compose -f docker-compose-prod.yml up --build
    ```

11) Hit the web app with a browser
    - Use the public DNS URL outlined in EC2 instance dashboard – for example:
    http://ec2-18-207-204-246.compute-1.amazonaws.com
    - Note that our web app does not run on “https”

Other Notes:
- Stopping your EC2 instance in the AWS dashboard will shut it all down but will also assign a different public DNS (URL) to it when you restart the instance
- Any changes made to your web app requires:
o	Shut down docker container(s) with CTRL+C
o	Commit to Git locally and push to Github
o	Pull from Github to project folder in EC2 instance
o	Re-build of docker image (will be faster than first build)
o	Restart the docker containers

References:
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html
https://gist.github.com/npearce/6f3c7826c7499587f00957fee62f8ee9
