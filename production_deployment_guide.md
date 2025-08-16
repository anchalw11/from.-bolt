# Production Deployment Guide

This guide provides instructions for deploying the application to a production environment.

## Prerequisites

- Node.js and npm
- Python 3 and pip
- A production-ready web server (e.g., Gunicorn, uWSGI)
- A reverse proxy (e.g., Nginx, Caddy)

## 1. Install Dependencies

### Frontend

Install the frontend dependencies using npm:

```bash
npm install
```

### Backend

Install the backend dependencies using pip:

```bash
pip install -r journal/requirements.txt
```

## 2. Build the Frontend

Build the frontend for production using the `build` script:

```bash
npm run build
```

This will create a `dist` directory with the optimized and minified frontend assets.

## 3. Run the Backend

The backend is a Flask application that can be run using a production-ready web server like Gunicorn.

### Example with Gunicorn

Install Gunicorn:

```bash
pip install gunicorn
```

Run the application with Gunicorn:

```bash
gunicorn --bind 0.0.0.0:5000 wsgi:app
```

This will start the Flask application on port 5000.

## 4. Configure a Reverse Proxy

A reverse proxy is recommended for serving the frontend and proxying API requests to the backend.

### Example with Nginx

Create an Nginx configuration file with the following content:

```nginx
server {
    listen 80;
    server_name your_domain.com;

    location / {
        root /path/to/your/project/dist;
        try_files $uri /index.html;
    }

    location /api {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

Replace `your_domain.com` with your domain and `/path/to/your/project/dist` with the actual path to the `dist` directory.

## 5. Environment Variables

Create a `.env.production` file in the root of the project with the following environment variables:

```
FLASK_ENV=production
DATABASE_URL=your_production_database_url
SECRET_KEY=your_production_secret_key
```

Replace the values with your production configuration.
