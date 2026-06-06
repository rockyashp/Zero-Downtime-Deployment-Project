# ✅ CI/CD Setup Checklist

Complete each section in order. Expected time: ~40 minutes.

---

## 📋 PHASE 1: GitHub Secrets (10 min)

### Docker Hub Credentials

- [ ] Go to: https://hub.docker.com/settings/security/tokens/new
- [ ] Click "New Access Token"
- [ ] Name: "GitHub Actions" (or your preference)
- [ ] Set permissions: read/write
- [ ] Create and copy token

**Add to GitHub:**
- [ ] Go to: GitHub → Settings → Secrets and variables → Actions
- [ ] Click "New repository secret"
- [ ] Name: `DOCKER_USERNAME` | Value: `himanshux99` (or your username)
- [ ] Click "New repository secret"
- [ ] Name: `DOCKER_PASSWORD` | Value: [Paste token]

### Kubernetes Config

**On your machine with kubectl access:**

```bash
# Copy kubeconfig as base64
cat ~/.kube/config | base64 -w 0
# For macOS:
cat ~/.kube/config | base64 | tr -d '\n'
```

- [ ] Copy the entire base64 output (very long string)

**Add to GitHub:**
- [ ] Go to: GitHub → Settings → Secrets and variables → Actions
- [ ] Click "New repository secret"
- [ ] Name: `KUBE_CONFIG` | Value: [Paste entire base64 string]
- [ ] Click "New repository secret"
- [ ] Name: `KUBE_NAMESPACE` | Value: `default` (or your namespace)

### Verify Secrets
- [ ] Settings → Secrets and variables → Actions
- [ ] All 4 secrets should be listed:
  - `DOCKER_USERNAME`
  - `DOCKER_PASSWORD`
  - `KUBE_CONFIG`
  - `KUBE_NAMESPACE`

---

## 🛠️ PHASE 2: Kubernetes Setup (10 min)

### Verify Cluster Access

```bash
# Test kubectl works with your config
kubectl cluster-info
kubectl get nodes
```

- [ ] Both commands work without errors

### Check Existing Deployments

```bash
kubectl get deployments
```

- [ ] See output like:
  ```
  NAME                      READY   UP-TO-DATE
  astro-deployment          2/2     2
  backend-v2-deployment     2/2     2
  ```

If deployments don't exist, create them:

```bash
kubectl apply -f kubernetes/frontend/deployment-v1-enhanced.yaml
kubectl apply -f kubernetes/backend/backend-v2-enhanced.yaml
```

- [ ] Both deployments created successfully

### Verify Services

```bash
kubectl get svc
```

- [ ] Should see:
  ```
  NAME                    TYPE        CLUSTER-IP
  astro-service          ClusterIP   10.96.x.x
  backend-v2-service     ClusterIP   10.96.x.x
  ```

### Check Ingress

```bash
kubectl get ingress
```

- [ ] Should show app-ingress pointing to `app.local`

### Test Health Endpoints

```bash
# Forward frontend
kubectl port-forward svc/astro-service 8080:80 &
curl http://localhost:8080/
# Expected: HTML page

# Forward backend
kubectl port-forward svc/backend-v2-service 3000:80 &
curl http://localhost:3000/api/v1/healthCheck
# Expected: JSON with "status": "healthy"
```

- [ ] Frontend responds with HTTP 200
- [ ] Backend responds with health JSON (may get 404 if endpoint doesn't exist yet - that's OK)

---

## 🚀 PHASE 3: Application Updates (5 min)

### Backend Health Check Endpoint

**Edit:** `server/server.js` or `server/app.js`

Add this route:

```javascript
app.get('/api/v1/healthCheck', (req, res) => {
  res.json({
    status: 'healthy',
    timestamp: new Date(),
    version: process.env.VERSION || 'v1',
    uptime: process.uptime()
  });
});
```

- [ ] Health check endpoint added
- [ ] Test locally: `npm run dev` then `curl http://localhost:3000/api/v1/healthCheck`
- [ ] Commit and push: `git add . && git commit -m "Add health check endpoint"`

### Optional: Graceful Shutdown

**In `server/server.js`:**

```javascript
const server = http.createServer(app);

process.on('SIGTERM', () => {
  console.log('Shutting down gracefully...');
  server.close(() => process.exit(0));
  setTimeout(() => process.exit(1), 10000); // Force exit after 10s
});

server.listen(PORT, () => console.log(`Server on ${PORT}`));
```

- [ ] Graceful shutdown implemented (optional)

---

## 🧪 PHASE 4: Test Pipeline (15 min)

### Create Test Branch

```bash
git checkout -b test/cicd
git push origin test/cicd
```

- [ ] Test branch created and pushed

### Monitor Pipeline

- [ ] Go to: GitHub → Actions tab
- [ ] Should see: "CI/CD Pipeline" running
- [ ] Wait for jobs to complete:
  - [ ] lint-frontend ✅
  - [ ] lint-backend ✅
  - [ ] (build jobs skipped - only on main) ✅

### Fix Any Lint Issues

If lint jobs fail:

```bash
# Check what failed
cd Frontend && npm run build
cd ../server && npm ci
```

- [ ] Fix any errors
- [ ] Commit: `git add . && git commit -m "Fix lint issues"`
- [ ] Push: `git push origin test/cicd`
- [ ] Rerun pipeline (or close and reopen PR)

### Merge to Main (When Ready)

```bash
git checkout main
git merge test/cicd
git push origin main
```

- [ ] Test branch merged to main
- [ ] Full pipeline starts (Actions tab)

### Watch Full Deployment

- [ ] Pipeline shows all jobs running:
  - [ ] lint-frontend ✅
  - [ ] lint-backend ✅
  - [ ] build-frontend ⏳
  - [ ] build-backend ⏳
  - [ ] deploy ⏳
  - [ ] notify ✅

- [ ] All jobs complete successfully
- [ ] Notification shows ✅ Success

---

## 📊 PHASE 5: Verify Deployment (5 min)

### Check Kubernetes Pods

```bash
kubectl get pods -o wide
```

- [ ] At least 2 astro pods in Running/Ready state
- [ ] At least 2 backend pods in Running/Ready state
- [ ] IMAGE column shows: `himanshux99/astro-frontend:main-xxxxx`
- [ ] IMAGE column shows: `himanshux99/astro-backend:main-xxxxx`

### Verify Rollout

```bash
kubectl rollout status deployment/astro-deployment
kubectl rollout status deployment/backend-v2-deployment
```

- [ ] Both show: "deployment 'xxx' successfully rolled out"

### Check Pod Details

```bash
kubectl describe pod <pod-name>
```

- [ ] Status: Running
- [ ] Ready: True
- [ ] Recent events: All look healthy

### Access Application

Test your deployed application:

```bash
# If using port-forward
kubectl port-forward svc/astro-service 8080:80
# Visit: http://localhost:8080

# Or if using ingress
# Visit configured domain: http://app.local
```

- [ ] Frontend loads successfully
- [ ] No errors in browser console
- [ ] All pages working

### Test Backend

```bash
kubectl port-forward svc/backend-v2-service 3000:80
curl http://localhost:3000/api/v1/healthCheck
```

- [ ] Returns JSON with healthy status
- [ ] HTTP 200 response

---

## 🎯 PHASE 6: Optional - Enable Notifications

### Slack Notifications (5 min)

- [ ] Go to: https://api.slack.com/apps
- [ ] Create New App
- [ ] Enable Incoming Webhooks
- [ ] Add webhook for your channel
- [ ] Copy webhook URL
- [ ] Go to GitHub Secrets
- [ ] Add: `SLACK_WEBHOOK` = [paste URL]

See `ADVANCED_CICD_FEATURES.md` for implementation.

---

## 🚀 PHASE 7: Ready for Production!

### Before Production Deployments

- [ ] All phases 1-5 complete
- [ ] First test deployment successful
- [ ] Team aware of new CI/CD process
- [ ] Rollback procedure understood

### Go Live

- [ ] Create code change in feature branch
- [ ] Push to GitHub
- [ ] Create Pull Request (lint jobs run)
- [ ] Review and merge to main
- [ ] Watch pipeline deploy automatically
- [ ] Verify in production
- [ ] Done! 🎉

---

## 🐛 Troubleshooting Checklist

### Pipeline Doesn't Start
- [ ] GitHub Actions enabled: Settings → Actions
- [ ] All 4 secrets exist
- [ ] File exists: `.github/workflows/ci-cd.yaml`
- [ ] Push to main or create PR

### Build Fails
- [ ] Check Actions log
- [ ] Run locally: `npm install && npm run build`
- [ ] Fix errors and push again

### Deployment Doesn't Update
- [ ] Check Kubernetes cluster access
- [ ] Verify KUBE_CONFIG secret is correct
- [ ] Check: `kubectl get nodes`
- [ ] Pods should update within 2 minutes

### Health Checks Fail
- [ ] Backend endpoint must exist: `GET /api/v1/healthCheck`
- [ ] Frontend must respond to: `GET /`
- [ ] Test endpoints manually with `kubectl exec`
- [ ] Check pod logs: `kubectl logs <pod-name>`

### Still Stuck?
- [ ] Read: `DEPLOYMENT_GUIDE.md` Troubleshooting
- [ ] Check pod events: `kubectl describe pod <name>`
- [ ] View logs: `kubectl logs -f deployment/xxx`

---

## 📞 Support Resources

- **Pipeline Guide:** `.github/CICD_GUIDE.md`
- **Deployment Guide:** `DEPLOYMENT_GUIDE.md`
- **Advanced Features:** `ADVANCED_CICD_FEATURES.md`
- **Quick Reference:** `CICD_IMPLEMENTATION_SUMMARY.md`
- **This Checklist:** `SETUP_CHECKLIST.md`

---

## ✅ Sign Off

When complete, mark all phases done:

- [x] Phase 1: GitHub Secrets ✅
- [x] Phase 2: Kubernetes Setup ✅
- [x] Phase 3: Application Updates ✅
- [x] Phase 4: Test Pipeline ✅
- [x] Phase 5: Verify Deployment ✅
- [x] Phase 6: Notifications (optional) ✅
- [x] Phase 7: Production Ready ✅

**You're ready to deploy!** 🚀

---

**Time Estimate:** 40 minutes for Phases 1-5
**Total with Notifications:** 50 minutes

**Next Deploy:** Just `git push` to main! 🎉

---

**Version:** 1.0
**Last Updated:** 2026-01-25
