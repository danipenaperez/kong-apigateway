# kong-apigateway-jwt validation
Based on kong-basic-config project folder

# First run a simple echo service and test that works
```
docker run --rm -p 3000:80 ealen/echo-server 
$ curl http://localhost:3000?echo_body=amazing
"amazing"
```

# Configure kong
Using kong.conf
```
...
database = off
...
...
declarative_config = /etc/kong/kong.yml

```
# Run Kong Docker image using configuration files (pass as volume)
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

# TEST
Now Request agains Kong to fetch internal route service  will result on 401 Unauthorized
```
$ curl -i http://localhost:8000/public-path-to-echo-service?echo_body=amazing
HTTP/1.1 401 Unauthorized
Date: Thu, 05 May 2022 14:37:32 GMT
Content-Type: application/json; charset=utf-8
Connection: keep-alive
Content-Length: 26
X-Kong-Response-Latency: 0
Server: kong/2.8.1

{"message":"Unauthorized"}
```
Generate a Jwt expecified header Kid == jwt_secrets.key and secret == jwt_secrets.secret
![Jwt create token](images/jwtio-kong-1.png)
Now send with Authorization header:
```
$ curl -i -H "Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6InRoaXNzZXJ2ZXJpc3N1ZXJfa2lkIn0.eyJzdWIiOiIxMjM0NTY3ODkwIiwibmFtZSI6IkpvaG4gRG9lIiwiaWF0IjoxNTE2MjM5MDIyLCJleHAiOjE5MDAwMDAwMDB9.iXnZ2pWOUY5AXYNYHeu4Y3kRqx-FQQW3I4xnPAFh5zc" http://localhost:8000/public-path-to-echo-service?echo_body=amazing
HTTP/1.1 200 OK
Content-Type: application/json; charset=utf-8
Content-Length: 9
Connection: keep-alive
ETag: W/"9-9C3TDmXfhoWPizWzjFyCX+fxVeQ"
Date: Thu, 05 May 2022 14:45:01 GMT
X-Kong-Upstream-Latency: 32
X-Kong-Proxy-Latency: 1
Via: kong/2.8.1

"amazing"
```  


# References
More tutorials and configuration for kong on 
https://tech.aufomm.com/page/1