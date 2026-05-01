# Secure Event Ticketing Platform

Višeslojna web-aplikacija za prodaju ulaznica za događaje, izgrađena kroz cijeli
DevSecOps ciklus: lokalni razvoj s Podman Compose, CI/CD pipeline s GitHub Actions
i produkcijska isporuka na Kubernetes.

## Arhitektura

| Servis     | Tehnologija        | Port  | Uloga                              |
|------------|--------------------|-------|------------------------------------|
| frontend   | Node.js / Express  | 3000  | Korisničko sučelje                 |
| api        | Node.js / Express  | 8080  | Poslovna logika i REST API         |
| worker     | Node.js            | —     | Asinkrona obrada narudžbi          |
| postgres   | PostgreSQL 16      | 5432  | Trajna pohrana podataka            |
| redis      | Redis 7            | 6379  | Red zadataka i predmemorija        |

## Preduvjeti

- Podman + podman-compose (ili Docker + Docker Compose)
- Node.js 20+ (samo za lokalni razvoj bez kontejnera)

## Pokretanje lokalnog okruženja

### 1. Priprema environment varijabli

```bash
cp .env.example .env
```

Otvoriti `.env` i po potrebi promijeniti vrijednosti (lokalne lozinke).

### 2. Pokretanje cijelog stacka

```bash
podman compose up --build
```

Svi servisi pokreću se jednom naredbom. Redoslijed pokretanja je automatski
(postgres i redis moraju biti zdravi prije api i worker servisa).

### 3. Provjera zdravlja

```bash
# Frontend
curl http://localhost:3000/healthz

# API (liveness)
curl http://localhost:8080/healthz

# API (readiness — provjerava postgres i redis)
curl http://localhost:8080/readyz
```

Očekivani odgovor: `{"status":"ok"}`

### 4. Osnovni radni tok

```bash
# Dohvat popisa događaja
curl http://localhost:8080/events

# Kupnja ulaznice
curl -X POST http://localhost:8080/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId": 1, "email": "test@example.com"}'

# Provjera narudžbi
curl http://localhost:8080/tickets/orders
```

### 5. Zaustavljanje stacka

```bash
# Zaustavljanje (zadržava podatke u volumenu)
podman compose down

# Zaustavljanje i brisanje podataka
podman compose down -v
```

### 6. Hot-reload (razvoj)

Za razvoj s automatskim ponovnim učitavanjem koda:

```bash
podman compose watch
```

Svaka promjena u `frontend/src/`, `api/src/` ili `worker/src/` automatski se
sinkronizira u kontejner bez ponovnog izgradnje slike.

## Environment varijable

Sve varijable se konfiguriraju u `.env` datoteci (nikad se ne commitaju u repozitorij).

| Varijabla           | Opis                            | Primjer             |
|---------------------|---------------------------------|---------------------|
| `POSTGRES_DB`       | Naziv baze podataka             | `ticketing`         |
| `POSTGRES_USER`     | Korisničko ime baze             | `ticketing_user`    |
| `POSTGRES_PASSWORD` | Lozinka baze podataka           | `promijeni_lokalno` |
| `POSTGRES_HOST`     | Naziv hosta baze                | `postgres`          |
| `POSTGRES_PORT`     | Port baze podataka              | `5432`              |
| `REDIS_HOST`        | Naziv hosta Redis servisa       | `redis`             |
| `REDIS_PORT`        | Port Redis servisa              | `6379`              |
| `API_PORT`          | Port API servisa                | `8080`              |
| `FRONTEND_PORT`     | Port frontend servisa           | `3000`              |
| `QUEUE_NAME`        | Naziv Redis reda zadataka       | `ticket_orders`     |
| `NODE_ENV`          | Okruženje (development/production) | `development`    |

## CI/CD pipeline

Svaki `git push` na granu `main` automatski pokreće GitHub Actions pipeline:

1. Provjera koda — instalacija ovisnosti (`npm ci`) za sva 3 servisa
2. Izgradnja kontejnerskih slika (multi-stage build)
3. Trivy skeniranje ranjivosti — blokira HIGH i CRITICAL nalaze
4. Objava slika u GitHub Container Registry (`ghcr.io`)

SemVer tagiranje:
```bash
git tag v1.0.0
git push origin v1.0.0
```

## Produkcijski deployment (Kubernetes)

### Preduvjeti

- Kubernetes klaster (minikube, kind, OpenShift, GKE, AKS…)
- `kubectl` konfiguriran za ciljni klaster
- Nginx Ingress Controller instaliran u klasteru

### 1. Priprema tajni (Secrets)

Prije primjene manifesta zamijeni placeholder lozinku u `k8s/02-secret.yaml`
ili stvori Secret direktno (preporučeno za produkciju):

```bash
kubectl create namespace ticketing

kubectl create secret generic app-secrets \
  --namespace ticketing \
  --from-literal=POSTGRES_USER=ticketing_user \
  --from-literal=POSTGRES_PASSWORD=<jaka-lozinka>
```

Ako koristiš GHCR privatni registar, stvori i pull secret:

```bash
kubectl create secret docker-registry ghcr-pull-secret \
  --namespace ticketing \
  --docker-server=ghcr.io \
  --docker-username=<github-korisnik> \
  --docker-password=<github-token>
```

### 2. Primjena svih manifesta

```bash
kubectl apply -f k8s/
```

Manifesti su numerirani kako bi osigurali ispravan redoslijed primjene:
`00-namespace` → `01-configmap` → `02-secret` → `03-serviceaccount` →
`04-rbac` → `05-postgres` → `06-redis` → `07-api` → `08-worker` →
`09-frontend` → `10-ingress` → `11-networkpolicy` → `12-pdb`

### 3. Provjera isporuke

```bash
# Prati status podova
kubectl get pods -n ticketing -w

# Provjeri servise
kubectl get svc -n ticketing

# Provjeri Ingress
kubectl get ingress -n ticketing
```

Svi podovi trebaju biti u statusu `Running` s oznakom `1/1` ili `2/2`.

### 4. Ažuriranje na novu verziju (Rolling Update)

```bash
# Ažuriranje API servisa na novu verziju slike
kubectl set image deployment/api \
  api=ghcr.io/matej-basic/devops-algebra/api:v1.1.0 \
  -n ticketing

# Praćenje napretka ažuriranja
kubectl rollout status deployment/api -n ticketing

# Isti postupak vrijedi za frontend i worker
kubectl set image deployment/frontend \
  frontend=ghcr.io/matej-basic/devops-algebra/frontend:v1.1.0 \
  -n ticketing
kubectl set image deployment/worker \
  worker=ghcr.io/matej-basic/devops-algebra/worker:v1.1.0 \
  -n ticketing
```

Strategija `RollingUpdate` s `maxUnavailable: 0` osigurava nultu
nedostupnost — nova replika mora biti zdrava (`/healthz`) prije nego
se stara ugasi. `PodDisruptionBudget` (manifest `12-pdb.yaml`) dodatno
štiti od istovremenog prekida više replika tijekom održavanja čvora.

### 5. Povrat na prethodnu verziju (Rollback)

```bash
# Pregled povijesti isporuka
kubectl rollout history deployment/api -n ticketing

# Povrat na prethodnu verziju
kubectl rollout undo deployment/api -n ticketing

# Povrat na specifičnu reviziju
kubectl rollout undo deployment/api --to-revision=2 -n ticketing

# Provjera da su podovi zdravi nakon povrata
kubectl get pods -n ticketing -l app=api
```

### 6. Uklanjanje iz klastera

```bash
# Ukloni sve resurse iz namespacea
kubectl delete namespace ticketing
```

## Dokumentacija

| Datoteka                      | Sadržaj                                      |
|-------------------------------|----------------------------------------------|
| `docs/architecture.md`        | Arhitekturni dijagram i usporedba s VM       |
| `docs/image-policy.md`        | Politika upravljanja kontejnerskim slikama   |
| `docs/devsecops.md`           | DevSecOps metodologija i sigurnosne provjere |
| `docs/delivery-metrics.md`    | Mjerenje brzine isporuke — baseline vs CI/CD |
| `RUNBOOK.md`                  | Upute za rješavanje incidenata               |
| `k8s/`                        | Kubernetes manifesti za produkcijsku isporuku|
