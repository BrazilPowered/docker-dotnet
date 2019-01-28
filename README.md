# Introduction 
This repo is copied from https://github.com/dockersamples/global-2018-hol/tree/master/mta-dotnet as of 12/2018.

It's purpose is to provide a solid application to experiement with .NET inside Jabil. 
These instructions are modified from Docker's original format to fit Jabil screens.

# Getting Started
To get started, please complete the Docker Training course provided by the Developer Services Team. 
While not required, it is strongly recommended.

In order to perform this lab, you will need to:
1.	Administrator rights on your machine
2.  Install Docker-Desktop for Windows
3.	Git for windows
4.	A non-IE web-browser (optional, but recommended)
5.	A choice Code Editor (Visual Studios Code is Recommended)
6.  A Powershell window (run as Administrator)

# Moving .NET/Windows Applications to Docker

You can run full .NET Framework apps in Docker on Windows containers. 
This hands-on lab shows you how to take an existing ASP.NET WebForms app, 
run it in Docker and then modernize the app by breaking features out and 
running them in separate containers.

The app you'll be using is modified from Docker's own video series 
[Modernizing .NET Apps for Developers](https://blog.docker.com/2018/02/video-series-modernizing-net-apps-developers/). 
This lab will get you experienced in using Docker to modernize traditional 
.NET applications, and then you can watch the video series for more in-depth walkthroughs.

> If you aren't familiar with the basic concepts of Docker, 
you can catch up by reviewing some of the self guided labs 
on the [Developer Services Site](https://jabil.sharepoint.com/sites/IT/DeveloperServices/Pages/Docker.aspx), 
or by reaching out to the Jabil Developer Team directly at 
[Docker@jabil.com](mailto:Docker@jabil.com).

## Table of Contents

* [Task 0: The Play with Docker Lab Environment](#0)
* [Task 1: Running the app in a container](#1)
* [Task 2: Deploy using Docker-Compose](#2)
* [Task 3: Fixing a performance issue](#3)
* [Task 4: Replacing the homepage](#4)
* [Task 5: Push Images to Docker Trusted Registry](#5)
* [Task 6: Deploy on Universal Control Plane](#6)

## <a name="0"></a>Step 0: The Docker EE Lab Environment

Login to the Docker Enterprise Edition session at the URL provided by your workshop organizer. 
This is a hybrid cluster, with Linux and Windows nodes. This workshop is designed for 
people attending an in-person workshop, but Steps 1-5 of this lab can be completed on your local 
machine, and then later pushed to Jabil's Docker Environmentvfor Step 6.

The left-hand navigation lists all the features avaiable to this UCP. Click on 'Shared Resources' 
and then 'Nodes' to see all the environments in this Swarm. You should see a list of hostnames, 
with the corresponding role (of manager or worker), and with at least 1 worker that has Windows 
under OS/Arch. This is required for Step 6 to work. That will be the Windows worker node on the 
Swarm where this app will run when complete.

![Windows terminal in Play with Docker](./images/pwd-windows-terminal.jpg)

For this lab, we will need to use Powershell. Make sure Docker is running on your environment- 
it runs as a background Windows Service:

```powershell
Start-Service docker
```

If you have used this environment for other labs, first remove any existing containers:

```.term1
docker container rm --force $(docker container ls --quiet --all)
```

> Disregard any error stating that "docker container rm requires at least 1 argument" this time around. In this particular case, it means you have no containers on the system.

To start with you'll work with the Windows node directly, but at the end of the lab you'll deploy to the cluster to see how to run applications in production with Docker Enterprise.

## <a name="1"></a>Task 1: Running the app in a container

You're going to start with an ASP.NET 4.7.2 WebForms app which is meant to be built in Visual Studio and run in IIS. Rather than set up a unique environment, we will use Docker to build and run the app in a container.

You can package Windows server applications to run in Docker using existing deployment tools (like MSIs), or from source code. In this task you'll compile and package an ASP.NET app from source using Docker.

Start by cloning the Git repository for the sample app:

```.term1
cd ~/Documents

git clone https://github.com/BrazilPowered/docker-dotnet

cd docker-dotnet
```

## Task 1a: Build your Web-App Image

Your first task will be to build an image for our web app ui. Let's use the following requirements
1.  Place the Dockerfile in /docker/web
2.  Use a Multi-Stage Build
3.  Use the dotnet-framework version 4.7.2-sdk as the Base Image for your build-server image
4.  In each stage, define "powershell" as your default SHELL using the following args in exec form: "-Command", "$ErrorActionPreference = 'Stop';"

For your Build-Server stage:
1.  Use the "C:\src" folder as your container's Working Directory
2.  Copy the src\SignUp folder to the Working Directory on the Container Image
3.  Run the build.ps1 script (this will compile the app)

For your App Image stage:
1.  Use aspnet version 4.7.2-windowsservercore-ltsc2016 as the base image for your App server
2.  Use the "C:\web-app" as your container's Working Directory
3.  Remove the IIS Default Web Site with the following command: Remove-Website -Name 'Default Web Site';
4.  Add your website as the new IIS web-app on port 80 with the following command: New-Website -Name 'web-app' -Port 80 -PhysicalPath 'C:\web-app'
5.  Copy the compiled files from your build-server stage into the working directory of this container image. They are located at "C:\out\\_PublishedWebsites\SignUp.Web"

## Task 1b: Set up your Database

This version 1 uses containers for a SQL Server database and the ASP.NET app. You can set this up by building a Docker image from a Dockerfile that includes your passwords, your DB init or import scripts, and defining the volume that will store your data (remember, data in a container is not persistent sio we MUST use a Voloume for DB data).

Notice that we are using two separate images for the web app and DB services. It is important to keep as many part of your application that do different types of tasks as separate as possible. Since unrelated tasks will no longer be stuck to eachother this way, the app is kept cleaner and the maintenence made much easier.

First let's place the DB Dockerfile in docker/db, and fill it with this content:

```Dockerfile
# escape=`
FROM microsoft/mssql-server-windows-express:2016-sp1
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop'; $ProgressPreference = 'SilentlyContinue';"]

ENV ACCEPT_EULA="Y" `
    sa_password="DockerCon!!!" `
    DATA_PATH="C:\mssql"

VOLUME ${DATA_PATH}

WORKDIR C:\init
COPY . .

CMD ./init.ps1 -sa_password $env:sa_password -Verbose

HEALTHCHECK CMD powershell -command `
    try { `
     $result = Invoke-SqlCmd -Query 'SELECT TOP 1 1 FROM Countries' -Database SignUpDb; `
     if ($result[0] -eq 1) { return 0} `
     else {return 1}; `
} catch { return 1 }
```

> Notice the "# escape=`" at the top. For windows files, this eliminates the need for using double backslashes (\\) for every single pathname. But this *alone* MUST be the very first line of the Dockerfile in order to work

This Dockerfile pulls 8 layers:
1.  The base image uses mssql-server-windows-express version 2016-sp1
2.  It sets the execution environment to be powershell, enabling special functionality like a "SilentlyContinue" to the SQL-Server setup
3.  It creates 3 environment variables: 1 meets the requirement of accepting an EULA, 2 sets the DB password, and 3 sets the path all the SQL server data will be stored, which is important for when:
4.  It defines a volume to be at a path pulled from that same path in the ENV set above, keeping all the DB data persistent
5.  It sets a working directory for copying and executing all the files and scripts for this image
6.  It copies the files in the docker/db folder to the working directory in the container (".")
7.  It executes the init.ps1 script with an array of required arguments
8.  And finally sets a "Healthcheck" to provide a way for ther Docker engine to know if the DB service fails. This one returns healthy *if* one or more results are returned by the DB query. the Countries table MUST return values for this web-app to work, so we know it is unhealthy if $result isn't 1.

Healthchecks are vital to your application's lifecycle in a production container environment. While Docker can detect when your container fails, it doesn't know how your app works; so if your web application has an error that kills your user access, but leaves the server up and running, Docker will keep the container running in a 'healthy' state. To prevent this problem, Healthchecks can be used to alert Docker when normal functionality is broken, giving you the chance to enable automatic actions, or receive messages yourself. Never deploy to PROD without a solid healthcheck that you understand.

> Let's take a minute to notice that the Windows machine you are using in this lab doesn't need to have SQL Server, Visual Studio or even MSBuild installed - every step in the build process happens inside containers, using images which are packaged with the build toolchain.

#### Examine your web-app Dockerfile
While the images are building, have a look at the Dockerfile for the Web application in "docker/web". You'll see there are two stages. The first stage compiles the application using MSBuild:

```Dockerfile
FROM microsoft/dotnet-framework:4.7.2-sdk AS builder
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop';"]

WORKDIR C:\src
COPY src\SignUp .
RUN .\build.ps1
```

This `builder` stage uses the `microsoft/dotnet-framework:4.7.2-sdk` image, which has the entire .NET 4.7.2 Software Development Kit toolchain needed to compile an ASP.NET 4.7.2 application (including MSBuild, NuGet, and the Web Deploy packages). It is made specifically for cases like our Multi-Stage build.

>Remember, we don't always need everything used to compile an application when we run an application. A multi-stage build allows us to separate the build process from the running application. Using this method allows our PROD container can be much smaller & more efficent, having only the minimum resources needed to run your app with maximum performance.

The `builder` stage Dockerfile copies the source code into the image and just runs the existing <a href="https://github.com/BrazilPowered/docker-dotnet/blob/1-imagebuilding/src/SignUp/build.ps1" target="_blank">build.ps1</a>) script. When this stage completes, the output is a published website folder, which will be available for later stages to use. 

The second stage uses the `aspnet:4.7.2-windowsservercore-ltsc2016` image as the base, which is a Windows Server Core image with IIS and ASP.NET 4.7.2 already configured:

```Dockerfile
FROM microsoft/aspnet:4.7.2-windowsservercore-ltsc2016
SHELL ["powershell", "-Command", "$ErrorActionPreference = 'Stop';"]

WORKDIR C:\web-app
RUN Remove-Website -Name 'Default Web Site'; `
    New-Website -Name 'web-app' -Port 80 -PhysicalPath 'C:\web-app'

COPY --from=builder C:\out\_PublishedWebsites\SignUp.Web .
```

The `RUN` command sets up the website using PowerShell. The `COPY` instruction copies the published website from the builder stage into your new container.

Again, you don't need Visual Studio or IIS installed on your own machine to run the web app in a container, nor do you need SQL Server installed to run the application database. It will all be done using only Docker.

When the build has finished, we will want to deploy the application using Docker Compose.

Make sure you are in the root `~/Documents/docker-dotnet` directory you pulled from github and then go ahead and build both those images.:

```s
cd ~/Documents/docker-dotnet

docker image build -t signup-db:v1 docker/db

docker image build -t signup-app:v1 -f docker/web/Dockerfile .
```

> Note: The second build command above uses the `-f` (`--file`) option, pointing to the Dockerfile at the location shown thereafter. This is because the Dockerfile being used needs the local context (the `.` at the end) in order to send files from both the directory after `-f` as well as the `src` folder in the top directory.

##  <a name="2"></a>Task 2: Deploy using Docker-Compose

If we want to stand up a single container, it is easy to execute a `docker run` command with the options we learned will set it up in our environment. But to work in tandem with another app, we would need to *manually* declare the appropriate networks, volumes, etc for EACH container EVERY time... Let's save ourselves that hassle with docker-compose.

We need to write a docker-compose file (remember: it's a YAML file) with the following requirements:

1.  Use compose version 3.3
2.  Include 2 services, named signup-db and signup-app.
3.  The signup-db service should have 1 network, app-net, and use the image we can build with the SQL-Server DB Dockerfile
4.  The signup-app service also needs the app-net network, but will expose port 80 to the outside network (on port 80). The image here should be the one we can build from the docker/web Dockerfile.
5.  The signup-app service should also use the "depends_on" parameter to make sure the signup-db is started first, to make sure the connection strings will find the db.
6.  The app-net network should be declared, and it should be configured to use the existing external network which is already named "nat"

### How should the Docker-Compose file look?
Here's what the docker-compose fole should look like when complete:

> PLEASE ATTEMPT THIS EXERCISE ON YOUR OWN USING THE REQUIREMNTS ABOVE BEFORE REVIEWING THE FILE BELOW

```yml
version: '3.3'

services:
  signup-db:
    image: signup-db:v1
    networks:
      - app-net

  signup-app:
    image: signup-app:v1
    ports:
      - "80:80"
    depends_on:
      - signup-db
    networks:
      - app-net

networks:
  app-net:
    external:
      name: nat
```

Let's deploy the application using Docker Compose:

```s
docker-compose up -d
```

Docker Compose will start containers for the database and the web app. The compose file configures the services using the database image and the application image you've just built.

When the `docker-compose` command completes, your application is running in Docker, in Windows containers, on the Windows node. You can see all the running containers:

```s
docker container ls
```

The HTTP port for the web container is published so the app is available to the outside world.

> To see the app running, navigate to your IP address in a local web browser. Since the container was mapped to port `80`, we won't have to specify a port number.)

![PWD Session Information](./images/windows-session.jpg)

The application is a newsletter sign-up app for Play with Docker. It will take a little while to load because the app uses Entity Framework and on startup it deploys the database schema. The home page has some basic information and link buttons. Click the _Sign Up_ button and you'll see a registration form:

![PWD newsletter sign up page](./images/part-1-signup.jpg)

> Don't worry, the data you use for this lab only goes to your local DB container, and is never shared anywhere else. It will all be gone once you end the lab and clean up your environment.

Go ahead and fill in the form. When you click _Go_, the data is saved to SQL Server running in a container. The SQL Server container doesn't publish any ports, so it's only accessible to other containers and the Docker API. 

Check that your data is stored by running a PowerShell command in the Windows terminal:

```s
docker container exec app_signup-db_1 powershell -Command "Invoke-SqlCmd -Query 'SELECT * FROM Prospects' -Database SignUpDb"
```

This executes a SQL query inside the SQL Server container, using the `Invoke-SqlCmd` cmdlet which is installed in the image.

The output is the result of the query - you'll see a row for each time you submitted the form, something like this:

```
ProspectId          : 1
FirstName           : Elton
LastName            : Stoneman
CompanyName         : Docker, Inc.
EmailAddress        : elton@docker.com
Role_RoleCode       : DA
Country_CountryCode : GBR
```

> The app is running fine in Docker, with no code changes from the original ASP.NET 4.7.2 codebase. The Dockerfile has all the logic to build and package the app, so any developer (and all the CI servers) can build and run the app from source - the only dependency is Docker.

Next you'll go on to modernize the app, fixing some issues with the current architecture.
