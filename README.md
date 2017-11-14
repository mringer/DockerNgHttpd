# Creating the Angular 4 web host

# Building the angular app for distribution with the container

1. ''' $ ng install
2. ''' $ ng build

# Boot strapping the container with docker-compose

1. ``` $ docker-compose up -d

# Building and running httpd from the command line

1. ``` $ docker build .
2. ``` $ docker run -dit --name sample-ng-app  -p 8080:80 -v "$PWD/sample-app/dist":/usr/local/apache2/htdocs httpd:2.4


# Configuring our Httpd 

Angular supports two URL styles HTML5 pushState for modern browsers and Hash URL for legacy browsers. More info on Angular Route style support [here](https://angular.io/guide/router#appendix-locationstrategy-and-browser-url-styles). By default Angular's Router is configured for HTML5 PushState, this is great because it will allows use to do server side rendering later if we need to. However we will need to modify our configuration of Httpd to support deep linking any potential location in our app with out triggering a 404 error. 

In order for our Httpd instance to properly support our Angular app we must catch requests that would trigger a 404 error and instead server Index.html, the root of our webapp. To do this we're going to need to enable Apache's mod_rewrite and configure an .htaccess file in our HTTP root, to make sure we serve the correct file no matter what link is requested. 



1. Create a .htaccess file to 