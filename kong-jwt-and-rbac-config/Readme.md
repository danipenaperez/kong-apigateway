# KONG WITH JWT AND RBAC PLUGINS

# Instalar esta imagen
```
# Build locally
docker build -t danipenaperez/kong .
# Publish dockerhub
docker login
docker tag danipenaperez/kong danipenaperez/kong:latest
#Push
docker push danipenaperez/kong:latest
```
Esta imagen esta basada en latest de kong e instala el plugin kong-plugin-jwt-auth-rbac
que puedes encontrar en la pagina https://luarocks.org/modules/prabago/kong-plugin-jwt-auth-rbac

# Añadiendo custom plugins
El plugin  jwt-auth-rbac no viene incluido en los plugins bundled que trae kong por defecto, por eso hay que instalarlo.

Se añade el plugin local al directorio de plugins de kong (Dockerfile)
```
ADD kong-plugin-jwt-auth-rbac/ /usr/local/share/lua/5.1/kong/plugins/jwt-auth-rbac/
```
Habilitamos el plugin en kong.conf
```
(ojo que el nombre del plugin es registrado sin "kong-plugin")
plugins = bundled,jwt-auth-rbac
```
Tambien podiamos haberlo habilitado indicando la variable de entorno
```
docker run 
....
-e KONG_PLUGINS="bundled,jwt-auth-rbac"
...
```
# Crear la imagen
```
docker build -t danipenaperez/kong .
```
# RUN Image
```
docker run --rm --name danipenaperez_kong \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -e "KONG_LOG_LEVEL=debug" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    -v ${PWD}:/etc/kong/ \
    --net=host \
    danipenaperez/kong
```    

# Testing
## Arrancando el servicio protegido
Vamos a levantar un servicio detras del apigateway
El servicio se puede arrancar asi, respondera echo a nuestras request en el http://localhost:3000 :
```
docker run --rm -p 3000:80 ealen/echo-server
y llamamos y nos hace echo  
$ curl http://localhost:3000?echo_body=amazing
"amazing"
```

## Configurando apigateway
En el fichero kong.yml incluimos estas lineas para crear el enrutado y configurar el plugin jwt y el jwt-rbac
```
routes:
  - name: echo-service-route
    service: echo-service
    paths:
      - /public-path-to-echo-service
plugins:
  - name: jwt
    route: echo-service-route
    enabled: true
    config: 
      key_claim_name: kid
      claims_to_verify:  
        - exp
  - name: jwt-auth-rbac
    route: echo-service-route
    enabled: true
    config:
      roles_claim_name: rolecillos
      roles: [role1,role2,role3]
consumers:
  - username: thisserverissuer
jwt_secrets:
  - consumer: thisserverissuer
    key: thisserverissuer_kid
```

## Arrancamos Kong

```
docker run --rm --name danipenaperez_kong \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -e "KONG_LOG_LEVEL=debug" \
    -e KONG_PLUGINS="bundled,jwt-auth-rbac" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    -v ${PWD}:/etc/kong/ \
    --net=host \
    docker run --rm --name danipenaperez_kong \
    -e "KONG_PROXY_ACCESS_LOG=/dev/stdout" \
    -e "KONG_ADMIN_ACCESS_LOG=/dev/stdout" \
    -e "KONG_PROXY_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_ERROR_LOG=/dev/stderr" \
    -e "KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl" \
    -e "KONG_LOG_LEVEL=debug" \
    -p 8000:8000 \
    -p 8443:8443 \
    -p 8001:8001 \
    -p 8444:8444 \
    -v ${PWD}:/etc/kong/ \
    --net=host \
    danipenaperez/kong
```   
## Invocando sin Token [401 UnAauthorized]
```   
$ curl -i  http://localhost:8000/public-path-to-echo-service?echo_body=amazing
HTTP/1.1 401 Unauthorized
Date: Fri, 06 May 2022 09:53:26 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 26
X-Kong-Response-Latency: 2
Server: kong/2.8.1.0-enterprise-edition

{"message":"Unauthorized"}
```   
## Invocando con token valido, pero sin roles [403 Forbidden]
Para generar el token usamos jwtio , metiendo los valores kid en las cabeceras, el campo exp y el secret con los valores correspondientes de la configuracion del plugin jwt que hemos visto arriba
![Jwt create token](docs/jwt-io-1.png)
```   
$ curl -i -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InRoaXNzZXJ2ZXJpc3N1ZXJfa2lkIn0.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE5MDAwMDAwMDB9.iXnZ2pWOUY5AXYNYHeu4Y3kRqx-FQQW3I4xnPAFh5zc" http://localhost:8000/public-path-to-echo-service?echo_body=amazing
HTTP/1.1 403 Forbidden
Date: Fri, 06 May 2022 10:00:58 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 63
X-Kong-Response-Latency: 1
Server: kong/2.8.1.0-enterprise-edition

{"message":"The claim roles are not informed in the JWT token"}
``` 
## Invocando con roles pero el rol no es valido para ese recurso
![Jwt create token](docs/jwt-io-3.png)
``` 
$ curl -i -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InRoaXNzZXJ2ZXJpc3N1ZXJfa2lkIn0.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE5MDAwMDAwMDAsInJvbGVjaWxsb3MiOlsicm9sZXBhY28iXX0.cFRblmz0lo-VOP3FlzLUpqD5xT8laEgJzfajxA6zXBM" http://localhost:8000/public-path-to-echo-service?echo_body=amazing
HTTP/1.1 403 Forbidden
Date: Fri, 06 May 2022 10:08:03 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 200
X-Kong-Response-Latency: 2
Server: kong/2.8.1.0-enterprise-edition

{"detail":"The permitted role for this invocation is [role1, role2, role3] and yours role are [rolepaco]","message":"To be able to use this service you must have at least one of the roles configured"}
``` 
## Invocando con token valido y roles [200 OK]
![Jwt create token](docs/jwt-io-2.png)
``` 
$ curl -i -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InRoaXNzZXJ2ZXJpc3N1ZXJfa2lkIn0.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE5MDAwMDAwMDAsInJvbGVjaWxsb3MiOlsicm9sZTEiXX0.hRkUzGkrLz5_vnwxvqCatum9nDaxKJwb0o1J9Jbj7uc" http://localhost:8000/public-path-to-echo-service?echo_body=amazing
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 9
Connection: keep-alive
ETag: W/"9-9C3TDmXfhoWPizWzjFyCX+fxVeQ"
Date: Fri, 06 May 2022 10:04:10 GMT
X-Kong-Upstream-Latency: 24
X-Kong-Proxy-Latency: 2
Via: kong/2.8.1.0-enterprise-edition

"amazing"
``` 

# Referencias
- Use Custom plugins: https://tech.aufomm.com/use-custom-plugins-with-kong/