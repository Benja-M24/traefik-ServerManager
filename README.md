# traefik-ServerManager
Using docker create only one traefik container to manage every project on my server.


# Steps to use

1. Clone the Repository

```git 
git clone https://github.com/Benja-M24/traefik-ServerManager.git
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
