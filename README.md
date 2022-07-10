

Dockerizando Shiny
================

Para crear una imagen Docker de Shiny y subirla a Google Cloud sin morir en el intento.

## Introducción

Muchas veces, la versión estándar de los servidores gratuitos de Shiny
no alcanzan para nuestras hermosas creaciones en R. La limitación que
más me molestaba era la de las 5 instancias activas simultáneamente.
Para entornos de test quizá no estaba mal, pero una vez que la
aplicación llega a ámbitos más profesionales donde probablemente más
personas utilicen la app al mismo tiempo es más difícil.

En este breve tutorial llevaremos una apliación de Shiny a un
repositorio Docker y de allí mismo haremos un deploy a un servicio de
Google Cloud con más RAM y CPUs disponibles.

## Ingredientes

-   Una aplicación Shiny, .Rmd, flexdashboard (como en este ejemplo),
    etc.
-   Docker y una cuenta Docker para subir la imagen a un repositorio.
-   Cuenta en Google Cloud y las APIs básicas activas (el proceso es
    bastante intuitivo).

## Instalando Docker

Para instalar Docker, se obtiene el archivo desde la página oficial y se
instala normalmente. En OSX, como es mi caso, después de descargar el
.dmg se instala de la siguiente manera desde la terminal:

``` bash
sudo hdiutil attach Docker.dmg
sudo /Volumes/Docker/Docker.app/Contents/MacOS/install
sudo hdiutil detach /Volumes/Docker
```

## Creando el Dockerfile

El archivo Dockerfile es un conjunto de líneas de texto que configuran
los paquetes que Docker va a utilizar para generar el entorno de nuestra
aplicación y es donde también le daremos “órdenes” para ejecutar la app.

En mi caso, creé una carpeta que se llama “Docker” con dos elementos:
una carpeta llamada “R” donde está todo lo relacionado a la aplicación
Shiny y el propio archivo “Dockerfile” (así, sin puntos, ni extensiones,
ni cosas raras).

En mi caso, el archivo Dockerfile contiene las siguientes líneas,
comentadas punto a punto para entender mejor:

``` bash
FROM rocker/shiny:latest # Se instala el entorno Shiny y R

RUN apt-get update -qq && apt-get -y --no-install-recommends install \ # Se instalan librerías necesarias 
    libgdal-dev \ # Ojo que probablemente necesites más o menos
    libproj-dev \ # depende de los paquetes que uses
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
    
RUN apt-get install pandoc # Tenía problemas con Pandoc e instalarlo por separado lo solucionó

RUN apt-get update && \       # Actualizar paquetes y limpiar
    apt-get upgrade -y && \
    apt-get clean

# Se instalan los paquetes necesarios
# notar que ya estamos ejecutando COMANDOS DE R dentro del Docker

RUN R -e "install.packages(pkgs=c('shiny','tidyverse',
'flexdashboard', 'sf', 'ggalluvial', 
'plotly', 'rmarkdown'), repos='https://cran.rstudio.com/')" 

RUN R -e "install.packages(pkgs=c('devtools'), 
repos='https://cran.rstudio.com/')" 

RUN R -e  "devtools::install_github('rstudio/leaflet')"

RUN mkdir /root/app # Creamos la carpeta de la app

COPY R /root/shiny_save # Copia la carpeta "R" a la carpeta shiny_save
 
EXPOSE 3838 #Importantísimo: asignarle un puerto de comunicación a la app.
#Después no vamos a poder hacer nada si no se expone el puerto

# Se corre la app (en este caso, al ser un flexdashboard 
# tengo que ejecutarlo con rmarkdown, si no sería shiny::runApp(...)). 
# Como argumento extra le paso el puerto abierto.
CMD ["R", "-e", "rmarkdown::run('/root/shiny_save/dash_v2.Rmd', 
shiny_args=list(host='0.0.0.0', port=3838))"] 
```

## Construcción de la imagen

Una vez que tengamos el dockerfile y la carpeta con todos los archivos
que necesita la app para correr, pasamos a construir la imagen de Docker
propiamente dicha. Para esto, abrimos la consola y ejecutamos:

``` bash
docker login
```

Nos pedirá el usuario y la contraseña de Docker. La escribimos y luego:

``` bash
docker build -t nombreapp .
```

Ese comando construirá una imagen llamada “nombreapp” con los parámetros
del dockerfile (para eso el punto al final). Esta imagen contiene el
entorno R y todo el código que hemos usado, dentro de un sistema
Linux.  
Es por esto que probablemente tarde bastante en ejecutarse la primera
vez (a mí me lleva aproximadamente 3 horas), obviamente dependiendo la
complejidad de la app. Si hacemos modificaciones al código de R de la
aplicación y pusheamos los cambios (ejecutando los mismos parámetros),
la construcción tarda muchísimo menos ya que la mayoría de las capas del
entorno son las mismas y no se modifican. Esto es algo que me encanta de
Docker. El trabajo se divide en “capas”, por lo que si corregís una
parte pequeña del código, el build y el push de la app tardan segundos.

``` bash
docker tag nombreapp usuario/nombreapp:latest
```

“Tagueamos” la app dentro del repositorio público que estamos por crear.
Es como decir que lo estamos “seleccionando” para subirlo. El
repositorio se llamará usuario/nombreapp. La última parte “:latest” es
un nombre que podemos poner para identificar a la última versión de la
app. Lo podemos ir modificando para cada versión.

``` bash
docker push usuario/nombreapp:latest
```

Se pushea la imagen al repositorio, tal como si fuera GitHub.

## Deploy en Google Cloud

Una vez creado el proyecto, procedemos a abrir la consola de comandos
dentro de la misma Web de Google. Lo primero que debemos hacer es
loguearnos con nuestras credenciales de Docker.

``` bash
docker login
```

Posteriormente, “traemos” la imagen del Docker a nuestro proyecto de
Google Cloud con los últimos cambios.

``` bash
docker pull usuario/nombreapp:latest
```

Luego, análogamente a lo que hacíamos en Docker, se “taguea” la imagen a
pushear, lo único es que esta vez no pusheamos al repo de Docker sino al
container de imágenes de Google Cloud. Reemplazá PROJECT_ID por el ID de
tu proyecto.

``` bash
docker tag usuario/nombreapp:latest gcr.io/PROJECT_ID/usuario/nombreapp:latest
```

Por último, pusheamos la imagen.

``` bash
docker push gcr.io/PROJECT_ID/usuario/nombreapp:latest
```

## Creación de la instancia del servicio

Ya casi terminamos. Luego de pusheada la imagen, en la pestaña de Google
Cloud Run clickeás “Crear Servicio”. Esto nos habilitará una máquina
virtual para hacer el deploy y que podamos compartir la app a un montón
de gente.

Donde dice “Url de la imagen” poné seleccionar y se van a desplegar las
opciones de imagen. Nos va a aparecer la imagen “nombreapp” con la
versión “latest”. Usamos esa y completamos los parámetros con lo que
necesite el proyecto.

**IMPORTANTE:** En la parte de más abajo donde dice “CONTENEDOR”,
completá dentro del campo “Puerto del contenedor” el puerto 3838 o el
que hayas elegido en la construcción del Dockerfile, ya que sin esto
Google va a enrutar mal el puerto generando errores.

Esperás unos segundos a que Google cree la instancia y listo! Ya está la
app deployada en Google Cloud Run. Cualquiera va a poder acceder a ella
a través de la URL que te muestra arriba del panel.
