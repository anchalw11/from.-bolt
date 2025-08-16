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

2.  **Create a Caddy configuration file** (`Caddyfile`) in the root directory to act as a reverse proxy:

    ```caddy
    your-domain.com {
        # Enable compression
        encode gzip

        # Set up logging
        log {
            output file /var/log/caddy/access.log
        }

        # Reverse proxy API requests to the Flask backend, keeping the /api prefix
        reverse_proxy /api/* journal:5000

        # Serve the static React application and handle client-side routing for the SPA
        root * /usr/share/caddy
        try_files {path} /index.html
        file_server
    }
    ```

3.  **Build and run the application** with Docker Compose:

    ```bash
    docker-compose up --build -d
    ```

## 4. SSL/TLS with Caddy

Caddy automatically handles SSL/TLS certificates for you. As long as your domain is correctly pointing to your server's IP address, Caddy will obtain and renew a certificate from Let's Encrypt for you.

## 5. Managing the Application

-   **Start the application**: `docker-compose up -d`
-   **Stop the application**: `docker-compose down`
-   **View logs**: `docker-compose logs -f <service_name>`

This guide provides a comprehensive overview of the deployment process. You may need to adjust the configurations based on your specific server environment and requirements.
