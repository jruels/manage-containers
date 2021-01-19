## Deploying a local private registry

This lab walks through deploying a local Docker registry. This registry uses a self-signed certificate for encryption, and should be not used in a production environment.    

Start a simple Docker registry server   

```bash
docker run -d -p 5000:5000 --restart=always -v/reg:/var/lib/registry --name registry registry:2
```

As discussed previously: 

`-d` = run container in background.   
`-p` = map port 5000 of the container to 5000 on the host.   
`-v` = Store images in a persistent volume on the host.   

The Docker daemon does not allow connection to **insecure** registries by default.   
To enable this we need to add the following to `/etc/docker/daemon.json`
```json
{
  "insecure-registries" : ["localhost:5000"]
}
```

Restart Docker daemon to apply changes:
```
sudo service docker restart
```

Confirm Docker registry is up and running 
```
curl -v localhost:5000/v2/
```

You should see something like:
```
*   Trying 127.0.0.1...
...snip
Connected to localhost (127.0.0.1) port 5000 (#0)
> GET /v2/ HTTP/1.1
> Host: localhost:5000
Connected to localhost (127.0.0.1) port 5000 (#0)
> GET /v2/ HTTP/1.1
> Host: localhost:5000
```

Fantastic, now your private Docker registry is up and running.
The registry has been deployed using HTTP which allows anyone to push and pull from it. 

Let's add encryption and authentication 

### Setup a self-signed certificate. 

Start by creating a directory and certificate. 

**NOTE**: When prompted for CN (Common Name) provide `localhost`. Fill in the other fields with whatever you want.
```
mkdir -p docker_reg_certs
openssl req  -newkey rsa:4096 -nodes -sha256 -keyout docker_reg_certs/domain.key -x509 -days 365 -out docker_reg_certs/domain.crt
```

Now install the certicates: 
```
sudo mkdir -p /etc/docker/certs.d/localhost:5000
sudo cp docker_reg_certs/domain.crt /etc/docker/certs.d/localhost:5000/ca.crt
sudo cp docker_reg_certs/domain.crt /usr/local/share/ca-certificates/ca.crt
sudo update-ca-certificates
```

### Setup authenticaton with htpasswd

Use htpasswd to create a username and password
```
mkdir docker_reg_auth
docker run -it --entrypoint htpasswd -v $PWD/docker_reg_auth:/auth -w /auth registry:2.7.0 -Bbc /auth/htpasswd admin password
```

Restart Docker daemon
```
sudo service docker restart
```

Stop the registry
```
docker rm -f registry 
```

Start the registry with new configuration 
```
docker run -d -p 5000:5000 --restart=always \
                           --name registry\
                           -v $PWD/docker_reg_certs:/certs \
                           -v $PWD/docker_reg_auth:/auth \
                           -v /reg:/var/lib/registry \
                           -e REGISTRY_HTTP_TLS_CERTIFICATE=/certs/domain.crt \
                           -e REGISTRY_HTTP_TLS_KEY=/certs/domain.key \
                           -e "REGISTRY_AUTH_HTPASSWD_REALM=Registry Realm" \
                           -e REGISTRY_AUTH_HTPASSWD_PATH=/auth/htpasswd \
                           -e REGISTRY_AUTH=htpasswd \
                           registry:2
```

### Test new registry 

Log into the registry using credentials previously created 
```
docker login -uadmin -ppassword localhost:5000
```

Now that we've authenticated let's pull an image from Docker Hub and push it to our private registry 
```
docker pull nginx 
docker tag nginx localhost:5000/private-http
docker push localhost:5000/private-http
```

If everything was successful you now have an image in your private registry that can be pulled. 
```
docker pull localhost:5000/private-http
```

To retrieve a list of all repositories in the registry you can run: 
```
curl -u admin:password -k https://localhost:5000/v2/_catalog
```


## Lab complete
