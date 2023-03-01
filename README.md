Sometimes we must create our own docker registry to keep our docker images safe. In this example, we create a private docker registry secured with SSL and basic authentication.

So let’s start.

Here we are using an amazon Linux machine for our project.

before proceeding, we need to make sure that we have already installed docker, we can use the following command to install docker if it is not installed yet.

```
$ sudo yum install docker -y
$ sudo systemctl restart docker
$ sudo systemctl enable docker
```

Now we can add our ec2-user to the docker group

```
$ sudo usermod -a -G docker ec2-user
```

Check the version using the following command.

```
$ docker  version

Client:
 Version:           20.10.17
Server:
 Engine:
  Version:          20.10.17
  API version:      1.41 (minimum version 1.12)
  Go version:       go1.18.6
```

In the next step, we are installing docker-compose, for that install docker-compose using the following command.

```
sudo curl -SL https://github.com/docker/compose/releases/download/v2.6.0/docker-compose-linux-x86_64 -o /usr/bin/docker-compose
```

Once it is completed, you need to allow execution permission for the location.

```
sudo chmod +x /usr/bin/docker-compose
```

Now verify it is configured correctly.

```
$ docker-compose version
Docker Compose version v2.6.0
```

In the next step, we are downloading nginx for setting up the web server, use the following command to install the same.

```
$ sudo amazon-linux-extras install nginx1 -y
$ sudo systemctl restart nginx
$ sudo systemctl enable nginx
```

We need to configure the server block for our nginx server, we will get back to it, before that we need to install let's encrypt for securing our private registry.

```
$ sudo yum update
$ sudo yum install https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
$ sudo yum-config-manager --enable epel
$ sudo yum install certbot-nginx
```

Make sure that the installation was successful.

```
$ certbot --version
certbot 1.11.0
```

Now we need to configure nginx, first, we can create a document root.

We are setting up the private registry as “registry.antonyan.tech” in this case, so we are creating a document root for that under /var/www/html

```
$ sudo mkdir -p /var/www/html/registry.antonyan.tech 
$ sudo chown -R $USER:$USER /var/www/html/registry.antonyan.tech
```

Now we need to add our domain to conf.d , for that create a new file under the sites-available section.

```
$ sudo vi /etc/nginx/conf.d/registry.antonyan.tech.conf
```

add a basic server block in the file.

```
server {
        listen 80;
      
        root /var/www/html/registry.antonyan.tech;
        index index.html index.htm index.nginx-debian.html;

        server_name registry.antonyan.tech;

        location / {
                try_files $uri $uri/ =404;
        }
}

```


Now save and exit from the file.

You can add a sample index file in the document root to verify it is up and running.

Now we can install SSL for our registry file, use the following command install SSL. Make sure that you have pointed the domain name to your registry server.Also, replace registry.antonyan.tech with your URL.

```
$ sudo certbot --nginx -d registry.antonyan.tech
```

Complete the installation process, it SSL is issued successfully, you will get the following message.

```
Congratulations! You have successfully enabled https://registry.antonyan.tech
```

In the next step, we are setting up our docker-compose file, so create a registry directory for our project.

```
$ mkdir registry
```

Now create a folder in the registry folder to store the images.

```
$ cd registry/
$ mkdir data
```
 
We have to set up nginx port forwarding, our registry is listening to port 5000.

Update the /etc/nginx/conf.d/registry.antonyan.tech.conf to the following content.

```
server {

        root /var/www/html/registry.antonyan.tech;
        index index.html index.htm index.nginx-debian.html;

        server_name registry.antonyan.tech;

        location / {
                 # Do not allow connections from docker 1.5 and earlier
    # docker pre-1.6.0 did not properly set the user agent on ping, catch "Go *" user agents
    if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
      return 404;
    }

    proxy_pass                          http://localhost:5000;
    proxy_set_header  Host              $http_host;   # required for docker client's sake
    proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
    proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header  X-Forwarded-Proto $scheme;
    proxy_read_timeout                  900;
        }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/registry.antonyan.tech/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/registry.antonyan.tech/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = registry.antonyan.tech) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


        listen 80;

        server_name registry.antonyan.tech;
    return 404; # managed by Certbot


}
```

Now let’s set up basic authentication, using htpasswd

```
$ sudo yum install httpd-tools -y
```


Now we need to create a directory for storing the credentials.

```
$ mkdir registry/auth
```

Create your user and password for the registry

```
$ htpasswd -Bbn testuser test@123 > registry/auth/registry.password
```

Here the username will “testuser” with password “test@123”, you can change it as per your choice.

Once you create the login credentials, we can set up the docker-compose file.

```
$ cd registry/
```

We are using the registry image for setting up our registry, also we are setting up a restart policy as always for our container.

```
$ vi docker-compose.yml
```

```
services:
  registry:
    restart: always
    image: registry:2
    ports:
    - "5000:5000"
    environment:
      REGISTRY_AUTH: htpasswd
      REGISTRY_AUTH_HTPASSWD_REALM: Registry
      REGISTRY_AUTH_HTPASSWD_PATH: /auth/registry.password
      REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data
    volumes:
      - ./auth:/auth
      - ./data:/data
```

REGISTRY_AUTH defines that we are using htpasswd as our authentication method.

REGISTRY_AUTH_HTPASSWD_PATH defines the password file.

Also, we are creating 2 volumes for our data and auth folders. We are tweaking a value here in our nginx.conf. Sometimes our docker images will be large in size, so we are updating the uploading size to 10 GB which will be enough to upload the images.

Add the following line in nginx.conf in http block.

```
client_max_body_size 10240m;
```

Now restart nginx service, and make sure that there is no issue with the configuration.

```
$ sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

$ sudo systemctl restart nginx
```

Finally, we are starting our container with the following command.

```
$ docker-compose up -d
```

You can see the following output in the console

```
[+] Running 6/6
 ⠿ registry Pulled                                                                                                                                                          4.7s
   ⠿ ef5531b6e74e Pull complete                                                                                                                                             1.0s
   ⠿ a52704366974 Pull complete                                                                                                                                             1.2s
   ⠿ dda5a8ba6f46 Pull complete                                                                                                                                             1.5s
   ⠿ eb9a2e8a8f76 Pull complete                                                                                                                                             1.6s
   ⠿ 25bb6825962e Pull complete                                                                                                                                             1.6s
[+] Running 2/2
 ⠿ Network registry_default       Created                                                                                                                                   0.1s
 ⠿ Container registry-registry-1  Started  
 ```
 
 Now we can check our container status.

```
$ docker container ls -a
CONTAINER ID   IMAGE        COMMAND                  CREATED          STATUS          PORTS                                       NAMES
3dbbd492959a   registry:2   "/entrypoint.sh /etc…"   16 minutes ago   Up 16 minutes   0.0.0.0:5000->5000/tcp, :::5000->5000/tcp   registry-registry-1
```

You can see that our container is running and it listening to port 5000 of our host machine.

We need to make sure that we can push and pull images to our registry, for testing purposes we are pulling an alpine image from the docker hub.

```
$ docker image pull alpine

Using default tag: latest
latest: Pulling from library/alpine
63b65145d645: Pull complete
Digest: sha256:69665d02cb32192e52e07644d76bc6f25abeb5410edc1c7a81a10ba3f0efb90a
Status: Downloaded newer image for alpine:latest
docker.io/library/alpine:latest
```

Now we are updating the tag.

```
$ docker tag alpine  registry.antonyan.tech/newimage
```

Now we can log in to our private registry.

```
$ docker login registry.antonyan.tech
Username: testuser
Password:
WARNING! Your password will be stored unencrypted in /home/ec2-user/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

Let’s push the image to the private registry.

```
$ docker push registry.antonyan.tech/newimage

Using default tag: latest
The push refers to repository [registry.antonyan.tech/newimage]
7cd52847ad77: Pushed
latest: digest: sha256:e2e16842c9b54d985bf1ef9242a313f36b856181f188de21313820e177002501 size: 528
```

Success, the image was pushed to the registry without any issues. Now we are checking the availability of the registry from a client machine.

Log in to a different server and login to our private registry.

```
[root@client ~]# docker login registry.antonyan.tech
Username: testuser
Password:
WARNING! Your password will be stored unencrypted in /root/.docker/config.json.
Configure a credential helper to remove this warning. See
https://docs.docker.com/engine/reference/commandline/login/#credentials-store

Login Succeeded
```

Let’s pull the image now.


```
[root@client ~]# docker image pull registry.antonyan.tech/newimage
Using default tag: latest
latest: Pulling from newimage
63b65145d645: Pull complete
Digest: sha256:e2e16842c9b54d985bf1ef9242a313f36b856181f188de21313820e177002501
Status: Downloaded newer image for registry.antonyan.tech/newimage:latest
registry.antonyan.tech/newimage:latest

[root@client ~]# docker image ls -a
REPOSITORY                        TAG       IMAGE ID       CREATED       SIZE
registry.antonyan.tech/newimage   latest    b2aa39c304c2   2 weeks ago   7.05MB
```

We have configured the private registry successfully.

I hope this document was able to provide a clearcut idea about setting up a private docker registry.

Thank you for reading :)





