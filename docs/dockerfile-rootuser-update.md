# Dockerfile Security Updates

## Key Changes Made

### 1. Non-Root User Implementation
```dockerfile
# Created non-root user
ARG UID=1001
ARG GID=1001
RUN addgroup -g ${GID} -S appuser && \
    adduser -S -D -H -u ${UID} -s /sbin/nologin -G appuser appuser

# Created app directory with proper ownership
RUN mkdir -p /app && chown -R appuser:appuser /app

# Switch to non-root user
USER appuser

# Proper File Ownership
COPY --from=builder --chown=appuser:appuser /go/src/accountapi/app .
```

### 2. Changed port(80 -> 8080) in Docker-compose.yaml
```bash
    # Docker-compose.yaml
    ports:
      - "7007:8080" (host:container) Traffic coming to host machine gets forwarded to port 8080 inside the container where your application is listening.
    environment:
      HTTP_PLATFORM_PORT: 8080
```
```bash
    # Run to pull latest image and recreate the container making changes
    docker-compose pull accounts-api-service
    docker-compose up -d accounts-api-service
```
```bash
    # To check the container user (mfsev11)
    docker ps
    docker exec -it <container_name> whoami

    # Docker-compose
    docker-compose exec -it <service-name> whoami
```

### 3. Kubernetes
```bash
    # Deployment
    ports:
        - containerPort: 8080
        env:
        - name: HTTP_PLATFORM_PORT
          value: "8080"
````
```bash
    # services
    # Added targetPort
    ports:
        - protocol: TCP
        port: 80
        targetPort: 8080 
```
```bash
    # Apply the recent changes
    kubectl apply -f accounts-api-service.yaml -n mfdev
    kubectl rollout restart deployment accounts-api-service-deployment -n mfdev  
```
```bash
    # Check user in pod container
    kubectl exec -it deployment/accounts-api-service-deployment -n mfdev -- whoami
```

###  Security Update: Docker Container Privilege Escalation Risk

I identified a security vulnerability in our Dockerfiles where containers are running as the ROOT user. This creates a significant risk. If an attacker gains access to our application, they could escalate to root privileges, escape the container, and potentially access the host system.

  I've been able to address this issue in the accounts-api-service by implementing a non-root user (appuser). This wil:
  - Limits potential damage if the container is compromised
  - Follows container security best practices
  - Maintains application functionality

This security fix should be applied to all our Dockerized applications to ensure consistent protection across our infrastructure.