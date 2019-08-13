---
title: 'The perfect nginx config'
published: true
taxonomy:
    category:
        - blog
    tag:
        - software
        - Plex
        - nginx
visible: true
---

I use docker for all of my applications running on my unRAID server, but for nginx I didn't find any image that fulfilled all my needs. So I wrote my own dockerimage which you can find here: https://github.com/Starbix/dockerimages/tree/master/nginx.

The following is my config and should explain what most things do and why.

**nginx.conf**

This limits the maximal connections per IP
```
worker_processes auto;
pid /nginx/run/nginx.pid;
daemon off;
pcre_jit on;

events {
    worker_connections 2048;
    use epoll;
}

http {
    limit_conn_zone $binary_remote_addr zone=limit_per_ip:10m;
    limit_conn limit_per_ip 128;
    limit_req_zone $binary_remote_addr zone=allips:10m rate=150r/s;
    limit_req zone=allips burst=150 nodelay;
```
<br>The custom log format is needed for [nginx amplify](https://www.nginx.com/products/nginx-amplify/) (statistics and more)
```
    include /nginx/conf/mime.types;
    default_type  application/octet-stream;

    log_format  main_ext  '$remote_addr - $remote_user [$time_local] "$request" '
                          '$status $body_bytes_sent "$http_referer" '
                          '"$http_user_agent" "$http_x_forwarded_for" '
                          '"$host" sn="$server_name" '
                          'rt=$request_time '
                          'ua="$upstream_addr" us="$upstream_status" '
                          'ut="$upstream_response_time" ul="$upstream_response_length" '
                          'cs=$upstream_cache_status' ;

    access_log /nginx/logs/nginx_access.log main_ext;
    error_log /nginx/logs/nginx_error.log warn;
```
<br>The maximum upload size is 25GB
nginx doesn't send that the webserver is `nginx` but `server`
```
    client_max_body_size 25G;

    aio threads;
    aio_write on;
    sendfile on;
    keepalive_timeout 15;
    keepalive_disable msie6;
    keepalive_requests 100;
    tcp_nopush on;
    tcp_nodelay on;
    server_tokens off;
    more_set_headers 'Server: secret';
```
<br>Content will be encoded in [brötli](https://github.com/google/brotli) (which Safari now supports)

```
    gzip off;

    brotli on;
    brotli_static on;
    brotli_buffers 16 8k;
    brotli_comp_level 6;
    brotli_types
        text/css
        text/javascript
        text/xml
        text/plain
        text/x-component
        application/javascript
        application/x-javascript
        application/json
        application/xml
        application/rss+xml
        application/vnd.ms-fontobject
        font/truetype
        font/opentype
        image/svg+xml;
```
<br> This will include all config files from /sites-enabled. The following configuration is the part you'll configure yourself.
```
    include /sites-enabled/*.conf;
    include /nginx/custom_sites/*.conf;
    include /nginx/conf.d/stub_status.conf;
}
```
The preceding configuration is part of the included nginx.conf of my Dockerimage.
<br>

/sites-enabled/**default.conf**

These TLS parameters currently result in a 100% score in all categories on https://www.ssllabs.com/ssltest

- OCSP stapling is enabled you can learn more about it [here](https://scotthelme.co.uk/ocsp-stapling-speeding-up-ssl/)
- I use [hybrid certificates](https://scotthelme.co.uk/hybrid-rsa-and-ecdsa-certificates-with-nginx/)

```
upstream php-handler {
    server unix:/php/run/php-fpm.sock;
}

ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers [TLS13+AESGCM+AES256|TLS13+CHACHA20]:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305;
ssl_ecdh_curve secp521r1:secp384r1;
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:20m;
ssl_session_timeout 15m;
ssl_session_tickets off;
ssl_stapling on;
ssl_dyn_rec_enable on;
resolver 1.1.1.1 1.0.0.1 ipv6=off;
ssl_stapling_verify on;
#RSA certificates
ssl_certificate /certs/example.com/fullchain.pem;
ssl_certificate_key /certs/example.com/key.pem;
#ECDSA certificates
ssl_certificate /certs/example.com_ecc/fullchain.pem;
ssl_certificate_key /certs/example.com_ecc/key.pem;

ssl_trusted_certificate /certs/example.com/fullchain.pem;
```
<br>I use custom HTTP error pages and I used this for creating them: https://github.com/AndiDittrich/HttpErrorPages

```
error_page 400 401 402 403 404 500 501 502 503 520 521 533 /error/HTTP$status.html;
```

<br> This server redirects all of the unencrypted connections to https. It uses the same url, except when it's accessed over the IP address, then it redirects to the root of the domain.
```
server {
  listen 8000 default_server;
  server_name _;
  include /sites-enabled/headers.conf;

  if ($host  ~ "\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b" ) {
    return 301 https://example.com$request_uri;

  }

  return 301 https://$host$request_uri;
}
```
<br> This server redirects unused subdomains.

```
server {
  listen 4430 ssl http2;
  server_name www.example.com;
  return 301 https://example.com$request_uri;
  include /nginx/conf.d/hsts.conf;
  include /sites-enabled/headers.conf;
}
```
<br> This is the server that listens on the root of the domain, in my case it redirects to the `media` subdomain.
```
server {
  listen 4430 ssl http2 default_server;
  #listen [::]:4430 ssl http2;
  server_name example.com;
  include /nginx/conf.d/hsts.conf;
  include /sites-enabled/headers.conf;

  return 301 https://media.example.com$request_uri;

  location /error/ {
    alias /www/errorpages/;
    internal;
    }

  }
```
<br> I use [Organizr](https://github.com/causefx/Organizr) to organize (duh) all of my apps I regularly use. Definitely check it out if you don't already use it, it's very actively developed and has so many features.
This server also has all of the reverse proxies for my apps I open to the public.
```
server {
    listen 4430 ssl http2;
    #listen [::]:4430 ssl http2;
    server_name media.example.com;
    include /nginx/conf.d/hsts.conf;
    include /sites-enabled/headers.conf;

    root /www/media;

    index index.php index.html;

		location = /robots.txt {
        allow all;
        log_not_found off;
        access_log off;
    }

    location /error/ {
      alias /www/errorpages/;
      internal;
      }

		location / {
				try_files $uri $uri/ /index.html /index.php?$args =404;
			}

		location ~ \.php$ {
			fastcgi_split_path_info ^(.+\.php)(/.+)$;
			fastcgi_pass unix:/php/run/php-fpm.sock;
			fastcgi_index index.php;
			fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
			include fastcgi_params;
		}

		##
		# Virtual Host Configs
		##
		location /sonarr {
    	proxy_pass http://192.168.1.126:8989;
    	proxy_set_header Host $host;
			proxy_set_header X-Real-IP $remote_addr;
			proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header    X-Forwarded-Host    $server_name;
			proxy_set_header    X-Forwarded-Proto   $scheme;
			proxy_set_header    X-Forwarded-Ssl     on;
			}

    location /request {
    	proxy_pass http://192.168.1.126:3579;
    	proxy_set_header Host $host;
    	proxy_set_header X-Real-IP $remote_addr;
    	proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
			}

		location /plex {
			proxy_pass              http://192.168.1.126:32400/web;
			proxy_set_header        Host                 $host;
    	proxy_set_header        X-Real-IP            $remote_addr;
      proxy_set_header        X-Forwarded-Host     $server_name;
      proxy_set_header        X-Forwarded-For      $proxy_add_x_forwarded_for;
      proxy_set_header        X-Forwarded-Proto    $scheme;
      proxy_set_header        X-Forwarded-Ssl      on;

			}

    if ($http_referer ~* /plex/) {
              rewrite ^/web/(.*) /plex/web/$1? redirect;
        }

		location /web {
      proxy_pass              http://192.168.1.126:32400;
      proxy_set_header        Host                 $host;
      proxy_set_header        X-Real-IP            $remote_addr;
			proxy_set_header        X-Forwarded-Host     $server_name;
      proxy_set_header        X-Forwarded-For      $proxy_add_x_forwarded_for;
      proxy_set_header        X-Forwarded-Proto    $scheme;
  		proxy_set_header        X-Forwarded-Ssl      on;
			}

		location /plexpy {
      proxy_pass http://192.168.1.126:8181;
    	proxy_set_header    Host $host;
    	proxy_set_header    X-Real-IP $remote_addr;
    	proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header    X-Forwarded-Host    $server_name;
			proxy_set_header    X-Forwarded-Proto   $scheme;
			proxy_set_header    X-Forwarded-Ssl     on;
			}

		location /radarr {
	   	proxy_pass http://192.168.1.126:7878;
	   	proxy_set_header    Host $host;
	   	proxy_set_header    X-Real-IP $remote_addr;
	    proxy_set_header    X-Forwarded-For $proxy_add_x_forwarded_for;
			proxy_set_header    X-Forwarded-Host    $server_name;
			proxy_set_header    X-Forwarded-Proto   $scheme;
			proxy_set_header    X-Forwarded-Ssl     on;
  		}

	 }
```
<br> I use a subdomain for Plex and you should be able to close the Plex port in your router if you use this if you add the subdomain to the `Custom server access URLs` in the Server settings of Plex. You need to use a environmental variable to set it if you use Docker like I do.

```
   upstream plex-upstream {
     server 192.168.1.126:32400;
   }

   upstream nextcloud-upstream {
     server 192.168.1.126:8888;
   }

   server {
     listen 4430 ssl http2;
     server_name plex.example.com;
     include /nginx/conf.d/hsts.conf;
     include /sites-enabled/headers.conf;

     location /error/ {
       alias /www/errorpages/;
       internal;
       }

     location / {
       # If a request to / comes in, 301 redirect to the main plex page,
       # but only if it doesn't contain the X-Plex-Device-Name header or query argument.
       # This fixes a bug where you get permission issues when accessing the web dashboard.
       set $test "";

       if ($http_x_plex_device_name = '') {
         set $test A;
       }
       if ($arg_X-Plex-Device-Name = '') {
         set $test "${test}B";
       }
       if ($test = AB) {
         rewrite ^/$ https://$host/web/index.html;
       }

       proxy_redirect off;
       proxy_buffering off;

       # Spoof the request as coming from ourselves since otherwise Plex will block access, e.g. logging:
       # "Request came in with unrecognized domain / IP 'tv.example.com' in header Referer; treating as non-local"
       proxy_set_header        Host                      $server_addr;
       proxy_set_header        Referer                   $server_addr;
       proxy_set_header        Origin                    $server_addr;

       proxy_set_header        X-Real-IP                 $remote_addr;
       proxy_set_header        X-Forwarded-For           $proxy_add_x_forwarded_for;
       proxy_set_header        X-Plex-Client-Identifier  $http_x_plex_client_identifier;
       proxy_set_header        Cookie                    $http_cookie;

       ## Required for Websockets
       proxy_http_version      1.1;
       proxy_set_header        Upgrade                   $http_upgrade;
       proxy_set_header        Connection                "upgrade";
       proxy_read_timeout      36000s;                   # Timeout after 10 hours

       proxy_next_upstream     error timeout invalid_header http_500 http_502 http_503 http_504;

       proxy_pass http://plex-upstream;
     }
   }
```
<br>This is [Nextcloud](https://nextcloud.com), my personal cloud
```
   server {
     listen 4430 ssl http2;
     server_name cloud.example.com;
     include /nginx/conf.d/hsts.conf;
     include /sites-enabled/headers.conf;

     location /error/ {
       alias /www/errorpages/;
       internal;
       }

     client_max_body_size 25G;

     location / {
     proxy_set_header Host $host;
     proxy_set_header X-Forwarded-Proto $scheme;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     add_header Front-End-Https on;
     proxy_pass http://nextcloud-upstream;
     proxy_set_header    X-Forwarded-Ssl     on;
     }
   }

   server {
    listen 4430 ssl http2;
    #listen [::]:4430 ssl http2;
    server_name  office.example.com;
    include /nginx/conf.d/hsts.conf;
    include /sites-enabled/headers.conf;

    proxy_buffering off;
    location / {
      return 301 https://cloud.example.com;
    }

    location /error/ {
      alias /www/errorpages/;
      internal;
      }

      # static files
      location ^~ /loleaflet {
          proxy_pass https://192.168.1.126:9980;
          proxy_set_header Host $http_host;
      }

      # WOPI discovery URL
      location ^~ /hosting/discovery {
          proxy_pass https://192.168.1.126:9980;
          proxy_set_header Host $http_host;
      }

      # Main websocket
      location ~ /lool/(.*)/ws$ {
          proxy_pass https://192.168.1.126:9980;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "Upgrade";
          proxy_set_header Host $http_host;
          proxy_read_timeout 36000s;
      }

      # Admin Console websocket
      location ^~ /lool/adminws {
          proxy_pass https://192.168.1.126:9980;
          proxy_set_header Upgrade $http_upgrade;
          proxy_set_header Connection "Upgrade";
          proxy_set_header Host $http_host;
          proxy_read_timeout 36000s;
      }

      # download, presentation and image upload
      location ^~ /lool {
          proxy_pass https://192.168.1.126:9980;
          proxy_set_header Host $http_host;
      }
}
```
<br>My blog which you are reading right now is [Grav](https://getgrav.org)

```
server {
  listen 4430 ssl http2;
  #listen [::]:4430 ssl http2;
  server_name blog.example.com;
  include /nginx/conf.d/hsts.conf;
  include /sites-enabled/headers.conf;

  root /www/blog;
  index index.html index.php;

  location /error/ {
    alias /www/errorpages/;
    internal;
    }



  location / {
            try_files $uri $uri/ /index.php?_url=$uri&$query_string;
        }

  location ~ \.php$ {
                fastcgi_pass unix:/php/run/php-fpm.sock;
                fastcgi_split_path_info ^(.+\.php)(/.+)$;
                fastcgi_index index.php;
                include fastcgi.conf;
                #sfastcgi_param SCRIPT_FILENAME $document_root/$fastcgi_script_name;
                fastcgi_max_temp_file_size 0;

fastcgi_buffer_size 4K;
fastcgi_buffers 64 4k;

        }

        ## Begin - Security
        # deny all direct access for these folders
        location ~* /(.git|cache|bin|logs|backup|tests)/.*$ { return 403; }
        # deny running scripts inside core system folders
        location ~* /(system|vendor)/.*\.(txt|xml|md|html|yaml|php|pl|py|cgi|twig|sh|bat)$ { return 403; }
        # deny running scripts inside user folder
        location ~* /user/.*\.(txt|md|yaml|php|pl|py|cgi|twig|sh|bat)$ { return 403; }
        # deny access to specific files in the root folder
        location ~ /(LICENSE.txt|composer.lock|composer.json|nginx.conf|web.config|htaccess.txt|\.htaccess) { return 403; }
        ## End - Security

}
```
<br>I store my passwords on [Bitwarden](https://bitwarden.com)
```
server {
 listen 4430 ssl http2;
 #listen [::]:4430 ssl http2;
 server_name  bitwarden.laubacher.io;
 include /nginx/conf.d/hsts.conf;
 include /includes/laubacher.tls.conf;
 include /includes/headers.conf;
 add_header X-Robots-Tag none;

 client_max_body_size 128M;

 location /error/ {
   alias /www/errorpages/;
   internal;
   }


 location / {
     proxy_pass        http://192.168.1.126:8343;
     proxy_set_header Host $host;
     proxy_set_header X-Real-IP $remote_addr;
     proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
     proxy_set_header X-Forwarded-Proto $scheme;
 }

 location /notifications/hub {
  proxy_pass http://192.168.1.126:3012;
  proxy_set_header Upgrade $http_upgrade;
  proxy_set_header Connection "upgrade";
}

location /notifications/hub/negotiate {
  proxy_pass http://192.168.1.126:8343;
}
}
```

<br>

/includes/**headers.conf**
```
add_header X-Frame-Options "ALLOW-FROM https://*.example.com" always;
add_header X-Content-Type-Options "nosniff";
add_header X-XSS-Protection "1; mode=block";
add_header Referrer-Policy "strict-origin";
add_header Expect-CT "enforce; max-age=86400; report-uri=https://laubacher.report-uri.io/r/default/ct/enforce";
add_header Expect-Staple "max-age=31536000; report-uri=https://laubacher.report-uri.io/r/default/staple/reportOnly; includeSubDomains; preload";
add_header Content-Security-Policy "frame-ancestors https://*.example.com https://example.com";
add_header X-Robots-Tag none;
```
Here the blocked countries get a `444` response.
```
# COUNTRY GEO BLOCK
if ($allowed_country = no) {
  return 444;
}
```

<br>

If you want to know more about blocking certain IPs with GeoIP2, check out [this guide](https://technicalramblings.com/blog/blocking-countries-with-geolite2-using-the-letsencrypt-docker-container/).

/sites-enabled/**geoip.conf**
```
geoip2 /includes/geolite2/GeoLite2-Country.mmdb {
  auto_reload 1d;
  $geoip2_data_country_code country iso_code;
  $geoip2_data_continent_name continent names en;
}
# GEO IP BLOCK SITE 1
map $geoip2_data_country_code $allowed_country {
    default yes;
    CN no; #China
    RU no; #Russia
    HK no; #Hong Kong
    IN no; #India
    IR no; #Iran
    VN no; #Vietnam
    TR no; #Turkey
    EG no; #Egypt
    MX no; #Mexico
    JP no; #Japan
    KP no; #North Korea 🙂
    PE no; #Peru
    BR no; #Brazil
    UA no; #Ukraine
    ID no; #Indonesia
    TH no; #Thailand
    AE no;
    AF no;
    SA no;
    MY no;
    BY no;
    MD no;
    AL no;
  }
```
