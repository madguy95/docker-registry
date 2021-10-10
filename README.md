# DOCKER REGISTRY

### Requirements
At this point, it’s assumed that:

- you understand Docker security requirements, and how to configure your docker engines properly
- you have installed Docker Compose
- it’s HIGHLY recommended that you get a certificate from a known CA instead of self-signed certificates
- inside the current directory, you have a X509 domain.crt and domain.key, for the CN myregistrydomain.com
(or have domain valid and using certbot to generate certificate)
- be sure you have stopped and removed any previously running registry (typically docker container stop registry && docker container rm -v registry)

### Config Registry

Use this [example YAML file](https://github.com/docker/distribution/blob/master/cmd/registry/config-example.yml) as a starting point.

### **USING Delete**
For deleting images, you need to activate the delete feature in your registry:

```yml
storage:
    delete:
      enabled: true
```

If you are running the static interface don't forget the environment variable <span style="background:gray" >DELETE_IMAGES </span>.

Refer: [docker-registry-ui](https://github.com/Joxit/docker-registry-ui)

### Authenticate proxy with nginx
Use-case🔗

People already relying on a nginx proxy to authenticate their users to other services might want to leverage it and have Registry communications tunneled through the same pipeline.

Alternatives
If you just want authentication for your registry, and are happy maintaining users access separately, you should really consider sticking with the native [basic auth registry feature](https://docs.docker.com/registry/recipes/nginx/#:~:text=basic%20auth%20registry%20feature).

--- create basic auth

Create a password file with one entry for the user testuser, with password testpassword:
**Replace testuser testpassword with your username password**

```sh
 $ mkdir auth
 $ docker run \
  --entrypoint htpasswd \
  httpd:2 -Bbn testuser testpassword > auth/htpasswd
```
Create the main nginx configuration auth/nginx.conf
You can need to change some value to match you enviroment
 - server_name poordev.ddns.net; (with your domain)
 - server registry:5000; (with your registry server container)
 - auth_basic_user_file /etc/nginx/conf.d/nginx.htpasswd; (with your basic auth file)

### CONFIG SSL By Let's Enscript , Certbot
First of all, we need two shared Docker volumes. One for the validation challenges, the other for the actual certificates.

Add this to the volumes list of the nginx section in docker-compose.yml.

```yml
- $HOME/data/certbot/conf:/etc/letsencrypt
- $HOME/data/certbot/www:/var/www/certbot
```

And this is the counterpart that needs to go in the certbot section:

```yml
volumes:
  - $HOME/data/certbot/conf:/etc/letsencrypt
  - $HOME/data/certbot/www:/var/www/certbot
```

Now we can make nginx serve the challenge files from certbot! Add this to the first (port 80) section of our nginx configuration (data/nginx/app.conf):

```
location /.well-known/acme-challenge/ {
    root /var/www/certbot;
}
```

After that, we need to reference the HTTPS certificates. Add the soon-to-be-created certificate and its private key to the second server section (port 443). ***Make sure to once again replace poordev.ddns.net with your domain name.***

```
ssl_certificate /etc/letsencrypt/live/poordev.ddns.net/fullchain.pem;
ssl_certificate_key /etc/letsencrypt/live/poordev.ddns.net/privkey.pem;
```

And while we’re at it: The folks at Let’s Encrypt maintain best-practice HTTPS configurations for nginx. Let’s also add them to our config file. This will score you a straight A in the SSL Labs test!

```
include /etc/letsencrypt/options-ssl-nginx.conf;
ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
```

Now for the tricky part. We need nginx to perform the Let’s Encrypt validation But nginx won’t start if the certificates are missing.
So what do we do? Create a dummy certificate, start nginx, delete the dummy and request the real certificates.
Luckily, you don’t have to do all this manually, I have created a convenient script for this.
Download the script to your working directory as init-letsencrypt.sh:
```sh
curl -L https://raw.githubusercontent.com/wmnnd/nginx-certbot/master/init-letsencrypt.sh > init-letsencrypt.sh
```

Edit the script to add in your domain(s) and your email address. If you’ve changed the directories of the shared Docker volumes, make sure you also adjust the data_path variable as well.
Then run <span style="background:gray" >chmod +x init-letsencrypt.sh </span> and <span style="background:gray" >sudo ./init-letsencrypt.sh </span> .

Note:

you can get issue when let's enscript can't ping to nginx port 80. you need assurance nginx can running and open port 80 to external (firewall or proxy) with path **http:/domain/.well-known/acme-challenge/...** .

Automatic Certificate Renewal
Last but not least, we need to make sure our certificate is renewed when it’s about to expire. The certbot image doesn’t do that automatically but we can change that!
Add the following to the certbot section of docker-compose.yml:

```yml
entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
```

This will check if your certificate is up for renewal every 12 hours as recommended by Let’s Encrypt.
In the nginx section, you need to make sure that nginx reloads the newly obtained certificates:

```sh
command: "/bin/sh -c 'while :; do sleep 6h & wait $${!}; nginx -s reload; done & nginx -g \"daemon off;\"'"
```

This makes nginx reload its configuration (and certificates) every six hours in the background and launches nginx in the foreground.

Finish setup and run services:

```sh
docker-compose up -d
```

Usage:
```sh
docker login -u=testuser -p=testpassword poordev.ddns.net:5000
docker tag nginx:alpine poordev.ddns.net:5000/nginx:alpine
docker push poordev.ddns.net:5000/nginx:alpine
docker pull poordev.ddns.net:5000/nginx:alpine
```
If you use a insecure registry, you need add daemon.json
If the daemon.json file does not exist, create it. Assuming there are no other settings in the file, it should have the following contents:
```json
{
  "insecure-registries" : ["poordev.ddns.net:5000"]
}
```
After restart docker service and test again:
```sh
sudo service docker restart
```