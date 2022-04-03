
## How to clone my repo:
- get the private ssh key that I have attached in the email
- go to the directory where you downloaded ssh key
- add it to ssh agent

```bash
eval "$(ssh-agent -s)"
ssh-add lendable_exercise
```
- clone the repo
```bash
git clone  --recursive git@github.com:yorg666/lendable_exercise.git
```

- cd to lendable_exercise/tech-test

- build docker container
```bash
docker build --ssh github=./lendable_exercise -t buildtest:v1 .
```

- run the container
```bash
docker run -p 80:80 buildtest:v1 -dit &
```

- check that nginx page is available
```bash
curl localhost:80
```


## Notes:

## Version 1 container is using code from local  host

First I have copied test files to my local host.

```bash
scp -i ~/.ssh/id_rsa -v -r ec2-user@34.253.236.221:/home/ec2-user/tech-test .
```

Then tried to build docker container but I got an error:

```bash
docker build -t lendable:v1 .
```

```bash
❯ docker run lendable:v1 -dit -v /home/yorg/lendable/tech-test/www:/www -p 80:80
nginx: [emerg] open() "/run/nginx/nginx.pid" failed (2: No such file or directory)
```

**Solution**

I have added to Dockerfile

```bash
RUN mkdir -p /run/nginx
```

Initially www directory was empty

```bash
❯ docker exec -it 5a5039c5192b ls -la /www
total 8
drwxr-xr-x    2 nginx    www           4096 Apr  1 14:54 .
drwxr-xr-x    1 root     root          4096 Apr  1 15:10 ..
```

But nginx configuration was in place

```bash
❯ docker exec -it 5a5039c5192b cat /etc/nginx/conf.d/default.conf
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        # Everything is a 404
        root                    /www;
        index                   index.html index.htm;
        server_name             localhost;


        # You may need this to prevent return 404 recursion.
        location = /404.html {
                internal;
        }
}% 

```
I have created www directory and basic create index.html file

```bash
❯ pwd
/home/yorg/lendable/tech-tes
❯ mkdir www


❯ docker exec -it ab7f39552625 ls -la  /www
total 12
drwxr-xr-x    1 nginx    www           4096 Apr  1 15:31 .
drwxr-xr-x    1 root     root          4096 Apr  1 15:32 ..
-rw-r--r--    1 root     root           183 Apr  1 15:24 index.html
```

Comand to run docker container was incorrect, becase local port was not forwarded to container port

**Solution**
I had to move -p 81:80 from the end of command and add it after run

```bash
docker run -p 81:80 lendable:v6 &
```

Port forwarding start working and I was able to see nginx welcome page

```bash
❯ docker ps
CONTAINER ID   IMAGE         COMMAND                  CREATED          STATUS          PORTS                NAMES
7db7bfb4e7f5   lendable:v6   "/bin/sh -c 'nginx -…"   51 seconds ago   Up 49 seconds   0.0.0.0:81->80/tcp   elegant_galileo
```


```bash
❯ curl localhost:81

<!doctype html>
<html lang="en">

<head>
    <meta charset="utf-8">
    <title>Krystian Nginx</title>
</head>

<body>
    <h2>Hello Lendable from Nginx container</h2>
</body>

</html>%  

```

At this point I had an working solution, but the container is using code from my local computer

## Version 2 container is cloning github repo using ssh

Container is missing packages to be able to clone git repo over ssh

```bash
sh: git: not found
```

**Solution**

```bash
apk add git 
```

Openssl is also required

```bash
/tmp # git clone --recursive git@github.com:yorg666/lendable_exercise.git
Cloning into 'lendable_exercise'...
fatal: unable to fork
```

**Solution**
```bash
apk add openssh
```

Dring cloning I was getting warning about ECDSA key fingerprint

```bash
/tmp # git clone  --recursive git@github.com:yorg666/lendable_exercise.git
Cloning into 'lendable_exercise'...
The authenticity of host 'github.com (140.82.121.4)' can't be established.
ECDSA key fingerprint is SHA256:p2QAMXNIC1TJYWeIOttrVc98/R1BUFWu3/LiyKgUfQM.
Are you sure you want to continue connecting (yes/no)?
```

**Solution**
```bash
ssh-keyscan github.com >> ~/.ssh/known_hosts
```

I tried to clone repo but I was missing ssh key

```bash
/tmp # git clone  --recursive git@github.com:yorg666/lendable_exercise.git
Cloning into 'lendable_exercise'...
git@github.com: Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

**Solution**
Generate ssh keypair and add public key to remote repo

```bash
❯ ssh-keygen -t ecdsa -b 521 -f lendable_exercise
Generating public/private ecdsa key pair.
Enter passphrase (empty for no passphrase): 
Enter same passphrase again: 
Your identification has been saved in lendable_exercise
Your public key has been saved in lendable_exercise.pub
The key fingerprint is:
SHA256:gc5gOJsEVFUcq0I9MkCKF/tzGlnI2gxgZcSJgAfzGcM yorg@DESKTOP-QJUE2VM
The key's randomart image is:
+---[ECDSA 521]---+
|@BX=o.oo.        |
|=*E@.. o.        |
|o.X+=oo..        |
| o.@o*o  .       |
|  +.B.+ S        |
|    .=           |
|    .            |
|                 |
|                 |
+----[SHA256]-----+

.rw------- yorg yorg 748 B  Fri Apr  1 18:24:18 2022  lendable_exercise
.rw-r--r-- yorg yorg 274 B  Fri Apr  1 18:24:18 2022  lendable_exercise.pub
```

Git push failed, so I had to add private key to ssh agent

```bash
❯ git push
ERROR: Permission to yorg666/lendable_exercise.git denied to krystian-sky.
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.
```

**Solution**

```bash
❯ eval "$(ssh-agent -s)"
Agent pid 835
❯ ssh-add tech-test/lendable_exercise
Identity added: tech-test/lendable_exercise (yorg@DESKTOP-QJUE2VM)
```

Finally I have scan my container for known vulnerabilities
```bash
❯ docker scan buildtest:v10

Testing buildtest:v10...

Package manager:   apk
Project name:      docker-image|buildtest
Docker image:      buildtest:v10
Platform:          linux/amd64
Base image:        alpine:3.8

✔ Tested 28 dependencies for known vulnerabilities, no vulnerable paths found.

According to our scan, you are currently using the most secure version of the selected base image
Alpine 3.8.5 is no longer supported by the Alpine maintainers. Vulnerability detection may be affected by a lack of security updates.

For more free scans that keep your images secure, sign up to Snyk at https://dockr.ly/3ePqVcp
```

After I have bump Alpine versio from 3.8 to 3.15 I was no longer able to build container
```bash
❯ docker build --ssh github=../../priv_key/lendable_exercise -t buildtest:v12 .
[+] Building 2.5s (14/15)                                                                                                                                                                    
 => [internal] load build definition from Dockerfile                                                                                                                                    0.0s
 => => transferring dockerfile: 690B                                                                                                                                                    0.0s
 => [internal] load .dockerignore                                                                                                                                                       0.0s
 => => transferring context: 2B                                                                                                                                                         0.0s
 => [internal] load metadata for docker.io/library/alpine:3.15                                                                                                                          1.2s
 => [ 1/12] FROM docker.io/library/alpine:3.15@sha256:f22945d45ee2eb4dd463ed5a431d9f04fcd80ca768bb1acf898d91ce51f7bf04                                                                  0.0s
 => CACHED [ 2/12] RUN addgroup www                                                                                                                                                     0.0s
 => CACHED [ 3/12] RUN adduser -D -g 'www' nginx                                                                                                                                        0.0s
 => CACHED [ 4/12] RUN apk update &&     apk add nginx &&     apk add git &&     apk add openssh                                                                                        0.0s
 => CACHED [ 5/12] RUN mkdir -p /run/nginx                                                                                                                                              0.0s
 => CACHED [ 6/12] RUN mkdir -m 700 /root/.ssh;   touch -m 600 /root/.ssh/known_hosts;   ssh-keyscan github.com > /root/.ssh/known_hosts                                                0.0s
 => CACHED [ 7/12] RUN --mount=type=ssh,id=github git clone git@github.com:yorg666/lendable_exercise.git                                                                                0.0s
 => CACHED [ 8/12] WORKDIR /lendable_exercise/tech-test                                                                                                                                 0.0s
 => [ 9/12] RUN ls -la && ls -la /etc/nginx/                                                                                                                                            0.3s
 => [10/12] RUN cp -r ./www /www/                                                                                                                                                       0.4s
 => ERROR [11/12] RUN cp ./nginx/default.conf /etc/nginx/conf.d/default.conf                                                                                                            0.5s
------
 > [11/12] RUN cp ./nginx/default.conf /etc/nginx/conf.d/default.conf:
#14 0.475 cp: can't create '/etc/nginx/conf.d/default.conf': No such file or directory
```

**Solution**
```bash
RUN mkdir /etc/nginx/conf.d/
```

But now I'm getting 404
```bash
❯ curl localhost:81
<html>
<head><title>404 Not Found</title></head>
<body>
<center><h1>404 Not Found</h1></center>
<hr><center>nginx</center>
</body>
</html>
```

```bash
❯ docker exec -ti 919e8ae539ca sh
/lendable_exercise/tech-test # 
```

```bash
/lendable_exercise/tech-test # nginx -t
nginx: [emerg] "server" directive is not allowed here in /etc/nginx/conf.d/default.conf:1
nginx: configuration file /etc/nginx/nginx.conf test failed
```

**Solution**
```bash
RUN cp ./nginx/default.conf /etc/nginx/http.d/default.conf

❯ docker exec -ti 4b3ab8af3063 sh
/lendable_exercise/tech-test # nginx -s reload
/lendable_exercise/tech-test # nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

## Documentation that I used
https://rtfm.co.ua/en/git-git-clone-fatal-unable-to-fork-and-rsa-key-fingerprint/
https://dzone.com/articles/clone-code-into-containers-how
https://www.docker.com/blog/how-to-use-the-official-nginx-docker-image/