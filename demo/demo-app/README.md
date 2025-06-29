# Demo Golang Gin API Application

A simple "Hello World" REST API built with Go and Gin framework, containerized with Docker.

## API Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| GET | `/` | Returns "Hello World" message |
| GET | `/hello/:name` | Returns personalized greeting |
| GET | `/health` | Health check endpoint |

## Docker

### Prerequisites

- Docker and Docker Compose installed
- Go 1.24+ (for local development)

### Local Development

1. **Install dependencies:**

   ```bash
   go mod download
   ```

2. **Run the application:**

   ```bash
   go run main.go
   ```

3. **Build for production:**

   ```bash
   go build -o demo-app main.go
   ```

### Using Docker directly

1. **Build the image:**

   ```bash
   docker build -t demo-app .
   ```

2. **Run the container:**

   ```bash
   docker run -p 8080:8080 demo-app
   ```

### Using Docker Compose

1. **Build and run the application:**

   ```bash
   docker-compose up --build
   ```

2. **Test the API:**

   ```bash
   # Hello World
   curl http://localhost:8080/

   # Custom greeting
   curl http://localhost:8080/hello/Nico

   # Health check
   curl http://localhost:8080/health
   ```

3. **Run with Nginx reverse proxy:**

   ```bash
   docker-compose --profile with-nginx up --build
   ```

   Access via: http://localhost (port 80)

## Kubernetes Deployment

### Prerequisites

- Docker installed and running
- kubectl installed
- kind (Kubernetes in Docker) installed

**Create a kind cluster:**

```bash
# Create a simple cluster
kind create cluster --name demo-cluster
```

### Deploy to Kubernetes

1. **Build and tag the Docker image:**

   ```bash
   docker build -t demo-app:latest .
   
   kind load docker-image --name demo-cluster demo-app:latest
   ```

2. **Deploy the application:**

   ```bash
   kubectl apply -f k8s-manifest.yaml
   ```

3. **Verify deployment:**

   ```bash
   # Check pods
   kubectl get pods -l app=demo-app
   
   # Check service
   kubectl get svc demo-app-service
   
   # Check ingress
   kubectl get ingress demo-app-ingress
   ```

4. **Access the application:**

   ```bash
   # Port forward to the service
   kubectl port-forward svc/demo-app-service 8080:80 &
   
   curl http://localhost:8080/
   curl http://localhost:8080/hello/Nico
   curl http://localhost:8080/health
   ```
