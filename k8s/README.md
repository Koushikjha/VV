# Echo — local Kubernetes (kind)

Built from what's actually in `application.yaml`, `SecurityConfig.java`,
`RedisConfig.java`, and the frontend source — not a generic template. Two
things in your own code shaped this more than anything else; read the
"Known constraints" section before you demo this.

## 1. Create the kind cluster

Needs three host ports mapped: 80/443 for Ingress, and 8080 for the backend
(see "Known constraints" for why 8080 is separate from Ingress).

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
    extraPortMappings:
      - containerPort: 80
        hostPort: 80
        protocol: TCP
      - containerPort: 443
        hostPort: 443
        protocol: TCP
      - containerPort: 30080   # matches nodePort in backend/service.yaml
        hostPort: 8080
        protocol: TCP
```

```bash
kind create cluster --name echo --config kind-config.yaml
```

(Reusing the same cluster as Seller AI's? Merge the `extraPortMappings`
lists into one `kind-config.yaml` and create one cluster instead of two.)

## 2. Install ingress-nginx

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/kind/deploy.yaml
kubectl wait --namespace ingress-nginx \
  --for=condition=ready pod \
  --selector=app.kubernetes.io/component=controller \
  --timeout=180s
```

## 3. Apply everything

No image build step — `koushikjha/echo-backend:latest` and
`koushikjha/echo-frontend:latest` are already on Docker Hub from your
existing GitHub Actions CI/CD, and kind's nodes can pull public images
directly. (Only rebuild + `kind load docker-image` if you have local
changes not yet pushed.)

```bash
kubectl apply -k k8s/
kubectl get pods -n echo -w
```

The two `wait-for-*` init containers on the backend pod block it until
MySQL and Redis both pass their own readiness checks — same ordering
`depends_on` would give you if your compose file had those services in it.

## 4. Point echo.local at your cluster

```bash
echo "127.0.0.1 echo.local" | sudo tee -a /etc/hosts
```

## 5. Test it

```bash
curl http://localhost:8080/actuator/health   # backend, direct
# {"status":"UP"}

curl -I http://echo.local/                   # frontend, via Ingress
# HTTP/1.1 200 OK
```

## Known constraints (found by reading your code, not introduced by this)

**1. Backend is NodePort, not Ingress-routed, on purpose.**
`frontend/src/api.js` and `useWebSocket.js` both hardcode
`http://localhost:8080` / `ws://localhost:8080/ws/websocket` instead of
reading a build-time env var. That works in your docker-compose setup
because compose happens to publish backend on host port 8080 too. To keep
these manifests a zero-code-change drop-in, the backend Service is NodePort
30080, mapped to host port 8080 by kind — so the hardcoded URL keeps
working unmodified. The real fix, whenever you want it, is a Vite env var
(`import.meta.env.VITE_API_URL`) baked in at build time instead of a
literal string, which would let the backend live cleanly behind Ingress
like the frontend does.

**2. CORS will likely block authenticated requests from this frontend.**
`SecurityConfig.java`'s `corsConfigurationSource()` only allows origin
`http://localhost:5173` (Vite's dev-server port). Neither `http://echo.local`
(Ingress) nor `http://localhost:3000` (your own docker-compose frontend
port, for that matter) match it. `OPTIONS` and `/login` are exempted from
auth, but any authenticated call the built frontend makes from a different
origin will get rejected by the browser's CORS check — this is a
pre-existing gap in `SecurityConfig.java`, not something these manifests
caused. Add your real origins (`http://echo.local`, plus whatever you
front this with later) to `configuration.setAllowedOrigins(...)` when
you're ready to fix it.

**3. `app.security.public-urls` has no default and nothing sets it.**
`PublicUrlProperties.getPublicUrls()` returns `null` unless
`app.security.public-urls` is configured somewhere, and `SecurityConfig`
immediately calls `.toArray()` on it — so with nothing supplying that list,
the app throws an NPE building the `SecurityFilterChain` and never starts.
Locally this is presumably set in your gitignored `application-secrets.yaml`
(not in the zip you gave me — correctly excluded from git). These manifests
supply the one entry actually needed here,
`APP_SECURITY_PUBLICURLS_0=/actuator/health` in `backend/configmap.yaml`,
so the k8s readiness/liveness probes can reach it without a JWT. If your
WebSocket handshake or other endpoints need to be public too, add more
`APP_SECURITY_PUBLICURLS_N` entries the same way.

**4. Don't set `SPRING_PROFILES_ACTIVE=prod` against this stack.**
`application-prod.yaml` has a real-looking MSG91 SMS API key committed in
plain text in the repo. `backend/configmap.yaml` here pins the `dev`
profile specifically to avoid ever loading it. Worth rotating that key and
moving it out of version control independent of anything here.

## What's still manual (matches the resume line, not hidden)

- No CI/CD wiring for this k8s deploy — you apply it by hand.
- Single replica everywhere, MySQL on `Recreate` strategy (one RWO volume).
- Secrets are plain `Opaque` with dev creds sitting in these files. Fine for
  kind, not a pattern for anywhere real.
