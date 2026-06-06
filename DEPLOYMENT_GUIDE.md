# Deployment & Configuration Guide

This document outlines the changes needed to fully implement the enhanced CI/CD pipeline.

---

## 📋 Implementation Checklist

### ✅ Phase 1: GitHub Secrets Setup

1. **Docker Hub Access**
   ```bash
   # Navigate to: https://hub.docker.com/settings/security/tokens/new
   # Create token with read/write access
   # Add as GitHub Secret: DOCKER_USERNAME, DOCKER_PASSWORD
   ```

2. **Kubernetes Configuration**
   ```bash
   # Get base64-encoded kubeconfig
   cat ~/.kube/config | base64 -w 0 | clip
   # Add as GitHub Secret: KUBE_CONFIG
   # Add target namespace: KUBE_NAMESPACE (default: 'default')
   ```

3. **Verify Secrets**
   - Go to: Settings → Secrets and variables → Actions
   - Ensure all 4 secrets are present
   - Never expose secret values

---

### ✅ Phase 2: Kubernetes Cluster Setup

#### 1. Verify Deployments Exist
```bash
kubectl get deployments

# Expected output:
# NAME                      READY   UP-TO-DATE   AVAILABLE   AGE
# astro-deployment          2/2     2            2           5d
# backend-v2-deployment     2/2     2            2           5d
```

If missing, create them:
```bash
kubectl apply -f kubernetes/frontend/deployment-v1-enhanced.yaml
kubectl apply -f kubernetes/backend/backend-v2-enhanced.yaml
```

#### 2. Verify Services Exist
```bash
kubectl get svc

# Expected output:
# NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)   AGE
# astro-service          ClusterIP   10.96.140.1     <none>        80/TCP    5d
# backend-v2-service     ClusterIP   10.96.230.100   <none>        80/TCP    5d
```

#### 3. Verify Health Check Endpoints
```bash
# Test frontend
kubectl port-forward svc/astro-service 8080:80
curl http://localhost:8080/

# Test backend
kubectl port-forward svc/backend-v2-service 3000:80
curl http://localhost:3000/api/v1/healthCheck
```

#### 4. Setup Pod Disruption Budgets
```bash
kubectl apply -f kubernetes/frontend/deployment-v1-enhanced.yaml
kubectl apply -f kubernetes/backend/backend-v2-enhanced.yaml

# Verify PDB
kubectl get pdb
```

#### 5. Verify Ingress Configuration
```bash
kubectl get ingress

# Should show:
# NAME              CLASS   HOSTS      ADDRESS        PORTS   AGE
# app-ingress       nginx   app.local  192.168.1.100  80      5d
```

---

### ✅ Phase 3: Application Updates

#### Frontend (Astro)

**Ensure health check works:**
```nginx
# nginx.conf
location / {
    try_files $uri $uri/ /index.html;
}
```

**Dockerfile - Already optimized** ✅
```dockerfile
FROM node:22-alpine AS builder
# ... builds to /app/dist/client

FROM nginx:stable-alpine
COPY --from=builder /app/dist/client /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
```

#### Backend (Express)

**Add health check endpoint:**
```javascript
// server.js or app.js
app.get('/api/v1/healthCheck', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date(),
    version: process.env.VERSION || 'v1',
    uptime: process.uptime()
  });
});
```

**Graceful shutdown:**
```javascript
import http from 'http';

const server = http.createServer(app);

process.on('SIGTERM', () => {
  console.log('SIGTERM received, closing gracefully...');
  
  server.close(() => {
    console.log('Server closed');
    process.exit(0);
  });
  
  // Force shutdown after 10 seconds
  setTimeout(() => {
    console.error('Forced shutdown');
    process.exit(1);
  }, 10000);
});

server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

---

### ✅ Phase 4: GitHub Actions Verification

#### Test the Pipeline

1. **Create test branch:**
   ```bash
   git checkout -b test/cicd
   git push origin test/cicd
   ```

2. **Verify lint jobs run:**
   - Go to: Actions tab
   - Should see: lint-frontend ✅, lint-backend ✅

3. **Push to main to trigger full pipeline:**
   ```bash
   git checkout main
   git merge test/cicd
   git push origin main
   ```

4. **Monitor deployment:**
   - Actions tab shows: build → deploy → notify
   - Check kubectl for pod updates
   - Verify health checks pass

#### Debug Failed Deployments
```bash
# Check pod status
kubectl get pods -o wide

# View pod logs
kubectl logs -f deployment/astro-deployment

# Check rollout status
kubectl rollout status deployment/astro-deployment --watch

# Check events
kubectl describe pod <pod-name>
```

---

### ✅ Phase 5: Monitoring & Logging (Optional but Recommended)

#### 1. Enable Pod Logging
```bash
# View recent logs
kubectl logs deployment/astro-deployment --tail=50 -f

# View logs from specific pod
kubectl logs <pod-name> -f

# View logs with timestamps
kubectl logs deployment/astro-deployment -f --timestamps=true
```

#### 2. Prometheus Metrics (if available)
```bash
# Access backend metrics
kubectl port-forward svc/backend-v2-service 3000:80
curl http://localhost:3000/metrics
```

#### 3. Pod Events
```bash
# Watch pod events in real-time
kubectl get events --watch

# Get recent events
kubectl get events --sort-by='.lastTimestamp'
```

---

## 🔍 Troubleshooting Guide

### Issue: `ImagePullBackOff`

**Cause:** Docker image not found or authentication failed

**Solution:**
```bash
# Check image exists in registry
docker pull himanshux99/astro-frontend:latest

# Verify Docker credentials in GitHub
# Settings → Secrets → DOCKER_USERNAME/DOCKER_PASSWORD

# Check pod events
kubectl describe pod <pod-name>
```

---

### Issue: Health Check Fails

**Cause:** Endpoint not responding or misconfigured

**Solution:**
```bash
# Test endpoint manually
kubectl exec -it <pod-name> -- curl http://localhost:80/

# For backend
kubectl exec -it <pod-name> -- curl http://localhost:3000/api/v1/healthCheck

# Check if pods are running
kubectl get pods

# View pod logs for errors
kubectl logs <pod-name>
```

---

### Issue: Deployment Stuck in Rollout

**Cause:** Pod failed to start, resource limits too low

**Solution:**
```bash
# Check pod resource usage
kubectl top pod <pod-name>

# Check for errors
kubectl describe pod <pod-name>

# Increase resource limits in deployment
kubectl edit deployment astro-deployment
# Increase resources.limits.memory/cpu

# Force rollback
kubectl rollout undo deployment/astro-deployment
```

---

### Issue: Automatic Rollback Triggered

**Cause:** Deployment failed health checks

**Investigation:**
```bash
# Check which version is current
kubectl get deployment astro-deployment -o wide

# View rollout history
kubectl rollout history deployment/astro-deployment

# Get detailed status
kubectl rollout status deployment/astro-deployment -v=2

# Check pod readiness
kubectl get pods -o custom-columns=\
NAME:.metadata.name,\
READY:.status.conditions[?(@.type=="Ready")].status,\
AGE:.metadata.creationTimestamp
```

---

### Issue: CI/CD Fails on Pull Request

**Expected Behavior:**
- ✅ Lint jobs run
- ❌ Build jobs skipped (only on push to main)
- ❌ Deploy job skipped

**If different, check:**
```yaml
# In ci-cd.yaml
if: github.event_name == 'push'  # Build only on push
if: github.event_name == 'push' && github.ref == 'refs/heads/main'  # Deploy only on main
```

---

## 📊 Performance Tuning

### Cache Management

**Node Dependencies Cache:**
```bash
# Cache is automatically managed
# Clearing cache: Settings → Actions → Caches → Delete all
```

**Docker Layer Cache:**
```bash
# Already enabled via GitHub Actions cache
# No action needed
```

### Optimize Build Time

1. **Use .dockerignore:**
   ```
   node_modules
   npm-debug.log
   .git
   .gitignore
   README.md
   dist/
   ```

2. **Optimize dependencies:**
   ```bash
   # Remove unused packages
   npm audit
   npm prune
   ```

3. **Monitor build logs:**
   - Actions → Workflow run → Build job
   - Note which steps take longest
   - Optimize those specifically

---

## 🔐 Security Enhancements (Optional)

### 1. Enable Dependency Scanning
```yaml
# Add to ci-cd.yaml
- name: Dependency Check
  run: npm audit --production
```

### 2. Docker Image Scanning
```yaml
- name: Scan Image
  uses: aquasecurity/trivy-action@master
  with:
    image-ref: 'himanshux99/astro-frontend:latest'
    format: 'sarif'
```

### 3. Limit Service Account Permissions
```bash
# Create RBAC role for CI/CD
kubectl create serviceaccount ci-cd-user
kubectl create rolebinding ci-cd-deployment \
  --clusterrole=edit \
  --serviceaccount=default:ci-cd-user
```

---

## ✨ Next Steps

1. ✅ Set up GitHub Secrets (REQUIRED)
2. ✅ Verify Kubernetes cluster configuration
3. ✅ Add health check endpoint to backend
4. ✅ Test on non-main branch first
5. ✅ Monitor logs during first production deployment
6. ✅ Set up notifications (Slack, Teams, etc.)

---

## 📞 Quick Reference

**Common Commands:**
```bash
# Deploy manually
kubectl apply -f kubernetes/frontend/
kubectl apply -f kubernetes/backend/

# Check status
kubectl get all -o wide

# Restart deployment
kubectl rollout restart deployment/astro-deployment

# View logs
kubectl logs -f deployment/astro-deployment

# Shell into pod
kubectl exec -it <pod-name> -- /bin/sh
```

---

**Version:** 2.0
**Last Updated:** 2026-01-25
