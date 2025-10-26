# Blue/Green Deployment Strategy

**Author:** Dominic Ifechuku

## Overview

This project implements a Blue/Green deployment strategy for a Node.js service using Nginx as a reverse proxy and load balancer. The system is designed to provide zero-downtime failover between two identical service instances using health-based automatic failover.

### Key Features

- Automatic failover from Blue to Green services on failure
- Zero-downtime failover
- Automatic retry mechanism within a single client request context
- Health-based failover

### System Overview

The system consists of three Docker containers:

1. **Nginx Container [load-balancer]**: Acts as the reverse proxy and load balancer, handling all incoming client requests on port 8080
2. **Blue App Container [blue-service]**: Primary Node.js service instance exposed on port 8081
3. **Green App Container [green-service]**: Backup Node.js service instance exposed on port 8082

### Application Endpoints

- **GET /version**: Returns version information with custom headers
  - `X-App-Pool`: Identifies which pool (blue or green) handled the request
  - `X-Release-Id`: Contains the release identifier for the instance
  
- **POST /chaos/start**: Simulates service downtime by making the instance return 500 errors or timeout
  
- **POST /chaos/stop**: Ends the simulated downtime 

- **GET /healthz**: Health check endpoint for process liveness verification

## Prerequisites

Before deploying this project, ensure you have the following installed:

- Docker Engine 20.10 or later
- Docker Compose 2.0 or later
- curl or similar HTTP client for testing
- git

## Quick Start

### Step 1: Clone the Repository
```bash
git clone https://github.com/Dom-HTG/hng13-stage2-devops
cd hng13-stage2-devops
```

### Step 2: Configure Environment Variables

Create a new .env file in the project root directory, copy existing content and refactor variable values as needed

```bash
cp sample.env .env
```

### Step 3: Start the Services
```bash
docker-compose up -d
```

This command will:
- Pull the specified Docker images for Blue and Green instances
- Generate the Nginx configuration from the template
- Start all three containers in detached mode

### Step 4: Verify Deployment
```bash
curl http://localhost:8080/version
```

Expected response should include headers indicating the Blue instance is serving traffic:
- `X-App-Pool: blue`
- `X-Release-Id: v1.0.0-blue`

## Environment Variable Configuration

The following configuration for environment variables is required:
```bash
BLUE_IMAGE=blue:sample-blue-image # docker image for blue service.
GREEN_IMAGE=green:sample-green-image # docker image for green service.
ACTIVE_POOL=blue # current active service [for manual toggle of primary production traffic].
RELEASE_ID_BLUE=v1.0.0-blue # <string identifier> unique ID of current blue release.
RELEASE_ID_GREEN=v1.0.0-green # <string identifier> unique ID of current green release.
PORT=8080 # application internal port.
```

## Deployment Mechanics

### Initial Startup Sequence

1. Docker Compose reads the `.env` file and substitutes variables
2. The Nginx configuration template is processed with `envsubst` to generate the final configuration
3. Docker pulls the specified images 
4. Containers are created and started in order (services first, then Nginx)
5. Nginx begins routing traffic to the active pool (Blue by default)

### Configuration Template Generation

The project uses environment variable substitution to generate the Nginx configuration dynamically. This is handled through a script that runs first when attempting to spin an NGINX container from docker-compose:
```bash
#!/bin/bash
envsubst '${ACTIVE_POOL}' < /etc/nginx/nginx.conf.template > /etc/nginx/nginx.conf
nginx -g 'daemon off;'
```

### Request Retry Logic

When a request to Blue fails, Nginx retries to Green **within the same client request**. This means:

- The client sends one HTTP request
- Nginx attempts to proxy to Blue (fails)
- Nginx immediately retries to Green (succeeds)
- Client receives one HTTP response with status 200
- From the client's perspective, there was no failure (all was handled within same request context)

This retry mechanism ensures zero failed client requests during failover.

### Configuration Reload

To update the active pool without downtime:

1. Modify the `ACTIVE_POOL` variable in `.env`
2. Regenerate the Nginx configuration:
```bash
   docker-compose exec load-balancer /bin/sh -c "envsubst < /etc/nginx/nginx.conf.template > /etc/nginx/nginx.conf"
```
3. Reload Nginx:
```bash
   docker-compose exec load-balancer nginx -s reload
```

## Testing and Verification

