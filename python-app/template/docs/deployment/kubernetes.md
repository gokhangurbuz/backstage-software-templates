# Kubernetes Deployment

Complete guide for deploying ${{values.app_name}} to Kubernetes.

## Prerequisites

- Kubernetes cluster (1.27+)
- kubectl configured
- Helm 3.x (optional)
- Container image built and pushed

## Deployment Architecture
```mermaid
graph TB
    subgraph "Ingress Layer"
        A[Ingress Controller]
    end
    
    subgraph "Service Layer"
        B[Service]
    end
    
    subgraph "Application Layer"
        C[Pod 1]
        D[Pod 2]
        E[Pod 3]
    end
    
    subgraph "Configuration"
        F[ConfigMap]
        G[Secret]
    end
    
    A --> B
    B --> C
    B --> D
    B --> E
    C -.-> F
    C -.-> G
    D -.-> F
    D -.-> G
    E -.-> F
    E -.-> G
```

## Kubernetes Manifests

### Namespace
```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: ${{values.app_name}}
  labels:
    name: ${{values.app_name}}
    environment: ${{values.app_env}}
```

### ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: ${{values.app_name}}-config
  namespace: ${{values.app_name}}
data:
  APP_ENV: "${{values.app_env}}"
  PORT: "8080"
  LOG_LEVEL: "INFO"
```

### Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: ${{values.app_name}}-secret
  namespace: ${{values.app_name}}
type: Opaque
data:
  # Base64 encoded values
  API_KEY: <base64-encoded-key>
```

### Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${{values.app_name}}
  namespace: ${{values.app_name}}
  labels:
    app: ${{values.app_name}}
    version: v1
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: ${{values.app_name}}
  template:
    metadata:
      labels:
        app: ${{values.app_name}}
        version: v1
    spec:
      containers:
      - name: ${{values.app_name}}
        image: your-registry/${{values.app_name}}:latest
        imagePullPolicy: Always
        ports:
        - containerPort: 8080
          name: http
        envFrom:
        - configMapRef:
            name: ${{values.app_name}}-config
        - secretRef:
            name: ${{values.app_name}}-secret
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "200m"
        livenessProbe:
          httpGet:
            path: /api/v1/healthz
            port: 8080
          initialDelaySeconds: 10
          periodSeconds: 30
          timeoutSeconds: 5
          failureThreshold: 3
        readinessProbe:
          httpGet:
            path: /api/v1/healthz
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
          timeoutSeconds: 3
          failureThreshold: 3
```

### Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: ${{values.app_name}}
  namespace: ${{values.app_name}}
  labels:
    app: ${{values.app_name}}
spec:
  type: ClusterIP
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
  selector:
    app: ${{values.app_name}}
```

### Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: ${{values.app_name}}
  namespace: ${{values.app_name}}
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - ${{values.app_name}}-${{values.app_env}}.roofstacks.com
    secretName: ${{values.app_name}}-tls
  rules:
  - host: ${{values.app_name}}-${{values.app_env}}.roofstacks.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: ${{values.app_name}}
            port:
              number: 80
```

## Deployment Steps

### 1. Build and Push Image
```bash
# Build Docker image
docker build -t your-registry/${{values.app_name}}:latest .

# Tag with version
docker tag your-registry/${{values.app_name}}:latest \
  your-registry/${{values.app_name}}:1.0.0

# Push to registry
docker push your-registry/${{values.app_name}}:latest
docker push your-registry/${{values.app_name}}:1.0.0
```

### 2. Create Namespace
```bash
kubectl create namespace ${{values.app_name}}
```

### 3. Apply Configuration
```bash
# Apply ConfigMap
kubectl apply -f k8s/configmap.yaml

# Apply Secret
kubectl apply -f k8s/secret.yaml

# Apply Deployment
kubectl apply -f k8s/deployment.yaml

# Apply Service
kubectl apply -f k8s/service.yaml

# Apply Ingress
kubectl apply -f k8s/ingress.yaml
```

### 4. Verify Deployment
```bash
# Check deployment status
kubectl get deployments -n ${{values.app_name}}

# Check pods
kubectl get pods -n ${{values.app_name}}

# Check service
kubectl get svc -n ${{values.app_name}}

# Check ingress
kubectl get ingress -n ${{values.app_name}}
```

### 5. View Logs
```bash
# View pod logs
kubectl logs -f -l app=${{values.app_name}} -n ${{values.app_name}}

# View logs from specific pod
kubectl logs -f <pod-name> -n ${{values.app_name}}
```

## Scaling

### Manual Scaling
```bash
# Scale to 5 replicas
kubectl scale deployment ${{values.app_name}} \
  --replicas=5 \
  -n ${{values.app_name}}
```

### Horizontal Pod Autoscaler
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: ${{values.app_name}}-hpa
  namespace: ${{values.app_name}}
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: ${{values.app_name}}
  minReplicas: 2
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
```

Apply HPA:
```bash
kubectl apply -f k8s/hpa.yaml
```

## Rolling Updates

### Update Deployment
```bash
# Update image
kubectl set image deployment/${{values.app_name}} \
  ${{values.app_name}}=your-registry/${{values.app_name}}:1.1.0 \
  -n ${{values.app_name}}

# Watch rollout status
kubectl rollout status deployment/${{values.app_name}} \
  -n ${{values.app_name}}
```

### Rollback Deployment
```bash
# Rollback to previous version
kubectl rollout undo deployment/${{values.app_name}} \
  -n ${{values.app_name}}

# Rollback to specific revision
kubectl rollout undo deployment/${{values.app_name}} \
  --to-revision=2 \
  -n ${{values.app_name}}
```

### View Rollout History
```bash
kubectl rollout history deployment/${{values.app_name}} \
  -n ${{values.app_name}}
```

## Resource Management

### Resource Quotas
```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: ${{values.app_name}}-quota
  namespace: ${{values.app_name}}
spec:
  hard:
    requests.cpu: "4"
    requests.memory: 8Gi
    limits.cpu: "8"
    limits.memory: 16Gi
    pods: "20"
```

### Limit Ranges
```yaml
apiVersion: v1
kind: LimitRange
metadata:
  name: ${{values.app_name}}-limits
  namespace: ${{values.app_name}}
spec:
  limits:
  - max:
      cpu: "1"
      memory: 1Gi
    min:
      cpu: "100m"
      memory: 128Mi
    default:
      cpu: "200m"
      memory: 256Mi
    defaultRequest:
      cpu: "100m"
      memory: 128Mi
    type: Container
```

## Network Policies
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ${{values.app_name}}-netpol
  namespace: ${{values.app_name}}
spec:
  podSelector:
    matchLabels:
      app: ${{values.app_name}}
  policyTypes:
  - Ingress
  - Egress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: ingress-nginx
    ports:
    - protocol: TCP
      port: 8080
  egress:
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 443
  - to:
    - namespaceSelector: {}
    ports:
    - protocol: TCP
      port: 53
    - protocol: UDP
      port: 53
```

## Monitoring Integration

### ServiceMonitor (Prometheus Operator)
```yaml
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: ${{values.app_name}}
  namespace: ${{values.app_name}}
spec:
  selector:
    matchLabels:
      app: ${{values.app_name}}
  endpoints:
  - port: http
    path: /metrics
    interval: 30s
```

## Troubleshooting

### Pod Not Starting
```bash
# Describe pod
kubectl describe pod <pod-name> -n ${{values.app_name}}

# Check events
kubectl get events -n ${{values.app_name}} --sort-by='.lastTimestamp'

# Check logs
kubectl logs <pod-name> -n ${{values.app_name}} --previous
```

### Service Not Accessible
```bash
# Test service internally
kubectl run -it --rm debug --image=busybox --restart=Never -- \
  wget -qO- http://${{values.app_name}}.${{values.app_name}}.svc.cluster.local/api/v1/healthz

# Check endpoints
kubectl get endpoints -n ${{values.app_name}}
```

### Ingress Issues
```bash
# Describe ingress
kubectl describe ingress ${{values.app_name}} -n ${{values.app_name}}

# Check ingress controller logs
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx
```

## Helm Deployment (Optional)

### Chart Structure
```
helm/
├── Chart.yaml
├── values.yaml
├── values-dev.yaml
├── values-prod.yaml
└── templates/
    ├── deployment.yaml
    ├── service.yaml
    ├── ingress.yaml
    ├── configmap.yaml
    └── secret.yaml
```

### Deploy with Helm
```bash
# Install chart
helm install ${{values.app_name}} ./helm \
  -n ${{values.app_name}} \
  --create-namespace \
  -f helm/values-${{values.app_env}}.yaml

# Upgrade release
helm upgrade ${{values.app_name}} ./helm \
  -n ${{values.app_name}} \
  -f helm/values-${{values.app_env}}.yaml

# Uninstall release
helm uninstall ${{values.app_name}} -n ${{values.app_name}}
```

## Next Steps

- [Configuration Management](configuration.md)
- [Environment Variables](environment.md)
- [Monitoring Setup](../operations/monitoring.md)