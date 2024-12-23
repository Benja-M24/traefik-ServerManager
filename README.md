# traefik-ServerManager
Using docker create only one traefik container to manage every project on my server.


# Steps to use

1. Clone the Repository

```git 
git clone https://github.com/Benja-M24/traefik-servermanager.git
```
2. Create Docker Network
Traefik and the projects need to communicate over a shared Docker network.

```bash 
docker network create traefik-server
```
3. Traefik Docker Container
Set up the necessary environment variables for Traefik and each project.

```bash 
cp .env.example .env
nano .env
```
Edit the .env file to set your domain and other necessary variables.

```.env
USERNAME=your_admin_username
PASSWORD=your_admin_password
DOMAIN=yourdomain.com
ACME_EMAIL=youremail@domain.com
```

4. Generate and set the hashed password in one command for Traefik's HTTP Basic Auth.
```bash
export HASHED_PASSWORD=$(openssl passwd -apr1 $PASSWORD)
```

5. Start Traefik container:
```bash
docker compose up -d
```

# For other project
Using Docker Containers for each one, create a .env and docker-compose.yml files based on the following examples.

```.env
# Enter the DOMAIN for that project or use sub-domain.
DOMAIN=project1.yourdomain.com
# or
DOMAIN=otherProjectDomain.com
```

```docker-compose.yml
services:
  project1:
    image: your-image-name
    environment:
    - ENVIRONMENT=example
    labels:
      - traefik.enable=true
      - traefik.http.routers.project-http.rule=Host(`${DOMAIN}`)
      - traefik.http.routers.project-http.entrypoints=http
      - traefik.http.routers.project-http.middlewares=https-redirect
      - traefik.http.routers.project-https.rule=Host(`${DOMAIN}`)
      - traefik.http.routers.project-https.entrypoints=https
      - traefik.http.routers.project-https.tls=true
      - traefik.http.routers.project-https.tls.certresolver=le
      - traefik.http.routers.project-https.service=project-service
      - traefik.http.services.project-service.loadbalancer.server.port=3000
    networks:
      - traefik-server
volumes:

networks:
  traefik-server:
    external: true
```
# How to setup cloudflare DNS as TLS certificate provider
1. Go to the Cloudflare dashboard and copy the API Token.
2. Create a file named cloudflare.ini in the root directory of your project and add the following content:

```cloudflare.ini	
dns_cloudflare_email = YOUR_CLOUDFLARE_EMAIL
dns_cloudflare_api_key = YOUR_CLOUDFLARE_API_KEY
```
3. Update the docker-compose.yml file to include the following labels for Traefik:

```docker-compose.yml
services:
  traefik:
    image: traefik:3.0
    ports:
      - 80:80 # HTTP port
      - 443:443 # HTTPS port
    restart: always
    labels:
      # Traefik configuration
      - traefik.enable=true
      - traefik.docker.network=traefik-server
      - traefik.http.services.traefik-dashboard.loadbalancer.server.port=8080

      # HTTP router configuration
      - traefik.http.routers.traefik-dashboard-http.entrypoints=http
      - traefik.http.routers.traefik-dashboard-http.rule=Host(`traefik.${DOMAIN?Variable not set}`)

      # HTTPS router configuration
      - traefik.http.routers.traefik-dashboard-https.entrypoints=https
      - traefik.http.routers.traefik-dashboard-https.rule=Host(`traefik.${DOMAIN?Variable not set}`)
      - traefik.http.routers.traefik-dashboard-https.tls=true
      - traefik.http.routers.traefik-dashboard-https.tls.certresolver=dns-cloudflare
      - traefik.http.routers.traefik-dashboard-https.service=api@internal

      # HTTPS redirect middleware
      - traefik.http.middlewares.https-redirect.redirectscheme.scheme=https
      - traefik.http.middlewares.https-redirect.redirectscheme.permanent=true
      - traefik.http.routers.traefik-dashboard-http.middlewares=https-redirect

      # HTTP Basic Auth middleware
      - traefik.http.middlewares.admin-auth.basicauth.users=${USERNAME?Variable not set}:${HASHED_PASSWORD?Variable not set}
      - traefik.http.routers.traefik-dashboard-https.middlewares=admin-auth
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro # Docker socket for reading service labels
      - traefik-server-certificates:/certificates # Certificate storage
      - /path/to/cloudflare.ini:/etc/traefik/cloudflare.ini
    command:
      # Docker provider configuration
      - --providers.docker
      - --providers.docker.exposedbydefault=false

      # Entrypoints configuration
      - --entrypoints.http.address=:80
      - --entrypoints.https.address=:443

      # Let's Encrypt configuration
      - --certificatesresolvers.dns-cloudflare.acme.email=${ACME_EMAIL?Variable not set}
      - --certificatesresolvers.dns-cloudflare.acme.storage=/certificates/acme.json
      - --certificatesresolvers.dns-cloudflare.acme.dnschallenge.provider=cloudflare
      - --certificatesresolvers.dns-cloudflare.acme.dnschallenge.resolvers=1.1.1.1:53

      # Logging and API configuration
      - --accesslog
      - --log
      - --api

    networks:
      - traefik-server # Public network for Traefik and other services

volumes:
  traefik-server-certificates: # Persistent volume for certificates

networks:
  traefik-server:
    external: true # Use pre-existing traefik-server network
```	
With this configuration, Traefik will use Cloudflare to resolve TLS certificates via the DNS challenge. Make sure to restart the Traefik container after making these changes.

# Generarte images and push to GHCR 
docker build -t ghcr.io/benja-m24/traefik-servermanager:lts .
docker push ghcr.io/benja-m24/traefik-servermanager:lts