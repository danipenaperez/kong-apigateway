# kong-apigateway
Kong api gateway demo

Show how to deploy a service behind kong api gateway (DB Less- local)

First run a simple echo service and test that works
```
docker run --rm -p 3000:80 ealen/echo-server
y llamamos y nos hace echo  
$ curl http://localhost:3000?echo_body=amazing
"amazing"
```

Then run Kong API gateway (kong is configured via kong.conf file, and redirect is via kong.yml)
```
docker run --rm --name kong \
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
Now Request agains Kong to fetch internal route service 
```
$ curl http://localhost:8000/public-path-to-echo-service?echo_body=amazing
"amazing"

```

