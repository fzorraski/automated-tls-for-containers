# automated-tls-for-containers

A simple way to enable HTTPS (TLS) for applications running in containers is by using [nginx-proxy](https://github.com/nginx-proxy/nginx-proxy) together with [acme-companion](https://github.com/nginx-proxy/acme-companion). This combo automates the creation of TLS certificates via Let's Encrypt and handles HTTPS traffic redirection to your container.

The idea is to place a **reverse proxy** (nginx) in front of your application. This way:

- Nginx listens on ports 80 (HTTP) and 443 (HTTPS);
- It handles HTTPS decryption;
- Then forwards the request to your app using HTTP, all automatically.

## How it works

1. The `nginx-proxy` watches active containers through the Docker socket and auto-generates virtual host configs.
2. The `acme-companion` works alongside nginx, handling the generation and renewal of Let's Encrypt certificates.
3. Your app container just needs to set a few environment variables and join the same Docker network as the proxy and companion.

---

## Requirements

- Docker and Docker Compose installed
- A public domain pointing to your server's IP (e.g. `foo.bar.com`)
- Ports 80 and 443 open on your firewall

---

## Container setup

Here’s a working example using `docker-compose.yml`:

### Proxy + Certbot

```yaml
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
    networks:
      - nginx-proxy
    logging:
      options:
        max-size: "15m"
        max-file: "3"
    healthcheck:
      test: ["CMD", "curl", "--fail", "http://localhost/"]
      interval: 30s
      timeout: 5s
      retries: 3

  acme-companion:
    image: nginxproxy/acme-companion
    container_name: acme-companion
    restart: unless-stopped
    depends_on:
      - nginx-proxy
    environment:
      DEFAULT_EMAIL: foo@bar.com
      NGINX_PROXY_CONTAINER: "nginx-proxy"
    volumes:
      - conf:/etc/nginx/conf.d:rw
      - vhost:/etc/nginx/vhost.d:rw
      - html:/usr/share/nginx/html
      - dhparam:/etc/nginx/dhparam
      - certs:/etc/nginx/certs:rw
      - acme:/etc/acme.sh
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks:
      - nginx-proxy
    logging:
      options:
        max-size: "15m"
        max-file: "3"

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

---

### Example application behind HTTPS

```yaml
version: '3.8'

services:
  wildfly:
    image: fabriciozrk/jboss:latest
    container_name: app_server
    environment:
      VIRTUAL_HOST: foo.bar.com
      VIRTUAL_PORT: 8080
      LETSENCRYPT_HOST: foo.bar.com
    ports:
      - "8083:8080"
    networks:
      - nginx-proxy
    restart: always
    logging:
      options:
        max-size: "15m"
        max-file: "3"

networks:
  nginx-proxy:
    external: true

```

## Step-by-step

1. **Set up your domain**
   - Make sure you have a valid domain name (e.g., `foo.bar.com`) pointing to your server's public IP address.

2. **Open required ports**
   - Ensure ports **80 (HTTP)** and **443 (HTTPS)** are open on your firewall or cloud provider settings.

3. **Create the Docker network**
   ```bash
   docker network create nginx-proxy

   

4. **Start the reverse proxy and companion**
   - Run the `nginx-proxy` and `acme-companion` containers using the provided `docker-compose.yml` file.
   - Example:
     ```bash
     docker-compose -f docker-compose.proxy.yml up -d
     ```

5. **Deploy your application container**
   - Make sure your application container includes the following environment variables:
     - `VIRTUAL_HOST`: your domain name (e.g., `foo.bar.com`)
     - `VIRTUAL_PORT`: the internal port your app listens to (e.g., `8080`)
     - `LETSENCRYPT_HOST`: same as your domain name
   - Also ensure the container is on the same Docker network as the proxy:
     ```yaml
     networks:
       - nginx-proxy
     ```

6. **Access your application**
   - Open your browser and go to:
     ```
     https://your-domain.com
     ```
   - You should see your application running with a valid SSL certificate issued by Let's Encrypt.


