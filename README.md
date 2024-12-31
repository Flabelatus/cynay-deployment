# Deploy the Cynay Search Engine
***The application is currently on an AWS EC2 Instance***
located at `/var/www/Cynay-search-engine`
Within the `/var/www` folder the `reverse-proxy` directory contains the `Caddyfile` for the settings of the
reverse proxy which we would run inside using a docker image

# Steps to follow:

## Setup Caddy as Reverse-proxy

### Run the Caddy via Docker
First execute the following:
```bash
docker pull caddy
```
Then you can run the pulled image via the following commad:
```bash
docker run -d \
--network=host \
--name caddy \
-p 80:80 \
-p 443:443 \
-v /var/www/reverse-proxy/Caddyfile:/etc/caddy/Caddyfile \
-v caddy_data:/data \
-v caddy_config:/config \
caddy
```

To ensure the docker image for caddy is running simply check via `docker ps`
Also to read it's encyption logs you can check `docker logs caddy`

***This would only work once the DNS is configured correctly by pointing at the ip of the server***

### Add the Caddyfile

Create the Caddyfile at `/var/www/reverse-proxy/Caddyfile` with the following content
```
{
  admin off
}

cynay.com, www.cynay.com {

        log {
                output discard
        }

        tls urfavalm@gmail.com

        @api {
                path /config
                path /healthz
                path /stats/errors
                path /stats/checker
        }

        @static {
                path /static/*
        }

        @notstatic {
                not path /static/*
        }

        @imageproxy {
                path /image_proxy
        }

        @notimageproxy {
                not path /image_proxy
        }

        header {
                # Enable HTTP Strict Transport Security (HSTS) to force clients to always connect via HTTPS
                Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"

                # Enable cross-site filter (XSS) and tell browser to block detected attacks
                X-XSS-Protection "1; mode=block"

                # Prevent some browsers from MIME-sniffing a response away from the declared Content-Type
                X-Content-Type-Options "nosniff"

                # Disable some features
                Permissions-Policy "accelerometer=(),ambient-light-sensor=(),autoplay=(),camera=(),encrypted-media=(),focus-without-user-activation=(),geolocation=(),gyroscope=(),magnetometer=(),microphone=(),midi=(),payment=(),picture-in-picture=(),speaker=(),sync-xhr=(),usb=(),vr=()"

                # Disable some features (legacy)
                Feature-Policy "accelerometer 'none';ambient-light-sensor 'none'; autoplay 'none';camera 'none';encrypted-media 'none';focus-without-user-activation 'none'; geolocation 'none';gyroscope 'none';magnetometer 'none';microphone 'none';midi 'none';payment 'none';picture-in-picture 'none'; speaker 'none';sync-xhr 'none';usb 'none';vr 'none'"

                # Referer
                Referrer-Policy "no-referrer"

                # X-Robots-Tag
                X-Robots-Tag "noindex, noarchive, nofollow"

                # Remove Server header
                -Server
        }

        header @api {
                Access-Control-Allow-Methods "GET, OPTIONS"
                Access-Control-Allow-Origin  "*"
        }

        # Cache
        header @static {
                # Cache
                Cache-Control "public, max-age=31536000"
                defer
        }

        header @notstatic {
                # No Cache
                Cache-Control "no-cache, no-store"
                Pragma "no-cache"
        }

        # CSP (see http://content-security-policy.com/ )
        header @imageproxy {
                Content-Security-Policy "default-src 'none'; img-src 'self' data:"
        }

        header @notimageproxy {
                Content-Security-Policy "upgrade-insecure-requests; default-src 'none'; script-src 'self'; style-src 'self' 'unsafe-inline'; form-action 'self' https://github.com/searxng/searxng/issues/new; font-src 'self'; frame-ancestors 'self'; base-uri 'self'; connect-src 'self' https://overpass-api.de; img-src 'self' data: https://*.tile.openstreetmap.org; frame-src https://www.youtube-nocookie.com https://player.vimeo.com https://www.dailymotion.com https://www.deezer.com https://www.mixcloud.com https://w.soundcloud.com https://embed.spotify.com"
        }

        # Cynay
        handle {
                encode zstd gzip
                reverse_proxy localhost:8080 {
                        header_up Host {http.request.host}
                        header_up X-Forwarded-Port {http.request.port}
                        header_up X-Forwarded-Proto {http.request.scheme}
                        header_up X-Real-IP {remote_host}
                }
        }
}
```

## Run the Cynay Search Engine Application
Since the decision was made to not run this instance in a docker, we can run it via the `make run` command inside the ubuntu machine
However, its best to have it run by the supervisor 

### Install the Supervisor and Run the Application
1. To install the supervisor run: `sudo apt update` and `sudo apt install supervisor -y`

2. To verify installation run `supervisord --version` and ensure that the supervisor service is running: `sudo systemctl status supervisor`

3. If its not running start it: `sudo systemctl start supervisor`

4. After installing the supervisor in ubuntu add the following config file for the cynay project `/etc/supervisor/conf.d/cynay.conf`
5. Then add the following commands:
```
[program:cynay]
command=make run
directory=/var/www/Cynay-search-engine
autostart=true
autorestart=true
stderr_logfile=/var/log/cynay.err.log
stdout_logfile=/var/log/cynay.out.log
user=ubuntu
environment=INSTANCE_NAME="cynay",BASE_URL="https://cynay.com"
```
6. After this make sure the supervisor is updated with new config file
`sudo supervisorctl reread`
`sudo supervisorctl update`

7. Then you can start the program:
`sudo supervisorctl start cynay`

***Note:*** 

- To stop the app run `sudo supervisorctl stop cynay`
- Check the status: `sudo supervisorctl status`
- You also can read the logs from the `/var/log/cynay.err.log` and `/var/log/cynay.out.log` files
- Running this, the application would be started on the `localhost:8080`

