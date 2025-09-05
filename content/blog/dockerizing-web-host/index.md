---
title: "Dockerizing a web host"
date: 2017-03-03T22:14:04-05:00
draft: false

taxonomies:
    tags:
      - homelab
      - docker
---

## Intro

I maintain a web server for this blog and a few other things (my [portfolio](https://bjschafer.com), for example). I had a pretty decent setup going, whereby nginx would serve all incoming requests. Sites like my portfolio (which is a Jekyll static site) are served straight-up by nginx, whereas sites like this blog (which runs on Ghost) are proxy-passed to the correct process.
I even had a pretty nice reusable snippet for setting up SSL using Let's Encrypt certs. Everything worked. But knowing me, I can't leave well enough alone :)
I switched to a new VPS provider to save a bit of money. Turns out, they also support Docker! So, I thought - may as well throw these sites in their own containers, for security and ease-of-hosting. Well...it's not as straightforward as it may seem, especially for somebody who had never used Docker before...

Thanks to [this fantastic guide/repo](https://gilyes.com/docker-nginx-letsencrypt/), I was able to get a head start. Basically, this creates a `docker-compose.yml` file. It contains directions for an nginx container as well as a meta-container which generates nginx configs based on other running containers. It also has a container which takes care of getting certs through Let's Encrypt.
From there, I simply have to Dockerize each of my sites.
I like to keep this in `/opt/docker`, but you do you.

## Static site

The simplest one to get started! My portfolio is all static files.
First things first - we have to create a (simple) Dockerfile. It should look something like this and live in `./www/site.com/`:

```dockerfile
FROM nginx
COPY ./ /usr/share/nginx/html/
```

Next, we'll need a tiny little config file for nginx. I throw it in a /volumes/ subdirectory of my project location.
This contains:

```
server {  
    listen                  80;
    server_name             site.com;
    root                    /usr/share/nginx/html;
}
Add it to the docker-compose.yml file as follows:
  site-name:
    restart: always
    image: site-name
    build: ./www/site.com
    container_name: site-name
    volumes:
      - "./volumes/site.com/conf.d/:/etc/nginx/conf.d"
    environment:
      - VIRTUAL_HOST=site.com
      - VIRTUAL_NETWORK=nginx-proxy
      - VIRTUAL_PORT=80
      - LETSENCRYPT_HOST=site.com
      - LETSENCRYPT_EMAIL=user@site.com
```

## Simple service

Next, I needed to set up my blog, which runs on Ghost. This was a bit trickier to figure out but probably easier to set up. Note that you shouldn't need to write your own Dockerfile here, since it's pretty bog-standard. Just attach a volume for persistent storage of your db, config, and assets.
Add to `docker-compose.yml`:

```yaml
  service-name:
    restart: always
    image: ghost
    container_name: service-name
    volumes:
      - "./volumes/service-name:/var/lib/ghost/content"
    environment:
      - VIRTUAL_HOST=service-name.com
      - VIRTUAL_NETWORK=nginx-proxy
      - VIRTUAL_PORT=2368 # or whatever port your service runs on
      - LETSENCRYPT_HOST=service-name.com
      - LETSENCRYPT_EMAIL=user@site.com
      - NODE_ENV=production # tells Ghost to use the production config section.
```

Note that for Ghost to work properly, I had to modify its config.js file and add the following to the bottom of the production section:

```
        paths: {
            contentPath: path.join(__dirname, '/content/')
        }
```

## Service with dependencies

Ahh, the fun part :) I host a site for some personal documentation. This requires a MySQL database to store its information.
Modify your `docker-compose.yml` as follows:

```yaml
  wiki-mysql:
    restart: always
    image: mysql:5.7.12
    volumes:
      - "./volumes/servicename/mysql/:/var/lib/mysql"
    container_name: wiki-mysql
    environment:
      - VIRTUAL_NETWORK=nginx-proxy
      - MYSQL_ROOT_PASSWORD=rootpassword
      - MYSQL_DATABASE=dbname
      - MYSQL_USER=username
      - MYSQL_PASSWORD=password
service-name:
    restart: always
    image: whatever-you-need
    container_name: service-name
    depends_on:
      - wiki-mysql
    volumes:
      - # Add any persistent storage your service needs here, e.g. for config or assets.
    environment:
      - VIRTUAL_HOST=service-name.com
      - VIRTUAL_NETWORK=nginx-proxy
      - VIRTUAL_PORT=80
      - LETSENCRYPT_HOST=service-name.com
      - LETSENCRYPT_EMAIL=user@site.com
      - DB_HOST=wiki-mysql:3306
      - DB_DATABASE=dbname
      - DB_USERNAME=username
      - DB_PASSWORD=password
```

## Fire it up!

It's time to see how things work. Simply run `docker-compose up` from the directory containing your `docker-compose.yml` file. If all goes well, it should let you know it's starting your containers, and then you'll see log messages fly.
Update your DNS records to point over here (if you haven't already), and see how things work!
If you need a shell in a container, you can use `docker exec -it <container-name> bash`. Note that most images are pretty sparse, so you'll have to make do with pretty much just `cat` and `more`. Most apps, though, should log to docker-compose's logs.
Assuming everything looks good, you can stop things with `docker-compose down`. Then, restart it as a service with `docker-compose up -d`.
Run `crontab -e` to edit your crontab, and stick in:
`@reboot /usr/local/bin/docker-compose -f /opt/docker/docker-compose.yml -d`
to ensure that it starts up when your box reboots.

## Conclusion

I am still very much a Docker novice. However, I was able to make this work - and so can you! I'll update this as I tweak things.

One final note - I'd recommend versioning things that matter, such as your `docker-compose.yml`. I'm a huge git fan, but you should use whatever you're most comfortable with.
