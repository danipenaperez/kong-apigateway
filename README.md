# kong-apigateway
Kong api gateway demo

# KONG

Es un api gateway extensible.

PAgina muy buena con how-to de kong: https://tech.aufomm.com/ 
https://www.youtube.com/watch?v=iz6i-RwsrJw
https://medium.com/gdgcloudsantiago/kong-gateway-configurar-con-yaml-o-interfaz-131b72e9f3c6

Lo normal es ponerlo en produccion manejado por una base de datos, que podemos compartir entre varias instancias y tambien
podemos actualizar y guardar su configuracion.


# Arranque 
Para local vamos a usarla sin persistencia de bbdd, asi lo probamos:

Lo podemos arrancar en local asi (https://hub.docker.com/_/kong):
```

$ docker run -d --name kong \
    -e "KONG_DATABASE=off" \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    kong

```
Podemos acceder a http://localhost:8001/ y ver la configuracion en formato json

# Configuracion
Para configurarlo en modo DB Less , es decir sin hacer uso de una bbdd,  necesitamos 2 archivos fundamentales 
kong.conf : que sera el archivo de configuracion global de kong 
kong.yml: es el que especificara la configuracion de redirecciones y mas asuntos de kong.

KONG.CONF:
Lo puedes copiar de este fuente oficial https://github.com/Kong/kong/blob/master/kong.conf.default y cambiarle el nombre a kong.conf.
De este archivo vamos a fijarnos en cambiar 2 properties:

```
# Vemos que por defecto los enrutados de Kong los hara de entrada desde el puerto 8000. Lo podemos cambiar
# proxy_listen = 0.0.0.0:8000
...
...
#El portal de administrador web estara por defecto en 8001 y 8444. Lo podemos cambiar.
# admin_listen = 127.0.0.1:8001
...
...
# Indicamos que no queremos bbdd, cambiando el valor a off
database = off
..
..
#Indicamos donde estara el fichero de configuracion kong.yml que lo editaremos ahora despues
declarative_config = /etc/kong/kong.yml
...
```

KONG.YML:
Es el fichero que indica los enrutamientos y transformaciones, Vamos a generar uno con este contenido:
```
services:
  - name: echo-service
    url: http://localhost:3000/
routes:
  - name: echo-service-route
    service: echo-service
    paths:
      - /public-path-to-echo-service
```
Vemos que hemos creado un route que indica que el /public-path-to-echo-service debe redireccionar a http://localhost:3000
Los ficheros de configuracion deben encontrarse en **/etc/kong**, como queremos inyectar nuestra configuracion, vamos a pasarle el archivo
por volumen a la imagen y vamos a configurar que use nuestra configuracion mediante variable de entorno.
Ya no nos hacen falta las variables de entorno, porque hemos especificado en el kong.conf como queremos que se comporte.
Como el servicio al que queremos enruta estara en nuestro ordendador indico el parametro --net=host
Nos queda asi:
```
docker run --rm --name kong \
    -e "KONG_DATABASE=off" \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    -v ${PWD}:/etc/kong/ \
    --net=host \
    kong
```
Podemos probar el API de Kong, que nos devuelve la configuracion:
```
$ curl http://localhost:8001 
{"tagline":"Welcome to kong","hostname":"dpena","configuration":{"cassandra_repl_f ....

# Y usando https en el 8444
$ curl https://localhost:8444
curl: (60) SSL certificate problem: self signed certificate
More details here: https://curl.haxx.se/docs/sslcerts.html

curl failed to verify the legitimacy of the server and therefore could not
establish a secure connection to it. To learn more about this situation and
how to fix it, please visit the web page mentioned above.

```

## Arrancando el servicio detras del api Gateway
Como es un api gateway, vamos a configurarlo para que actue como apigateway proxy delante de un servicio muy simple que retorna un echo
El servicio se puede arrancar asi, respondera echo a nuestras request en el http://localhost:3000 :
```
docker run --rm -p 3000:80 ealen/echo-server
y llamamos y nos hace echo  
$ curl http://localhost:3000?echo_body=amazing
"amazing"
```
Para configurar kong vamos a meter esta configuracion en kong.yml.
```

```
## Probando la solucion
Podemos acceder al servicio a traves del api gateway
```
$ curl http://localhost:8000/public-path-to-echo-service?echo_body=amazing
"amazing"

```

