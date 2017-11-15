# Creating the Angular 4 web host
Angular 4 is a great framework for building robust testable applications and the Angular cli provides a tool kit for scaffolding, testing, and running your application locally. Eventually you will need to deploy your application outside of your local dev environment, and to do this we will need some additional tools. 

One of the main strengths of Angular's single page application design is that it is relatively straight forward to host through Firebase or AWS or on your own webserver of choice. But we'll be looking at a different approach for hosting, via a [Docker container](https://www.docker.com/what-container). Docker offers lightweight linux containers which can run anywhere using a Docker Host VM. This makes for deployments that are consistent, portable and repeatable. In fact you can deploy a container from an image or script to all major cloud platforms as well as continuous integration platforms such as [CircleCI](https://circleci.com/).

To that end let's see what a basic workflow for "Dockerizing" an Angular app might look like.

# Building the angular app for distribution with the container

The first step is the easy part. If you've worked with Angular CLI before this should be pretty familiar.

Install our sample app's dependencies (you can use NPM also, I prefer Yarn)
``` bash
  $ yarn install
```
Build the app for inclusion in our container, you might use a specific build flag for the environment your build is targeting, ie production, staging, etc. 
``` 
  $ ng build
```

Once ng build completes you should have a directory called 'dist' at the root of your Angular app folder. We're now ready to move on to configuring Httpd and building our container.

# Configuring our Httpd

Apache is a the most common webserver available, it is well supported, open source and free to use, which makes it a great choice for hosting our angular app. Apache Httpd has an [officially supported docker image](https://hub.docker.com/_/httpd/) which  we will use as the basis of our container. This saves us the step of installing Apache and it's dependencies on a baseline container and insures that only the minimum necessary dependencies are installed on our container.

To that end we will need to create 2 configuration files, the httpd.config which contains the general configuration data for our server and a .htaccess file which will declare specific request rewrite rules we need to serve the Angular app properly.

Angular supports two URL styles HTML5 pushState for modern browsers and Hash URL for legacy browsers. More info on Angular Route style support [here](https://angular.io/guide/router#appendix-locationstrategy-and-browser-url-styles). By default Angular's Router is configured for HTML5 PushState, this is great because it will allows use to do server side rendering later if we need to. However we will need to modify our configuration of Httpd to support deep linking any potential location in our app with out triggering a 404 error. 

In order for our Httpd instance to properly support our Angular app we must catch requests that would trigger a 404 error and instead server Index.html, the root of our webapp. To do this we're going to need to enable Apache's mod_rewrite and configure an .htaccess file in our HTTP root, to make sure we serve the correct file no matter what link is requested. 

# .htaccess 
```
  RewriteEngine on
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d

  # don't rewrite requests for css, js and images
  RewriteCond %{REQUEST_URI} !\.(?:css|js|map|jpe?g|gif|png)$ [NC]
  
  # rewrite everything else to index.html
  RewriteRule ^(.*)$ /index.html?path=$1 [NC,L,QSA]
```

To utilize our rewriting rules we will need to enable mod_rewrite in our httpd.config. Locate and uncomment the following line.

```
  LoadModule rewrite_module modules/mod_rewrite.so
``` 

Additionally we will need to set the default rules for our Document Root. Find the DocumentRoot and set accordingly.
```
  DocumentRoot "/usr/local/apache2/htdocs"
  <Directory "/usr/local/apache2/htdocs">
    Options Indexes FollowSymLinks
    AllowOverride All
    Require all granted
  </Directory>
```

# The Container

Now that we have our compiled app and the necessary configuration files to run our Apache web server we can configure the container which will run it.

Creating a Docker container consists of 2 main steps. Building the container image, and running a container instance. The build step is where we specify what the base container image will be, and what modifications should be made to the image before it is run. 

Building a docker instance is simple, we'll start by creating a file called Dockerfile at the root of our application directory (for security reasons docker does not have access to files outside of it's root directory). We'll take the docker file line by line. 

First we'll define the container image from which we want to base our container. Dockers layered file system allows us to derive a new image from the specified base image. In this case we will use the official [Apache httpd:2.4 image](https://hub.docker.com/_/httpd/). Which contains all the required dependencies to run httpd.
```
  FROM httpd:2.4
```

Next we'll copy our Angular application and .htaccess to the HttpRoot of our server. The COPY command allows us to copy files from our local file system to our containers file system.
```
  COPY ./sample-app/dist/ /usr/local/apache2/htdocs/
  COPY ./.htaccess /usr/local/apache2/htdocs/
```

Last we'll copy our custom httpd.config settings into our container at the path that httpd expects it.
```
  COPY ./httpd.config /usr/local/apache2/conf/httpd.conf
```

Note: by default Containers will not retain changes to their filesystem when stopped. Only files copied to the image pre-run or mounted in volumes will be present when the container starts. 


Now that we've configured our image. Docker offers two paths to go about building and running an image as a container and we'll briefly look at them both here.

## Building the image and running the container with the docker command.

Now we're just two commands away from having a running instance of our server. First we'll need to build our container. From our repo root we can call docker build like this.
``` 
  $ docker build -t ng_httpd .
```

This command is fairly straight forward, the -t (--tag) flag lets you set the name and tag for the image and it takes a single argument of path containing a Dockerfile.

You should see docker working in the terminal, what it's doing is downloading all the dependent images that our new image will be based on, as mentioned previously the layered filesystem, allows each image to be comprised of it's changes to it's parent image. This helps keep image sizes small and allows us to utilize caching to speed up our container builds.

Once docker build is completes the only thing left to do is run our image as a container, with the docker run command. Docker run offers a variety of command flags that allow us to set important options at run time. Let's take the following command.

```
  $ docker run -dit --name sample-ng-app  -p 8080:80 -v "$PWD/sample-app/dist":/usr/local/apache2/htdocs ng_httpd
```

Here is a brief rundown of the important flags. 

-d runs the container in daemon mode. 
-i runs the container as interactive, keeping STDIN open.
-t allocates pseudo-TTY
--name flag will set the name the container will run as, if omitted docker will come up with a clever unique name.
-p maps our outside port (8080) to the internal port our process is listening on. By default docker containers expose no ports.
-v mounting a volume containing our application, this will essentially simlink this directory over the containers internal file system. Changes made to files inside the volume will be persisted between container runs, as they exist outside the containers file system.

The container also takes the name or hash of the image we wish to run as an argument, in this case 'ng_httpd' the image we created in the last step.

After running our container we can easily check on the process with docker ps. 
```
$ docker ps

CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS              PORTS                    NAMES
539dabe9564c        ng_httpd            "httpd-foreground"       5 seconds ago       Up 4 seconds        0.0.0.0:8080->80/tcp     sample-ng-app
```

## Building the container
We have two possible options for building and running our container, either building and running out container with the docker command, or using a docker-compose.yml file and docker-compose to script the build. Either option will achieve the same outcome, docker-compose is handy in cases where we need to orchestrate several linked containers with various container settings that would be cumbersome to notate in a sequence of docker commands. Docker compose takes case of ensuring that each containers dependencies are brought up before running the container. You can read more about docker-compose [here](https://docs.docker.com/compose/overview/).

With docker-compose a single command and our container will be up and running and serving our app.

# Boot strapping the container with docker-compose
``` 
  $ docker-compose up -d
```