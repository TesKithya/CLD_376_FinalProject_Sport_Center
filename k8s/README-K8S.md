# Sports Center on Kubernetes (Minikube) – step by step

## 0. Prerequisites
Docker Desktop (or Docker Engine), `minikube`, `kubectl`, Node 18+ (only for the smoke test).

## 1. Create YOUR MongoDB Atlas database (cloud DB)
1. https://cloud.mongodb.com → create a free **M0** cluster.
2. **Database Access** → Add user (username + password). Prefer a password without special characters.
3. **Network Access** → Add IP `0.0.0.0/0` (fine for a lab; minikube's outgoing IP changes).
4. **Connect → Drivers** → copy the host part, e.g. `cluster0.ab1cd2e.mongodb.net`.
Databases (`registration_db`, `authentication_db`, `facility_db`, `reservation_db`, `matching_db`, `maintenance_db`) are created automatically on first write.

## 2. Edit the secret (the only file you must edit)
`k8s/00-secret.yaml` → set `MONGO_USER`, `MONGO_PASSWORD`, `MONGO_HOST`.
`JWT_SECRET` and `INTERNAL_API_KEY` are already freshly generated.

## 3. Start Minikube
```bash
minikube start --cpus=2 --memory=4096
minikube addons enable ingress
kubectl get pods -n ingress-nginx        # wait until controller is Running
```

## 4. Build the 7 images inside Minikube
Run from the project root:
```bash
minikube image build -t sports/api-gateway:1.0          ./APIGateway_Microservice
minikube image build -t sports/registration-service:1.0 ./Registration_Microservice
minikube image build -t sports/auth-service:1.0         ./Authentication_Microservice
minikube image build -t sports/facility-service:1.0     ./Facility_Microservice
minikube image build -t sports/reservation-service:1.0  ./Reservation_Microservice
minikube image build -t sports/matching-service:1.0     ./Matching_Microservice
minikube image build -t sports/maintenance-service:1.0  ./Maintenance_Microservice
minikube image ls | grep sports
```

## 5. Deploy (YAML only)
```bash
kubectl apply -f k8s/00-namespace.yaml
kubectl apply -f k8s/
kubectl get pods -n sports-center -w      # every pod should reach 2/2 Running
```

## 6. Point the domain at the cluster (Fanout DNS)
- **Docker driver on Windows/macOS:** add `127.0.0.1 aupp.com` to the hosts file and keep `minikube tunnel` running in a separate terminal.
- **Linux / other drivers:** add `<output of minikube ip> aupp.com` to the hosts file.
Hosts file: Windows `C:\Windows\System32\drivers\etc\hosts` (open editor as Administrator); Linux/macOS `/etc/hosts` (sudo).
```bash
kubectl get ingress -n sports-center
```

## 7. Test
```bash
# a) everything up, via ingress -> gateway
curl http://aupp.com/health/services

# b) full end-to-end flow
GATEWAY_URL=http://aupp.com STAFF_SIGNUP_CODE=aupp-staff-2026 node scripts/smoke-test.js
#    (PowerShell:  $env:GATEWAY_URL="http://aupp.com"; $env:STAFF_SIGNUP_CODE="aupp-staff-2026"; node scripts/smoke-test.js)

# c) rule 3 – gateway only
curl -i http://aupp.com/internal/anything            # 404 (blocked by gateway)
kubectl run t --rm -it --restart=Never --image=curlimages/curl -n sports-center -- curl -i http://facility-service:3003/   # 403 Direct access forbidden

# d) rule 6 – shared volume read by two containers
kubectl exec -n sports-center deploy/api-gateway -c app        -- ls -l /var/log/app
kubectl exec -n sports-center deploy/api-gateway -c log-reader -- tail -n 5 /logs/api-gateway.log
kubectl logs -n sports-center deploy/api-gateway -c log-reader --tail=5

# e) gateway replicas (2) are load balanced
kubectl get pods -n sports-center -l app=api-gateway -o wide
for i in 1 2 3 4 5 6; do curl -s http://aupp.com/health >/dev/null; done
kubectl logs -n sports-center -l app=api-gateway -c app --prefix --tail=3

# f) rule 4 – open Atlas → Browse Collections: you should see the 6 databases with data
```

## Troubleshooting
| Symptom | Fix |
|---|---|
| Pod `ErrImageNeverPull` / `ImagePullBackOff` | Image not built in minikube. Re-run step 4 (check `minikube image ls`). |
| Pods run but never Ready, logs say `MongoDB connection attempt failed` | Wrong user/password/host in the secret, special characters not URL-encoded, or Atlas Network Access missing `0.0.0.0/0`. After fixing: `kubectl apply -f k8s/00-secret.yaml && kubectl rollout restart deploy -n sports-center` |
| `curl aupp.com` can't resolve / hangs | Hosts entry missing, or (Docker driver) `minikube tunnel` not running. |
| Ingress has no ADDRESS | Ingress addon not ready: `kubectl get pods -n ingress-nginx`. |
| `429 Too many attempts` on auth | Auth rate limit (50 / 15 min / gateway pod). Wait, or `kubectl rollout restart deploy/api-gateway -n sports-center`. |
| Change code | Rebuild with a new tag (e.g. `:1.1`), update `image:` in the YAML, `kubectl apply`. |

Clean up: `kubectl delete namespace sports-center`
