

Scaling Shiny apps in Docker
================

In this tutorial, we are going to create a Docker image for your Shiny App and upload it to Google Cloud without dying trying.

## Intro

Sometimes the standard version of the Shiny free servers are not enough to deploy our beautiful creations with R.
The most annoying limitation is the 5 active instances limit. 
For test environments maybe it wasn't bad, but once the
application reaches more professional areas where probably more
people use the app at the same time is more difficult.
In this short tutorial we will take a Shiny application to a
Docker repository and from there we will deploy to a service of
Google Cloud with more RAM and CPUs available.

## Ingredients

-   A Shiny app, .Rmd, flexdashboard (as in this example),
    etc.
-   Docker and a Docker account to upload the image to a repository.
-   Google Cloud account and active basic APIs (the process is
    quite intuitive).

## Installing docker

To install Docker, the file is obtained from the official page and
install normally. On OSX, as is my case, after downloading the
.dmg is installed as follows from the terminal:

``` bash
sudo hdiutil attach Docker.dmg
sudo /Volumes/Docker/Docker.app/Contents/MacOS/install
sudo hdiutil detach /Volumes/Docker
```

## Creating the Dockerfile

The Dockerfile is a set of lines of text that configure
the packages that Docker is going to use to generate the environment of our
application and it is where we will also give “commands” to execute the app.  

In my case, I created a folder called "Docker" with two elements:
a folder called “R” where everything related to the application is
Shiny and the “Dockerfile” file itself (so, without dots, or extensions,
no weird stuff).  

In my case, the Dockerfile contains the following lines,
commented point by point to better understand:  

``` bash
FROM rocker/shiny:latest # Shiny and R environment are installed

RUN apt-get update -qq && apt-get -y --no-install-recommends install \ # Installing dependencies
    libgdal-dev \ # They quite depend of the packages you are using in the app
    libproj-dev \ # 
    libgeos-dev \
    default-libmysqlclient-dev \
    libmysqlclient-dev \
    libudunits2-dev \
    netcdf-bin \
    libxml2-dev \
    libcairo2-dev \
    libsqlite3-dev \
    libpq-dev \
    libssh2-1-dev \
    unixodbc-dev \
    libcurl4-openssl-dev \
    libssl-dev
    
RUN apt-get install pandoc # I had some Pandoc problems but this line fixed it

RUN apt-get update && \       # Updating packages and cleaning
    apt-get upgrade -y && \
    apt-get clean

# Installing R packages
# notice that we are already executing R COMMANDS inside Docker

RUN R -e "install.packages(pkgs=c('shiny','tidyverse',
'flexdashboard', 'sf', 'ggalluvial', 
'plotly', 'rmarkdown'), repos='https://cran.rstudio.com/')" 

RUN R -e "install.packages(pkgs=c('devtools'), 
repos='https://cran.rstudio.com/')" 

RUN R -e  "devtools::install_github('rstudio/leaflet')"

RUN mkdir /root/app # Creating the app folder

COPY R /root/shiny_save # Copies the R folder into shiny_save
 
EXPOSE 3838  #Very important: assign a communication port to the app.  
#Later we will not be able to do anything if the port is not exposed


# The app is run (in this case, being a flexdashboard
# I have to run it with rmarkdown, otherwise it would be shiny::runApp(...)).
# As an extra argument I pass the open port.

CMD ["R", "-e", "rmarkdown::run('/root/shiny_save/dash_v2.Rmd', 
shiny_args=list(host='0.0.0.0', port=3838))"] 
```

## Image building 

Once we have the dockerfile and the folder with all the files
that the app needs to run, we proceed to build the Docker image
itself. For this, we open the console and execute:

``` bash
docker login
```

It will ask us for the Docker username and password. We write it and then:

``` bash
docker build -t nombreapp .
```

That command will build an image called “appname” with the parameters
of the dockerfile (for that the period at the end). This image contains the
R environment and all the code that we have used, within a system
Linux.
This is why it probably takes a long time to run the first
time (it takes me approximately 3 hours), obviously depending on the
app complexity. If we make modifications to the R code of the
application and push the changes (executing the same parameters),
the construction takes much less time since most of the layers of the
environment are the same and are not changed. This is something that I love about
Docker. The work is divided into "layers", so if you correct a
small part of the code, the build and push of the app take seconds.

``` bash
docker tag nombreapp usuario/nombreapp:latest
```

We “tag” the app within the public repository that we are about to create.
It is like saying that we are “selecting” it to upload it. The
repository will be named user/appname. The last part “:latest” is
a name that we can put to identify the latest version of the
app. We can modify it for each version.

``` bash
docker push usuario/nombreapp:latest
```

The image is pushed to the repository, just as if it were GitHub.

## Deploy to Google Cloud

Once the project is created, we proceed to open the command console
within the same Google website. The first thing we must do is
log in with our Docker credentials.

``` bash
docker login
```

Later, we “bring” the Docker image to our project.
Google Cloud with the latest changes.

``` bash
docker pull usuario/nombreapp:latest
```

Then, analogously to what we did in Docker, the image is "tagged"
push, the only thing is that this time we don't push to the Docker repo but to the
Google Cloud image container. Replace PROJECT_ID with the ID of
your project.

``` bash
docker tag usuario/nombreapp:latest gcr.io/PROJECT_ID/usuario/nombreapp:latest
```

Finally, we push the image.

``` bash
docker push gcr.io/PROJECT_ID/usuario/nombreapp:latest
```

## Create the service instance

We are almost done. After pushing the image, in the Google tab
Cloud Run click “Create Service”. This will enable us a machine
virtual to do the deploy and that we can share the app to a lot
of people.

Where it says "Url of the image" put select and the
image options. The image “appname” will appear with the
latest version. We use that and fill in the parameters with what
need the project.

**IMPORTANT:** In the part below where it says “CONTAINER”,
Fill in the “Container port” field with port 3838 or the
that you have chosen in the construction of the Dockerfile, since without this
Google will mis-route the port generating errors.

You wait a few seconds for Google to create the instance and that's it! It's already the
app deployed to Google Cloud Run. Anyone will be able to access it
through the URL that shows you above the panel.
