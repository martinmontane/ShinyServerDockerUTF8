# Hostear shiny apps en servidores propios usando shinyproxy y docker

## El problema

Las shinyapps son una herramienta muy interesante para cumplir muchos objetivos. Desde un dashboard privado hasta el diseño de un simulador del COVID/19 desde el gobierno, la flexibilidad de shiny nos permite construir aplicaciones relativamente complejas con una inversión relativamente baja en otros lenguajes de programación más allá de R.

El mayor problema es que las aplicaciones de shiny se ejecutan del lado del servidor, lo que hace que haya problemas de escalabilidad o de saturación del servidor cuando múltiples procesos están corriendo. Una solución práctica a este problema es usar los servidores en [shinyapps.io](https://www.shinyapps.io/), aunque viene a un costo alto y con relativamente bajas opciones de personalización (tienen una opción de 25 horas mensuales gratuitas de actividad de la aplicación, que para algunos contextos pueden servir).

## Una posible solución: ShinyProxy

La gente de Open Analytics desarrolló una alternativa de código abierto muy interesante: [ShinyProxy](https://www.shinyproxy.io/). Lo que hace es correr containers preconstruidos de [Docker](https://www.docker.com/resources/what-container) cada vez que alguien consulta el link de las aplicaciones. Estos containers son imagenes virtuales que contienen todo lo necesario para correr las shiny apps que desarrollamos.

Este repositorio explica y brinda el armado básico para poner en funcionamiento a [nginx](https://nginx.org/en/) con shinyproxy 2.3.0 con encoding UTF-8 y conexión encriptada para que nuestro sitio sea seguro.

## Manos en el Docker

El docker-compose.yml que ven en la carpeta root de este proyecto pone en funcionamiento tres containers: nginx, nginx-gen, letsencrypt y un shinyproxy. Veamos en resumidas cuentas qué hace cada uno:

* El **nginx** y **nginx-gen** juntos nos permiten generar un nginx que funciona como reverse proxy. El objetivo es que redireccione las consultas que lleguen al servidor hacia la aplicación de shinyproxy (desarrollada en Java) ida y vuelta. Los puertos que usará este contenedor son apareados al 80 y 443 del server, como es predeterminado en nginx.
* **letsencrypt** se encarga de generar y actualizar los certificados necesarios para que la comunicación entre cliente y servidor se encuentre encriptada.
* Finalmente, **shinyproxy** pone en funcionamiento a una imagen de shinyproxy corriendo shinyproxy 2.3.0 y con UTF-8 para que se puedan usar diversos símbolos que en la versión predeterminada de shinyproxy no es posible. Shinyproxy estará en funcionamiento en el puerto 8080 por default.

Para que este docker se ponga en funcionamiento correctamente, deben modificar algunas de las lineas de acuerdo a sus necesidades. En particular, en la última parte del docker-compose, deben escribir el dominio a encriptar tanto como VIRTUAL_HOST como en LETSENCRYPT_HOST, poner un mail de reporte válido como LETSNECRYPT_MAIL y crear una red local por la cual se van a comunicar los containers. Esta red debe ser la misma que ponen en el docker-compose.yml que se muestra abajo y también en el listado de las aplicaciones en la configuración de shinyproxy (ver más abajo). Para crearlo, escriban **docker network create redshinyproxy**

```yaml
 shinyproxy:
    container_name: shinyproxy
    image: martinmontane/shinyproxy_utf8
    expose:
      - 8080
    ports:
      - 8080:8080
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock"
      - "./shinyproxy/:/opt/shinyproxy/app_yml:rw"
    environment:
      - VIRTUAL_HOST=dominioancriptar.com
      - VIRTUAL_NETWORK=redshinyproxy
      - LETSENCRYPT_HOST=dominioancriptar.com
      - LETSENCRYPT_EMAIL=maildereporte@mail.com
    networks:
      - redshinyproxy
    restart: unless-stopped
```

Por último, antes de ejecutar el comando para correr el docker, deben comunicarte a nginx que hay un dominio a encriptar como virtual host. Esto lo hacen cambiandole el nombre al archivo que hay en */volumes/proxy/vhost.d/* y dice *dominicioaencriptar.com*.

Una vez que tengan esto configurado, ya puden ejecutar docker-compose up -d para tener todos los servicios funcionando. Recuerden que tienen que asociar al dominio con la IP del servidor [acá está cómo hacerlo con Linode](https://www.linode.com/docs/platform/manager/dns-manager/)

## Configurando shinyproxy

Aunque las configuraciones de shinyproxy no están en el scope de está documentación, en la carpeta /shinyproxy/ hay una archivo YAML reemplaza al default de shinyproxy. Desde allí pueden agregar más apps imitando está parte:

```yaml
  specs:
  - id: testapp
    display-name: shinyapptest
    description: Primera shiny app en este servidor local
    container-cmd: ["R", "-e", "shiny::runApp('/root/firstapp')"]
    container-network: redshinyproxy
    container-image: testappimage
```

Si quisieran agregar otra, solamente deberían hacer lo siguiente

```yaml
  specs:
  - id: app2
    display-name: nuevaapp
    description: Segunda shiny app en este servidor local
    container-cmd: ["R", "-e", "shiny::runApp('/root/secondapp')"]
    container-network: redshinyproxy
    container-image: nombreimagenapp2
```
Para comprender el resto de las configuraciones de shinyproxy, les recomiendo consultar su [página de configuración](https://www.shinyproxy.io/configuration/)
Sin embargo, aunque tengan todo en funcionamiento, deben crear imagenes de docker que contengan a sus aplicaciones shiny. Este es el último paso.

## Agregando apps

Este repositorio tiene una carpeta que se llama testapp, que contiene la aplicación *Hello* de shiny, mostrando sus capacidads básicas. Les recomiendo mantener la estructura de esta carpeta y seguir los siguientes pasos:

1) Desarrollen la app
2) Una vez que la tengan desarrollada, hagan que esté todo contenido en una carpeta llamada app
3) Muevan esa carpeta hacia una nueva que tenga un nombre facilmente identificable como testapp
4) Agreguen el Rprofile.site que está en /testapp/ en este repositorio
5) Modifiquen el Dockerfile para que el contenedor pueda reproducir la app. Esto implica dos codas
5.A) Que instalen todos los paquetes que usan en la aplicación en esta sección

```docker
# Install R packages that are required
# TODO: add further package if you need!
RUN R -e "install.packages(c('shiny'))"
```

También puede ser que algunos paquetes que quieran instalar from source no se pueda y tengan que instalar los binarios. Para eso, encuentran el paquete de instalación en ubuntu y lo ponen acá

```docker
# Place to install libs from binary that might not be possible to install within R 
# RUN apt-get update && apt-get install -y 
# RUN sudo apt-get install r-cran-rcpp -y
```

5.B) Garantizando que copian la aplicación con el nombre correcto y terminan el dockerfile llamando a la dirección correcta. En otras palabras, que si usaron COPY app /root/secondapp, hayan creado ese directorio en la linea anterior y luego modifiquen la linea de CMD con /root/secondapp

```docker
# copy the app to the image
RUN mkdir /root/firstapp
COPY app /root/firstapp

COPY Rprofile.site /usr/lib/R/etc/

CMD ["R","-e", "shiny::runApp('/root/firstapp')"]
```

Una vez que tengan esto armado, hacen docker build y le agregan un tag (-t) que debe ser el mismo que usaron en el application.yml de shinyproxy. Es la forma que tiene de saber cuál contenedor tiene que disparar cuando llegue la consulta.

## Contacto y comentarios

Esta es una primera versión que estoy seguro es mejorable y adaptable, pero actualmente funciona en diversos servidores. Cualquier consulta para la implementación pueden escribirme a martinmontane@gmail.com

