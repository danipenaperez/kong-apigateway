FROM kong/kong-gateway:latest
USER root

ADD kong-plugin-jwt-auth-rbac/ /usr/local/share/lua/5.1/kong/plugins/jwt-auth-rbac/

RUN \
 apk add \
  --virtual build-dependencies \
   build-base \
   gcc \
   wget \
   git \
   ruby \
   openssl-dev \
   libuuid \
   yaml-dev


# Not fails, but nothing is downloaded on image creation
# expected that download plugin and store at /usr/local/share/lua/5.1/kong/plugins/  but not fails but does nothing
#RUN \
# luarocks install kong-plugin-jwt-auth-rbac --local 

USER kong