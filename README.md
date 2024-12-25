# Run the Caddy Reverse-proxy via Docker
First execute the following:
```bash
        docker pull caddy
```
Then you can run the pulled image via the following commad:
```bash
        docker run -d \
        --name caddy \
        -p 80:80 \
        -p 443:443 \
        -v /var/www/Caddyfile:/etc/caddy/Caddyfile \
        -v caddy_data:/data \
        -v caddy_config:/config \
        caddy
```

# Install the Supervisor and Run the Application
After installing the supervisor in ubuntu add the following configs in `/etc/supervisor/conf.d/cynay.conf`

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

Create the Caddyfile
```
{
  admin off
}

cynay.com {

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
                        header_up X-Forwarded-Port {http.request.port}
                        header_up X-Forwarded-Proto {http.request.scheme}
                        header_up X-Real-IP {remote_host}
                }
        }
}
```

