# Fullstack Application Deployment on GCP / Ubuntu Server

## 1. Project Overview

This project demonstrates how to configure an Ubuntu server (GCP or any other cloud server) and deploy a fullstack application using Docker and Docker Compose.

The application consists of:

- **ReactJS** – Frontend
- **Spring Boot** – Backend REST API
- **PostgreSQL** – Relational Database
- **Nginx** – Web Server and Reverse Proxy
- **Docker** – Containerization
- **Docker Compose** – Multi-container orchestration
- **Portainer** – Docker container management

### Application Repositories

| Component | GitHub Repository |
|---|---|
| ReactJS Frontend | [sample-product-reactapp](https://github.com/keoKAY/sample-product-reactapp.git) |
| Spring Boot Backend | [spring-sample-product-jpa](https://github.com/keoKAY/spring-sample-product-jpa.git) |

## 2. Deployment Architecture

The application uses Docker Compose to manage the ReactJS frontend, Spring Boot backend, and PostgreSQL database.

```text
                    Internet
                       |
                       v
                Ubuntu Server
                       |
                  Host Nginx
                    Port 80
                       |
                       v
              ReactJS Frontend
                 Port 3000
                       |
                       v
              Spring Boot Backend
                 Internal API
                       |
                       v
                PostgreSQL
                 Port 5432
                       |
                Docker Volume
```

Portainer runs separately to manage Docker containers.

## 3. Update Ubuntu

Update the Ubuntu package lists and upgrade installed packages.

```bash
apt update
apt upgrade -y
```

Install basic packages:

```bash
apt install -y curl wget git ca-certificates openssl
```

## 4. Install Nginx

Install Nginx:

```bash
apt install nginx -y
```

Start Nginx:

```bash
systemctl start nginx
```

Enable Nginx at startup:

```bash
systemctl enable nginx
```

Verify Nginx:

```bash
systemctl status nginx
```

Check the Nginx version:

```bash
nginx -v
```

Test Nginx locally:

```bash
curl -I http://localhost
```

## 5. Install Docker

Download the official Docker installation script:

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
```

Install Docker:

```bash
sh get-docker.sh
```

Start Docker:

```bash
systemctl start docker
```

Enable Docker at startup:

```bash
systemctl enable docker
```

Verify Docker:

```bash
docker --version
```
Add your user to the Docker group

```bash
sudo usermod -aG docker $USER
```
Apply the new group membership
```bash
newgrp docker
```

Step 3. Verify your permissions
```bash
groups
```
Test Docker:

```bash
docker run hello-world
```

## 6. Install Docker Compose

Check whether Docker Compose is already installed:

```bash
docker compose version
```

If the command is not found, install the Compose plugin:

```bash
apt install docker-compose-plugin -y
```

Verify the installation:

```bash
docker compose version
```

## 7. Setup Portainer

Create a persistent Docker volume for Portainer:

```bash
docker volume create portainer_data
```

Run Portainer:

```bash
docker run -d \
    -p 8000:8000 \
    -p 9443:9443 \
    --name portainer \
    --restart=always \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v portainer_data:/data \
    portainer/portainer-ce:lts
```

Verify the Portainer container:

```bash
docker ps
```

Access Portainer through your browser:

```text
https://YOUR_PUBLIC_IP:9443
```

Create the administrator account to manage Docker containers.

## 8. Clone the Application Repositories

Create a project directory:

```bash
mkdir -p ~/fullstack-project
```

Navigate into the directory:

```bash
cd ~/fullstack-project
```

Clone the ReactJS frontend:

```bash
git clone https://github.com/keoKAY/sample-product-reactapp.git
```

Clone the Spring Boot backend:

```bash
git clone https://github.com/keoKAY/spring-sample-product-jpa.git
```

Verify the project directories:

```bash
ls -la
```

Expected structure:

```text
fullstack-project/
├── sample-product-reactapp/
└── spring-sample-product-jpa/
```

## 9. Configure Environment Variables

Create an `.env` file in the project root:

```bash
cd ~/fullstack-project
```

Generate a secure PostgreSQL password:

```bash
printf 'POSTGRES_DB=postgres\nPOSTGRES_USER=postgres\nPOSTGRES_PASSWORD=%s\n' "$(openssl rand -hex 16)" > .env
```

Protect the environment file:

```bash
chmod 600 .env
```

View the configuration:

```bash
cat .env
```

Keep the database password private. Do not commit `.env` to a public Git repository.

## 10. Configure Docker Compose

Create the Compose configuration:

```bash
nano docker-compose.yml
```

Add the following configuration:

```yaml
services:
  postgres:
    image: postgres:16
    container_name: product-postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}
      interval: 10s
      timeout: 5s
      retries: 5
    networks:
      - fullstack-network

  backend:
    container_name: product-backend
    build:
      context: ./spring-sample-product-jpa
      dockerfile: dev.Dockerfile
    restart: unless-stopped
    environment:
      SPRING_DATASOURCE_URL: jdbc:postgresql://postgres:5432/${POSTGRES_DB}
      SPRING_DATASOURCE_USERNAME: ${POSTGRES_USER}
      SPRING_DATASOURCE_PASSWORD: ${POSTGRES_PASSWORD}
    depends_on:
      postgres:
        condition: service_healthy
    volumes:
      - image_backup:/app/filestorage/images
    networks:
      - fullstack-network

  frontend:
    container_name: product-frontend
    build:
      context: ./sample-product-reactapp
      dockerfile: Dockerfile
    restart: unless-stopped
    ports:
      - "3000:80"
    depends_on:
      - backend
    networks:
      - fullstack-network

volumes:
  postgres_data:
  image_backup:

networks:
  fullstack-network:
    driver: bridge
```




## 11. Validate Docker Compose

Navigate to the project root:

```bash
cd ~/fullstack-project
```

Validate the Compose configuration:

```bash
docker compose config
```

If there are no configuration errors, continue to deployment.

## 12. Deploy the Fullstack Application

Build and start all services:

```bash
docker compose up -d --build
```

This command builds and starts:

1. PostgreSQL database container.
2. Spring Boot backend container.
3. ReactJS frontend container.

Check running containers:

```bash
docker ps
```

Check Compose services:

```bash
docker compose ps
```

## 13. Verify Application Logs

View all service logs:

```bash
docker compose logs
```

Follow logs continuously:

```bash
docker compose logs -f
```

Check the backend:

```bash
docker compose logs -f backend
```

Check the frontend:

```bash
docker compose logs -f frontend
```

Check PostgreSQL:

```bash
docker compose logs -f postgres
```

Press `CTRL + C` to stop following the logs.

## 14. Configure Nginx Reverse Proxy

Open the default Nginx configuration:

```bash
nano /etc/nginx/sites-available/default
```

Replace its content with:

```nginx
server {
    listen 80;
    listen [::]:80;

    server_name _;

    location / {
        proxy_pass http://127.0.0.1:3000;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

Test the Nginx configuration:

```bash
nginx -t
sudo nginx -t
```

Reload Nginx:

```bash
sudo systemctl reload nginx

sudo systemctl status nginx
```

Now the frontend can be accessed through port 80 as well as port 3000.

## 15. Configure GCP Firewall Rules

Configure inbound firewall rules in Google Cloud Console.

Navigate to:

**VPC Network → Firewall → Create Firewall Rule**

Allow the following TCP ports:

| Port | Service | Purpose |
|---|---|---|
| 22 | SSH | Remote server access |
| 80 | Nginx | HTTP access |
| 3000 | ReactJS | Frontend access |
| 9443 | Portainer | Container management |

Port 8080 and port 5432 do not need to be publicly accessible for this configuration.

For security, restrict SSH and Portainer access to trusted IP addresses where possible.

## 16. Verify PostgreSQL

Check whether PostgreSQL is ready:

```bash
docker exec product-postgres pg_isready -U postgres
```

Connect to the database:

```bash
docker exec -it product-postgres psql -U postgres -d postgres
```

List databases:

```sql
\l
```

List tables:

```sql
\dt
```

Exit PostgreSQL:

```sql
\q
```

## 17. Verify Backend

Check backend logs:

```bash
docker compose logs --tail=100 backend
```

If Spring Boot Actuator is configured, test its health endpoint:

```bash
docker exec product-backend wget -qO- http://localhost:8080/actuator/health
```

If Actuator is not installed, test an actual API endpoint provided by the application.

## 18. Access the Application

Replace `YOUR_PUBLIC_IP` with your server's external IP address.

| Service | Access URL |
|---|---|
| ReactJS Frontend | `http://YOUR_PUBLIC_IP:3000` |
| Nginx Reverse Proxy | `http://YOUR_PUBLIC_IP` |
| Portainer | `https://YOUR_PUBLIC_IP:9443` |

PostgreSQL and Spring Boot are accessible internally through the Docker network.

## 19. Useful Docker Compose Commands

Run these commands from the project root.

| Operation | Command |
|---|---|
| Start services | `docker compose up -d` |
| Build and start | `docker compose up -d --build` |
| Stop services | `docker compose stop` |
| Restart services | `docker compose restart` |
| View status | `docker compose ps` |
| View logs | `docker compose logs -f` |
| Stop and remove containers | `docker compose down` |

**Warning:** Avoid `docker compose down -v` unless you intentionally want to delete the PostgreSQL volume and its stored data.

## 20. Final Deployment Checklist

- Ubuntu updated successfully.
- Nginx installed and running.
- Docker installed successfully.
- Docker Compose installed.
- Portainer installed and accessible.
- ReactJS repository cloned.
- Spring Boot repository cloned.
- PostgreSQL configured in Docker Compose.
- Docker Compose configuration validated.
- All three application containers running.
- Nginx reverse proxy configured.
- GCP firewall rules configured.
- ReactJS frontend accessible through port 3000.
- Spring Boot successfully connects to PostgreSQL.

## 21. Conclusion

This deployment demonstrates how to configure an Ubuntu server and deploy a fullstack application using containerization.

Docker packages the application components into containers, while Docker Compose manages the frontend, backend, and database together. Nginx provides web server and reverse proxy functionality, and Portainer provides a graphical interface for Docker container management.

The final result is a deployed ReactJS and Spring Boot application connected to a PostgreSQL database and accessible through the server's public IP address.
