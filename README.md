# kaniko-bug-1

Create cert for registry
```
openssl req -newkey rsa:2048 -nodes -keyout tls.key -x509 -days 365 -out tls.crt
```

Setup network
```
docker network create --driver bridge kaniko
```

Start local registry
```
docker run -d -p 443:443 \
    --network kaniko \
    --hostname kaniko-registry \
    -v `pwd`/tls.crt:/opt/tls/tls.crt \
    -v `pwd`/tls.key:/opt/tls/tls.key \
    -e REGISTRY_HTTP_ADDR="0.0.0.0:443" \
    -e REGISTRY_HTTP_TLS_CERTIFICATE=/opt/tls/tls.crt \
    -e REGISTRY_HTTP_TLS_KEY=/opt/tls/tls.key \
    --name kaniko-registry \
    registry:2.7.1 
```

# Build via Kaniko

Build image push to local registry
```
docker run --network kaniko    \
 -v `pwd`:/opt/build  \
-it gcr.io/kaniko-project/executor:debug-v0.24.0   \
--skip-tls-verify-registry kaniko-registry:443    \
--destination=kaniko-registry:443/kaniko-bug-1:kaniko-build1   \
    --context=/opt/build --whitelist-var-run=false \
    --dockerfile=/opt/build/Dockerfile.4kaniko
```

Note the output of the `/etc/apache2/conf.d` permission prior to the `COPY` at line 53
```
INFO[0027] Running: [/bin/sh -c ls -al /etc/apache2]    
total 112
drwxr-xr-x    4 root     root          4096 Jul 16 21:26 .
drwxr-xr-x    1 root     root          4096 Jul 16 21:26 ..
drwxrwxrwx    2 root     root          4096 Jul 16 21:26 conf.d
```

Note the output of the `/etc/apache2/conf.d` permission after to the `COPY` at line 53
```
INFO[0027] Running: [/bin/sh -c ls -al /etc/apache2]    
total 112
drwxr-xr-x    4 root     root          4096 Jul 16 21:26 .
drwxr-xr-x    1 root     root          4096 Jul 16 21:26 ..
drwxr-xr-x    2 root     root          4096 Jul 16 21:26 conf.d
```

Shell in to inspect:
```

$ docker run -it --entrypoint /bin/bash localhost:443/kaniko-bug-1:kaniko-build1

bash-5.0$ ls -al /etc
total 212
...
lrwxrwxrwx    1 root     root            12 Jul 16 21:28 httpd -> /etc/apache2
```

# Build Via Docker

Build w/ docker. 
```
docker build -f Dockerfile.4kaniko -t kaniko-bug-1:docker-build1 .
```

Note the output of the `/etc/apache2/conf.d` permission prior to the `COPY` at line 53
```
Step 13/16 : RUN ls -al /etc/apache2
 ---> Running in bf0e36ca5aff
total 112
drwxr-xr-x    1 root     root          4096 Jul 16 21:28 .
drwxr-xr-x    1 root     root          4096 Jul 16 21:28 ..
drwxrwxrwx    1 root     root          4096 Jul 16 21:28 conf.d
```

Note the output of the `/etc/apache2/conf.d` permission after to the `COPY` at line 53
```
Step 16/16 : RUN ls -al /etc/apache2
 ---> Running in 15a6a50ad1a7
total 112
drwxr-xr-x    1 root     root          4096 Jul 16 21:28 .
drwxr-xr-x    1 root     root          4096 Jul 16 21:28 ..
drwxrwxrwx    1 root     root          4096 Jul 16 21:28 conf.d
```

Shell in to inspect:
```

$ docker run -it --entrypoint /bin/bash kaniko-bug-1:docker-build1

bash-5.0$ ls -al /etc
total 212
...
lrwxrwxrwx    1 root     root            12 Jul 16 21:28 httpd -> /etc/apache2
```