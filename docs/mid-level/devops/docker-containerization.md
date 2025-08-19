# Docker & Containerization

## Giriş

Docker ve containerization, modern software development ve deployment'da temel teknolojilerdir. Mid-level geliştiriciler için container teknolojilerini anlamak, scalable ve portable uygulamalar geliştirmede kritiktir. Bu dosya, Docker fundamentals, container orchestration, best practices ve .NET uygulamaları için containerization stratejilerini kapsar.

## Docker Fundamentals

### 1. Dockerfile Best Practices
.NET uygulamaları için optimize edilmiş Dockerfile örnekleri.

```dockerfile
# Multi-stage build for .NET application
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base
WORKDIR /app
EXPOSE 80
EXPOSE 443

# Build stage
FROM mcr.microsoft.com/dotnet/sdk:8.0 AS build
WORKDIR /src

# Copy csproj files and restore dependencies
COPY ["src/MyApp/MyApp.csproj", "src/MyApp/"]
COPY ["src/MyApp.Core/MyApp.Core.csproj", "src/MyApp.Core/"]
COPY ["src/MyApp.Infrastructure/MyApp.Infrastructure.csproj", "src/MyApp.Infrastructure/"]
RUN dotnet restore "src/MyApp/MyApp.csproj"

# Copy source code and build
COPY . .
WORKDIR "/src/src/MyApp"
RUN dotnet build "MyApp.csproj" -c Release -o /app/build

# Publish stage
FROM build AS publish
RUN dotnet publish "MyApp.csproj" -c Release -o /app/publish /p:UseAppHost=false

# Final stage
FROM base AS final
WORKDIR /app

# Install additional tools if needed
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# Copy published app
COPY --from=publish /app/publish .

# Create non-root user for security
RUN adduser --disabled-password --gecos '' appuser && chown -R appuser /app
USER appuser

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 2. Docker Compose Configuration
Multi-service uygulamalar için Docker Compose yapılandırması.

```yaml
version: '3.8'

services:
  # API Service
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: final
    ports:
      - "5000:80"
      - "5001:443"
    environment:
      - ASPNETCORE_ENVIRONMENT=Development
      - ConnectionStrings__DefaultConnection=Server=db;Database=MyAppDb;User=sa;Password=Your_password123
      - Redis__ConnectionString=redis:6379
    depends_on:
      - db
      - redis
    networks:
      - myapp-network
    volumes:
      - ./logs:/app/logs
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

  # Database Service
  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD=Your_password123
      - MSSQL_PID=Developer
    ports:
      - "1433:1433"
    volumes:
      - mssql-data:/var/opt/mssql
      - ./init-scripts:/docker-entrypoint-initdb.d
    networks:
      - myapp-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "/opt/mssql-tools/bin/sqlcmd -S localhost -U sa -P Your_password123 -Q 'SELECT 1' || exit 1"]
      interval: 10s
      timeout: 3s
      retries: 10
      start_period: 10s

  # Redis Cache Service
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    volumes:
      - redis-data:/data
    networks:
      - myapp-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5

  # Message Queue Service
  rabbitmq:
    image: rabbitmq:3-management-alpine
    ports:
      - "5672:5672"
      - "15672:15672"
    environment:
      - RABBITMQ_DEFAULT_USER=admin
      - RABBITMQ_DEFAULT_PASS=admin123
    volumes:
      - rabbitmq-data:/var/lib/rabbitmq
    networks:
      - myapp-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "rabbitmq-diagnostics", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5

  # Monitoring Service
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus-data:/prometheus
    networks:
      - myapp-network
    restart: unless-stopped
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
      - '--storage.tsdb.retention.time=200h'
      - '--web.enable-lifecycle'

  # Logging Service
  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.11.0
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ports:
      - "9200:9200"
    volumes:
      - elasticsearch-data:/usr/share/elasticsearch/data
    networks:
      - myapp-network
    restart: unless-stopped
    healthcheck:
      test: ["CMD-SHELL", "curl -f http://localhost:9200/_cluster/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 5

volumes:
  mssql-data:
  redis-data:
  rabbitmq-data:
  prometheus-data:
  elasticsearch-data:

networks:
  myapp-network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16
```

## Container Orchestration

### 1. Docker Swarm Configuration
Docker Swarm ile container orchestration.

```yaml
version: '3.8'

services:
  api:
    image: myapp:latest
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        order: start-first
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      resources:
        limits:
          cpus: '0.50'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
    ports:
      - "5000:80"
    environment:
      - ASPNETCORE_ENVIRONMENT=Production
    networks:
      - myapp-network
    secrets:
      - db_connection_string
      - redis_connection_string

  db:
    image: mcr.microsoft.com/mssql/server:2022-latest
    deploy:
      replicas: 1
      placement:
        constraints:
          - node.role == manager
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
    environment:
      - ACCEPT_EULA=Y
      - SA_PASSWORD_FILE=/run/secrets/db_password
    volumes:
      - mssql-data:/var/opt/mssql
    networks:
      - myapp-network
    secrets:
      - db_password

secrets:
  db_connection_string:
    external: true
  redis_connection_string:
    external: true
  db_password:
    external: true

volumes:
  mssql-data:
    external: true

networks:
  myapp-network:
    driver: overlay
    attachable: true
```

### 2. Kubernetes Deployment
.NET uygulamaları için Kubernetes deployment yapılandırması.

```yaml
# Namespace
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  labels:
    name: myapp

---
# ConfigMap for application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: myapp-config
  namespace: myapp
data:
  appsettings.json: |
    {
      "Logging": {
        "LogLevel": {
          "Default": "Information",
          "Microsoft": "Warning",
          "Microsoft.Hosting.Lifetime": "Information"
        }
      },
      "AllowedHosts": "*",
      "ConnectionStrings": {
        "DefaultConnection": "Server=myapp-db;Database=MyAppDb;User Id=sa;Password=$(DB_PASSWORD);"
      }
    }

---
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: myapp-secrets
  namespace: myapp
type: Opaque
data:
  db-password: U2FQYXNzd29yZDEyMw== # Base64 encoded
  jwt-secret: SlNlY3JldEtleQ== # Base64 encoded

---
# Deployment for API
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp-api
  namespace: myapp
  labels:
    app: myapp-api
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp-api
  template:
    metadata:
      labels:
        app: myapp-api
    spec:
      containers:
      - name: myapp-api
        image: myapp:latest
        ports:
        - containerPort: 80
          name: http
        - containerPort: 443
          name: https
        env:
        - name: ASPNETCORE_ENVIRONMENT
          value: "Production"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: db-password
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: myapp-secrets
              key: jwt-secret
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health/live
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /health/ready
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 3
          failureThreshold: 3
        volumeMounts:
        - name: app-logs
          mountPath: /app/logs
      volumes:
      - name: app-logs
        emptyDir: {}

---
# Service for API
apiVersion: v1
kind: Service
metadata:
  name: myapp-api-service
  namespace: myapp
spec:
  selector:
    app: myapp-api
  ports:
  - name: http
    port: 80
    targetPort: 80
  - name: https
    port: 443
    targetPort: 443
  type: ClusterIP

---
# Ingress for external access
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp-ingress
  namespace: myapp
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - myapp.example.com
    secretName: myapp-tls
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: myapp-api-service
            port:
              number: 80

---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: myapp-api-hpa
  namespace: myapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: myapp-api
  minReplicas: 3
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Percent
        value: 100
        periodSeconds: 15
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Percent
        value: 10
        periodSeconds: 60
```

## Container Security

### 1. Security Scanning
Container güvenlik taraması için Dockerfile ve script'ler.

```dockerfile
# Security-focused Dockerfile
FROM mcr.microsoft.com/dotnet/aspnet:8.0 AS base

# Install security updates
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
        curl \
        ca-certificates \
        && rm -rf /var/lib/apt/lists/*

# Create non-root user
RUN groupadd -r appuser && useradd -r -g appuser appuser

WORKDIR /app
EXPOSE 80

# Switch to non-root user
USER appuser

# Copy application
COPY --chown=appuser:appuser --from=build /app/publish .

# Health check
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
    CMD curl -f http://localhost/health || exit 1

ENTRYPOINT ["dotnet", "MyApp.dll"]
```

### 2. Security Scanning Script
```bash
#!/bin/bash

# Container security scanning script
echo "Starting container security scan..."

# Scan for vulnerabilities
echo "Scanning for vulnerabilities..."
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    aquasec/trivy image myapp:latest

# Check for secrets in image
echo "Checking for secrets..."
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
    wagoodman/dive myapp:latest

# Run security tests
echo "Running security tests..."
docker run --rm -v $(pwd):/app \
    --workdir /app \
    mcr.microsoft.com/dotnet/sdk:8.0 \
    dotnet test --filter Category=Security

echo "Security scan completed."
```

## Container Monitoring

### 1. Prometheus Configuration
```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alert_rules.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093

scrape_configs:
  - job_name: 'myapp-api'
    static_configs:
      - targets: ['myapp-api-service:80']
    metrics_path: '/metrics'
    scrape_interval: 5s

  - job_name: 'database'
    static_configs:
      - targets: ['myapp-db:1433']

  - job_name: 'redis'
    static_configs:
      - targets: ['redis:6379']

  - job_name: 'rabbitmq'
    static_configs:
      - targets: ['rabbitmq:15672']
    metrics_path: '/metrics'
```

### 2. .NET Metrics Configuration
```csharp
public class MetricsConfiguration
{
    public static void ConfigureMetrics(IServiceCollection services, IConfiguration configuration)
    {
        services.AddMetrics();
        
        // Configure Prometheus metrics
        services.AddPrometheusMetrics();
        
        // Configure custom metrics
        services.AddSingleton<ICustomMetrics, CustomMetrics>();
    }
}

public class CustomMetrics
{
    private readonly Counter _requestCounter;
    private readonly Histogram _requestDuration;
    private readonly Gauge _activeConnections;
    
    public CustomMetrics(IMetricsFactory metricsFactory)
    {
        _requestCounter = metricsFactory.CreateCounter("http_requests_total", "Total HTTP requests");
        _requestDuration = metricsFactory.CreateHistogram("http_request_duration_seconds", "HTTP request duration");
        _activeConnections = metricsFactory.CreateGauge("active_connections", "Active database connections");
    }
    
    public void IncrementRequestCount(string endpoint, string method, int statusCode)
    {
        _requestCounter.Increment(1, new KeyValuePair<string, object>("endpoint", endpoint),
            new KeyValuePair<string, object>("method", method),
            new KeyValuePair<string, object>("status_code", statusCode));
    }
    
    public void RecordRequestDuration(string endpoint, double duration)
    {
        _requestDuration.Record(duration, new KeyValuePair<string, object>("endpoint", endpoint));
    }
    
    public void SetActiveConnections(int count)
    {
        _activeConnections.Set(count);
    }
}
```

## Container CI/CD Pipeline

### 1. GitHub Actions Workflow
```yaml
name: Build and Deploy Container

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    
    steps:
    - name: Checkout code
      uses: actions/checkout@v4
      
    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.0.x'
        
    - name: Restore dependencies
      run: dotnet restore
      
    - name: Build
      run: dotnet build --no-restore
      
    - name: Test
      run: dotnet test --no-build --verbosity normal
      
    - name: Run security scan
      run: |
        dotnet tool install --global dotnet-security-scan
        dotnet security-scan
        
    - name: Build Docker image
      run: |
        docker build -t $IMAGE_NAME:${{ github.sha }} .
        docker build -t $IMAGE_NAME:latest .
        
    - name: Run container security scan
      run: |
        docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
          aquasec/trivy image $IMAGE_NAME:${{ github.sha }}
          
    - name: Push to Container Registry
      if: github.ref == 'refs/heads/main'
      run: |
        echo ${{ secrets.GITHUB_TOKEN }} | docker login $REGISTRY -u ${{ github.actor }} --password-stdin
        docker tag $IMAGE_NAME:${{ github.sha }} $REGISTRY/$IMAGE_NAME:${{ github.sha }}
        docker tag $IMAGE_NAME:latest $REGISTRY/$IMAGE_NAME:latest
        docker push $REGISTRY/$IMAGE_NAME:${{ github.sha }}
        docker push $REGISTRY/$IMAGE_NAME:latest
        
  deploy:
    needs: build-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
    - name: Deploy to Kubernetes
      run: |
        # Deploy to staging first
        kubectl apply -f k8s/staging/
        
        # Run smoke tests
        ./scripts/smoke-tests.sh
        
        # Deploy to production
        kubectl apply -f k8s/production/
        
        # Verify deployment
        kubectl rollout status deployment/myapp-api -n myapp
```

### 2. Azure DevOps Pipeline
```yaml
trigger:
  branches:
    include:
    - main
    - develop

variables:
  dockerfilePath: '**/Dockerfile'
  imageRepository: 'myapp'
  containerRegistry: 'azurecr.io'
  dockerfileContext: '$(Build.SourcesDirectory)'
  tag: '$(Build.BuildId)'

stages:
- stage: Build
  displayName: Build and Test
  jobs:
  - job: Build
    displayName: Build
    pool:
      vmImage: 'ubuntu-latest'
    steps:
    - task: UseDotNet@2
      inputs:
        version: '8.0.x'
        
    - script: |
        dotnet restore
        dotnet build --no-restore
        dotnet test --no-build --verbosity normal
      displayName: 'Build and Test .NET Application'
      
    - task: Docker@2
      inputs:
        containerRegistry: 'Azure Container Registry'
        repository: '$(imageRepository)'
        command: 'buildAndPush'
        Dockerfile: '$(dockerfilePath)'
        context: '$(dockerfileContext)'
        tags: |
          $(tag)
          latest
          
    - task: ContainerScan@0
      inputs:
        dockerFilePath: '$(dockerfilePath)'
        dockerContext: '$(dockerfileContext)'
        
- stage: Deploy
  displayName: Deploy
  dependsOn: Build
  condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
  jobs:
  - deployment: Deploy
    displayName: Deploy to Kubernetes
    pool:
      vmImage: 'ubuntu-latest'
    environment: 'production'
    strategy:
      runOnce:
        deploy:
          steps:
          - task: KubernetesManifest@0
            inputs:
              action: 'deploy'
              kubernetesServiceConnection: 'myapp-k8s-connection'
              manifests: |
                $(Pipeline.Workspace)/manifests/*.yml
              containers: |
                $(containerRegistry)/$(imageRepository):$(tag)
```

## Mülakat Soruları

### Temel Sorular

1. **Docker nedir ve neden kullanılır?**
   - **Cevap**: Containerization platform, uygulamaları izole eder, portable yapar, consistent environment sağlar.

2. **Container vs Virtual Machine arasındaki fark nedir?**
   - **Cevap**: Container OS kernel'i paylaşır, daha hafif, hızlı startup. VM tam OS çalıştırır, daha ağır.

3. **Multi-stage build nedir ve neden kullanılır?**
   - **Cevap**: Build ve runtime için farklı image'lar, final image boyutunu küçültür, security artırır.

4. **Docker Compose vs Kubernetes arasındaki fark nedir?**
   - **Cevap**: Compose local development için, Kubernetes production orchestration için, scaling ve high availability.

5. **Container security best practices nelerdir?**
   - **Cevap**: Non-root user, minimal base image, security scanning, regular updates, secrets management.

### Teknik Sorular

1. **Dockerfile'da ENTRYPOINT vs CMD arasındaki fark nedir?**
   - **Cevap**: ENTRYPOINT executable'ı belirler, CMD default parameters sağlar. ENTRYPOINT override edilemez.

2. **Container health check nasıl implement edilir?**
   - **Cevap**: HEALTHCHECK instruction, endpoint'te /health endpoint, liveness/readiness probes.

3. **Container networking nasıl çalışır?**
   - **Cevap**: Bridge, host, overlay networks, port mapping, service discovery.

4. **Container orchestration'da scaling nasıl yapılır?**
   - **Cevap**: Horizontal scaling, auto-scaling, load balancing, resource limits.

5. **Container monitoring ve logging nasıl implement edilir?**
   - **Cevap**: Prometheus metrics, centralized logging, health checks, alerting.

## Best Practices

1. **Dockerfile Optimization**
   - Multi-stage build kullanın
   - Layer caching optimize edin
   - Minimal base image kullanın
   - Security updates yapın

2. **Container Security**
   - Non-root user kullanın
   - Security scanning implement edin
   - Secrets management yapın
   - Regular updates yapın

3. **Performance Optimization**
   - Resource limits belirleyin
   - Health checks implement edin
   - Monitoring ve logging kurun
   - Auto-scaling yapın

4. **CI/CD Integration**
   - Automated testing yapın
   - Security scanning ekleyin
   - Blue-green deployment yapın
   - Rollback strategy hazırlayın

5. **Monitoring & Observability**
   - Metrics collection yapın
   - Centralized logging kurun
   - Alerting implement edin
   - Performance monitoring yapın

## Kaynaklar

- [Docker Official Documentation](https://docs.docker.com/)
- [Microsoft Container Documentation](https://docs.microsoft.com/en-us/dotnet/architecture/cloud-native/)
- [Kubernetes Documentation](https://kubernetes.io/docs/)
- [Container Security Best Practices](https://cloud.google.com/security/best-practices-for-containers)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
