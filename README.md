# automated-tls-for-containers


```
version: "3.8"

services:
  nginx-proxy:
    image: nginxproxy/nginx-proxy:alpine
    container_name: nginx-proxy
    restart: unless-stopped      
    ports:
      - "80:80"                 
      - "443:443"              
    volumes:
      - conf:/etc/nginx/conf.d:rw   
      - vhost:/etc/nginx/vhost.d:rw 
      - html:/usr/share/nginx/html
      - dhparam:/etc/nginx/dhparam
      - certs:/etc/nginx/certs:ro  
      - /var/run/docker.sock:/tmp/docker.sock:ro
    logging:
      options:
         max-size: "15m"
         max-file: "3"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost/"]
      interval: 30s
      timeout: 5s
      retries: 3
    networks:
      - nginx-proxy

  acme-companion:
    image: nginxproxy/acme-companion
    container_name: acme-companion
    restart: unless-stopped
    depends_on:
      - nginx-proxy               
    volumes:
      - conf:/etc/nginx/conf.d:rw   
      - vhost:/etc/nginx/vhost.d:rw 
      - html:/usr/share/nginx/html
      - dhparam:/etc/nginx/dhparam
      - certs:/etc/nginx/certs:rw
      - acme:/etc/acme.sh
      - /var/run/docker.sock:/var/run/docker.sock:ro
    logging:
      options:
         max-size: "15m"
         max-file: "3"
    environment:
      DEFAULT_EMAIL: foo@bar.com
      NGINX_PROXY_CONTAINER: "nginx-proxy"
    networks:
      - nginx-proxy

volumes:
  conf:
  vhost:
  html:
  dhparam:
  certs:
  acme:

networks:
  nginx-proxy:
    driver: bridge

```
