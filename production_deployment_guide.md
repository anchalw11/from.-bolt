# Production Deployment Guide

This guide provides instructions for setting up and deploying the application in a production environment.

## 1. Prerequisites

- Docker and Docker Compose installed on your server.
- A domain name pointing to your server's IP address.
- A production-ready database (e.g., PostgreSQL, MongoDB).

## 2. Initial Setup

1.  **Clone the repository** to your production server.
2.  **Run the setup script** to install all dependencies and build the frontend:

    ```bash
    chmod +x setup_production.sh
    ./setup_production.sh
    ```

3.  **Configure environment variables**:

    -   Copy the `.env.production` file to `.env`:
        ```bash
        cp .env.production .env
        ```
    -   Edit the `.env` file and replace the placeholder values with your actual production secrets and configuration.

## 3. Dockerized Deployment

This project is configured to run with Docker Compose, which simplifies the management of the multi-service application.

1.  **Create Dockerfiles** for each service. You will need to create a `Dockerfile` in each service's directory (`journal`, `customer-service`, etc.) and a `Dockerfile.frontend` in the root directory.

    -   **`Dockerfile.frontend` (root directory):**
        ```dockerfile
        # Use a multi-stage build to keep the final image small
        # 1. Build the React app
        FROM node:18 AS build
        WORKDIR /app
        COPY package*.json ./
        RUN npm install
        COPY . .
        RUN npm run build

        # 2. Serve the app with Nginx
        FROM nginx:stable-alpine
        COPY --from=build /app/dist /usr/share/nginx/html
        COPY nginx.conf /etc/nginx/conf.d/default.conf
        EXPOSE 80
        CMD ["nginx", "-g", "daemon off;"]
        ```

    -   **`Dockerfile` for Node.js services (e.g., `customer-service`):**
        ```dockerfile
        FROM node:18-alpine
        WORKDIR /app
        COPY package*.json ./
        RUN npm install --production
        COPY . .
        CMD ["node", "server.js"]
        ```

    -   **`Dockerfile` for Python services (e.g., `journal`):**
        ```dockerfile
        FROM python:3.9-slim
        WORKDIR /app
        COPY requirements.txt .
        RUN pip install --no-cache-dir -r requirements.txt
        COPY . .
        CMD ["gunicorn", "--bind", "0.0.0.0:5000", "wsgi:application"]
        ```

2.  **Create an Nginx configuration file** (`nginx.conf`) in the root directory to act as a reverse proxy:

    ```nginx
    server {
        listen 80;
        server_name your-domain.com;

        location / {
            root   /usr/share/nginx/html;
            index  index.html index.htm;
            try_files $uri $uri/ /index.html;
        }

        location /api/journal {
            proxy_pass http://journal:5000;
        }

        location /api/customer-service {
            proxy_pass http://customer_service:3001;
        }

        # Add other services here...
    }
    ```

3.  **Build and run the application** with Docker Compose:

    ```bash
    docker-compose up --build -d
    ```

## 4. SSL/TLS with Let's Encrypt

For a production environment, it is highly recommended to use HTTPS. You can use Certbot to obtain a free SSL/TLS certificate from Let's Encrypt.

1.  **Modify your `docker-compose.yml`** to include Certbot:

    ```yaml
    # ... (other services)

    certbot:
      image: certbot/certbot
      volumes:
        - ./certbot/conf:/etc/letsencrypt
        - ./certbot/www:/var/www/certbot
      entrypoint: "/bin/sh -c 'trap exit TERM; while :; do certbot renew; sleep 12h & wait $${!}; done;'"
    ```

2.  **Obtain the certificate**:

    ```bash
    docker-compose run --rm --entrypoint "\
      certbot certonly --webroot -w /var/www/certbot \
      --email your-email@example.com -d your-domain.com \
      --rsa-key-size 4096 --agree-tos --no-eff-email --force-renewal" certbot
    ```

3.  **Update your Nginx configuration** to use the SSL certificate and redirect HTTP to HTTPS.

## 5. Managing the Application

-   **Start the application**: `docker-compose up -d`
-   **Stop the application**: `docker-compose down`
-   **View logs**: `docker-compose logs -f <service_name>`

This guide provides a comprehensive overview of the deployment process. You may need to adjust the configurations based on your specific server environment and requirements.
