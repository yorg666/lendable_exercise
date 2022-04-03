
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

## Documentation that I used
https://rtfm.co.ua/en/git-git-clone-fatal-unable-to-fork-and-rsa-key-fingerprint/
https://dzone.com/articles/clone-code-into-containers-how