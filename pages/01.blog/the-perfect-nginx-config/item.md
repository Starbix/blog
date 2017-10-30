---
title: 'The perfect nginx config'
published: false
---

I use docker for all of my applications running on my unRAID server, but for nginx I didn't find any image that fulfilled all my needs. So I wrote my own dockerimage which you can find here: https://github.com/Starbix/dockerimages/tree/master/nginx. 
**nginx.conf**
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
This limits the maximal connections per IP
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

    access_log /nginx/log/nginx_access.log main_ext;
    error_log /nginx/log/nginx_error.log warn;
```
The custom log format is needed for nginx amplify (statistics and more)
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
The maximum upload size is 25GB, nginx doesn't send that the webserver is `nginx` but `server`.
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
Content will be encoded in brötli (which Safari now supports)
```
    include /sites-enabled/*.conf;
    include /nginx/custom_sites/*.conf;
    include /nginx/conf.d/stub_status.conf;
}
```
The preceding configuration is part of the included nginx.conf of my Dockerimage. It will include all config files from /sites-enabled. The following configuration is the part you'll configure yourself.

/sites-enabled/**default.conf**
```
upstream php-handler {
    server unix:/php/run/php-fpm.sock;
}

ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers TLS13-AES-256-GCM-SHA384:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384;
ssl_ecdh_curve secp521r1:secp384r1;
ssl_prefer_server_ciphers on;
ssl_session_cache shared:SSL:20m;
ssl_session_timeout 15m;
ssl_session_tickets off;
ssl_stapling on;
ssl_dyn_rec_enable on;
resolver 8.8.8.8 8.8.4.4 ipv6=off;
ssl_stapling_verify on;
#RSA certificates
ssl_certificate /certs/example.com/fullchain.pem;
ssl_certificate_key /certs/example.com/key.pem;
#ECDSA certificates
ssl_certificate /certs/example.com_ecc/fullchain.pem;
ssl_certificate_key /certs/example.com_ecc/key.pem;

ssl_ct on;
ssl_ct_static_scts /certs/example.com;
ssl_ct_static_scts /certs/example.com_ecc;

ssl_dhparam /certs/dhparam.pem;
ssl_trusted_certificate /certs/example.com/fullchain.pem;
```
This TLS parameters currently result in a 100% score in all categories, but keep in mind that this kind of config doesn't make sense for most websites and I just use it just because I want 100% everywhere.
```
error_page 400 /error/HTTP400.html;
error_page 401 /error/HTTP401.html;
error_page 402 /error/HTTP402.html;
error_page 403 /error/HTTP403.html;
error_page 404 /error/HTTP404.html;
error_page 500 /error/HTTP500.html;
error_page 501 /error/HTTP501.html;
error_page 502 /error/HTTP502.html;
error_page 503 /error/HTTP503.html;
error_page 520 /error/HTTP520.html;
error_page 521 /error/HTTP521.html;
error_page 533 /error/HTTP533.html;
```
I use custom HTTP error pages and I used this for creating them: https://github.com/AndiDittrich/HttpErrorPages
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
This server redirects all of the unencrypted connections to https. It uses the same url, except when it's accessed over the IP address, then it redirects to the root of the domain.
```
server {
  listen 4430 ssl http2;
  #listen [::]:4430 ssl http2;
  server_name www.example.com mail.example.com;
  return 301 https://example.com$request_uri;
  include /nginx/conf.d/hsts.conf;
  include /sites-enabled/headers.conf;
}
```
This server redirects unused subdomains.
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
This is the server that listens on the root of the domain, in my case it redirects to the `media` subdomain.
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
I use [Organizr](https://github.com/causefx/Organizr) to organize (duh) all of my apps I regularly use. Definitely check it out, it's very actively developed and has so many features.
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
This server 
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
This is [Nextcloud](https://nextcloud.com) which is my personal cloud.
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
My blog which is [Grav](https://getgrav.org)
<br><br>
```
server {
  listen 4430 ssl http2;
  #listen [::]:4430 ssl http2;
  server_name john.example.com;
  #return 301 https://media.example.com$request_uri;
  include /nginx/conf.d/hsts.conf;
  include /sites-enabled/headers.conf;

  root /www/root;

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
/sites-enabled/**headers.conf**
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