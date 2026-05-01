# RUNBOOK — Secure Event Ticketing Platform

Ovaj dokument opisuje postupke dijagnostike i ispravka incidenata u produkcijskom
Kubernetes okruženju. Svaki incident slijedi strukturu:
**Simptom → Dijagnoza → Uzrok → Ispravak → Validacija → Prevencija**

---

## Incident 1 — Pad PostgreSQL baze

### 1.1 Simptom

- API servis vraća HTTP 503 ili grešku `could not connect to server`
- Worker prestaje obrađivati narudžbe — nema novih zapisa u bazi
- `kubectl get pods` prikazuje postgres pod u stanju `OOMKilled`, `CrashLoopBackOff`
  ili `Pending`

```
NAME                        READY   STATUS             RESTARTS   AGE
postgres-7d9f8b6c4-xk2pq   0/1     OOMKilled          3          12m
api-6b8d9f7c5-m3nwp        1/1     Running            0          45m
```

### 1.2 Dijagnoza

```bash
# Provjera statusa svih podova
kubectl get pods -n ticketing

# Detalji o padu (razlog, poruke o grešci)
kubectl describe pod <postgres-pod-naziv> -n ticketing

# Pregled zadnjih 100 redaka dnevnika
kubectl logs <postgres-pod-naziv> -n ticketing --tail=100

# Provjera zauzeća memorije i CPU-a
kubectl top pod <postgres-pod-naziv> -n ticketing

# Provjera volumena (PersistentVolumeClaim)
kubectl get pvc -n ticketing
kubectl describe pvc postgres-pvc -n ticketing

# Provjera događaja u namespaceu
kubectl get events -n ticketing --sort-by='.lastTimestamp'
```

Ključni pokazatelji u izlazu naredbi:
- `OOMKilled` → pod potrošio više memorije od definiranog limita
- `Pending` → PVC nije dostupan ili nema dovoljno resursa na čvoru
- `database file appears to be corrupted` u dnevniku → oštećeni podaci

### 1.3 Uzrok

**OOMKilled**: PostgreSQL je prekoračio memorijski limit definiran u Kubernetes
manifestu. Do toga dolazi kada broj aktivnih veza ili veličina podataka raste
iznad predviđenog, a limit resursa nije prilagođen stvarnom opterećenju.

**PVC nedostupan**: Trajni volumen (PersistentVolumeClaim) nije montiran jer
StorageClass nije dostupan na čvoru, ili je volumen u stanju `Terminating`
zbog prekinutog brisanja prethodnog poda.

**Oštećeni podaci**: PostgreSQL nije pravilno zatvoren (npr. zbog nasilnog
prekida poda bez `graceful shutdown`), što može uzrokovati nepotpune transakcije
i oštećenje WAL (Write-Ahead Log) datoteka.

### 1.4 Ispravak

**Slučaj A — OOMKilled (povećanje memorijskog limita):**

```bash
# Uređivanje memorijskog limita u manifestu
kubectl edit deployment postgres -n ticketing
# Promijeniti: resources.limits.memory: "512Mi" → "1Gi"

# Ili primjena ažuriranog manifesta
kubectl apply -f k8s/05-postgres.yaml
```

**Slučaj B — PVC nije dostupan:**

```bash
# Provjera stanja volumena
kubectl get pv -n ticketing

# Ako je volumen u stanju Released, ručno ga osloboditi
kubectl patch pv <pv-naziv> -p '{"spec":{"claimRef": null}}'

# Ponovni pokušaj vezivanja
kubectl delete pvc postgres-pvc -n ticketing
kubectl apply -f k8s/05-postgres.yaml
```

**Slučaj C — Oštećeni podaci (oporavak iz sigurnosne kopije):**

```bash
# Zaustavljanje API i worker servisa (sprječavanje novih zapisa)
kubectl scale deployment api --replicas=0 -n ticketing
kubectl scale deployment worker --replicas=0 -n ticketing

# Kopiranje sigurnosne kopije u pod
kubectl cp backup.sql <postgres-pod-naziv>:/tmp/backup.sql -n ticketing

# Oporavak podataka
kubectl exec -it <postgres-pod-naziv> -n ticketing -- \
  psql -U ticketing_user -d ticketing -f /tmp/backup.sql

# Pokretanje servisa
kubectl scale deployment api --replicas=1 -n ticketing
kubectl scale deployment worker --replicas=1 -n ticketing
```

### 1.5 Validacija

```bash
# Provjera da postgres pod radi
kubectl get pod -l app=postgres -n ticketing
# Očekivani izlaz: STATUS = Running, READY = 1/1

# Provjera readiness sonde API servisa (potvrđuje vezu s bazom)
kubectl exec -it <api-pod-naziv> -n ticketing -- \
  wget -qO- http://localhost:8080/readyz
# Očekivani izlaz: {"status":"ok","postgres":"ok","redis":"ok"}

# Provjera dnevnika postgres poda — ne smije biti grešaka
kubectl logs -l app=postgres -n ticketing --tail=20
# Očekivani izlaz: "database system is ready to accept connections"

# Test kupnje ulaznice (provjera pisanja u bazu)
curl -X POST http://<ingress-adresa>/api/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":1,"email":"test@example.com"}'
# Očekivani izlaz: HTTP 202 s orderId
```

### 1.6 Prevencija

- Memorijski zahtjevi i limiti moraju biti postavljeni na temelju stvarnog
  opterećenja — koristiti `kubectl top` za praćenje i prilagoditi manifest
- Konfigurirati automatske sigurnosne kopije (CronJob koji svakodnevno izvršava
  `pg_dump` i sprema na vanjsku pohranu)
- Postaviti `terminationGracePeriodSeconds: 60` u manifestu da PostgreSQL
  ima dovoljno vremena za pravilno zatvaranje pri promjeni poda

---

## Incident 2 — ImagePullBackOff

### 2.1 Simptom

- Novi pod ostaje u stanju `ImagePullBackOff` ili `ErrImagePull` i ne pokreće se
- `kubectl get pods` prikazuje:

```
NAME                      READY   STATUS             RESTARTS   AGE
api-7f9d8b6c4-xk2pq      0/1     ImagePullBackOff   0          3m
```

- Isporuka nove verzije aplikacije blokirana — prethodna verzija i dalje radi

### 2.2 Dijagnoza

```bash
# Pregled statusa poda
kubectl get pods -n ticketing

# Detalji o grešci povlačenja slike
kubectl describe pod <pod-naziv> -n ticketing
# Tražiti sekciju Events — pisat će točna poruka greške

# Najčešće poruke:
# "manifest unknown" → tag ne postoji u registru
# "unauthorized" → istekla lozinka ili krivi pull secret
# "not found" → slika ne postoji na navedenoj putanji

# Provjera koji image pull secret se koristi
kubectl get secret ghcr-pull-secret -n ticketing -o yaml

# Provjera dostupnih tagova (lokalno)
podman pull ghcr.io/<owner>/devops-algebra/api:v1.0.0
```

### 2.3 Uzrok

**Krivi tag**: U manifestu je naveden tag koji ne postoji u registru (npr.
`v1.0.1` koji još nije objavljen, ili typo u nazivu). Do toga dolazi pri ručnom
uređivanju manifesta bez provjere dostupnih tagova.

**Istekli pull secret**: `ghcr-pull-secret` Kubernetes Secret sadrži GitHub
Personal Access Token (PAT) s ograničenim trajanjem. Nakon isteka tokena
Kubernetes ne može autenticirati se prema `ghcr.io` registru.

### 2.4 Ispravak

**Slučaj A — Krivi tag:**

```bash
# Provjera koji tagovi postoje u registru
# Na GitHub: repository → Packages → api → verzije

# Ispravak taga u manifestu
kubectl set image deployment/api \
  api=ghcr.io/<owner>/devops-algebra/api:v1.0.0 -n ticketing

# Ili primjena ispravnog manifesta
kubectl apply -f k8s/07-api.yaml
```

**Slučaj B — Istekli pull secret:**

```bash
# Brisanje starog secretа
kubectl delete secret ghcr-pull-secret -n ticketing

# Kreiranje novog secretа s novim tokenom
kubectl create secret docker-registry ghcr-pull-secret \
  --docker-server=ghcr.io \
  --docker-username=<github-korisnicko-ime> \
  --docker-password=<novi-github-pat> \
  -n ticketing

# Pokretanje novog rollout-a da pod preuzme novi secret
kubectl rollout restart deployment/api -n ticketing
```

### 2.5 Validacija

```bash
# Provjera da pod uspješno povlači sliku i pokreće se
kubectl get pods -l app=api -n ticketing
# Očekivani izlaz: STATUS = Running, READY = 1/1

# Provjera koji image koristi pokrenuti pod
kubectl describe pod <api-pod-naziv> -n ticketing | grep Image:
# Očekivani izlaz: Image: ghcr.io/<owner>/devops-algebra/api:v1.0.0

# Provjera zdravlja servisa
kubectl exec -it <api-pod-naziv> -n ticketing -- \
  wget -qO- http://localhost:8080/healthz
# Očekivani izlaz: {"status":"ok"}

# Provjera dnevnika — ne smije biti grešaka pri pokretanju
kubectl logs -l app=api -n ticketing --tail=20
# Očekivani izlaz: "API servis pokrenut na portu 8080"
```

### 2.6 Prevencija

- U CI/CD pipelineu koristiti `docker/metadata-action` za automatsko određivanje
  tagova — eliminira mogućnost ručne greške pri pisanju taga
- GitHub PAT za pull secret generirati s rokom trajanja od 90 dana i konfigurirati
  upozorenje u kalendaru 1 tjedan prije isteka za rotaciju
- Kubernetes manifest uvijek referencirati konkretnu SemVer verziju
  (npr. `v1.0.0`), nikad `latest` u produkciji

---

## Incident 3 — Krivi Secret / rotacija lozinke

### 3.1 Simptom

- API servis vraća HTTP 500 za sve zahtjeve koji zahtijevaju pristup bazi
- Worker prestaje obrađivati narudžbe
- `kubectl logs` prikazuje grešku autentikacije prema PostgreSQL:

```
Error: password authentication failed for user "ticketing_user"
```

- Incident se tipično javlja nakon rotacije lozinke baze u produkciji

### 3.2 Dijagnoza

```bash
# Provjera dnevnika API servisa
kubectl logs -l app=api -n ticketing --tail=50
# Tražiti: "password authentication failed" ili "ECONNREFUSED"

# Provjera dnevnika worker servisa
kubectl logs -l app=worker -n ticketing --tail=50

# Provjera sadržaja Kubernetes Secreta (base64 kodirano)
kubectl get secret app-secrets -n ticketing -o yaml

# Dekodiranje lozinke iz Secreta
kubectl get secret app-secrets -n ticketing \
  -o jsonpath='{.data.POSTGRES_PASSWORD}' | base64 --decode
# Usporediti s trenutnom lozinkom u PostgreSQL

# Test veze s bazom s lozinkom iz Secreta
kubectl exec -it <postgres-pod-naziv> -n ticketing -- \
  psql -U ticketing_user -d ticketing -c "SELECT 1;"
```

### 3.3 Uzrok

Kubernetes Secret `app-secrets` sadrži staru lozinku koja više ne
odgovara lozinci u PostgreSQL bazi podataka. Do neusklađenosti dolazi kada
se lozinka promijeni izravno u bazi (npr. `ALTER USER ticketing_user PASSWORD`)
bez istovremenog ažuriranja Kubernetes Secreta, ili obrnuto.

Kubernetes Secrets se ne sinkroniziraju automatski s bazom podataka — svaka
promjena lozinke mora se provesti na **oba mjesta** istovremeno.

### 3.4 Ispravak

```bash
# Korak 1 — Promjena lozinke u PostgreSQL bazi
kubectl exec -it <postgres-pod-naziv> -n ticketing -- \
  psql -U postgres -c "ALTER USER ticketing_user PASSWORD 'nova_lozinka';"

# Korak 2 — Ažuriranje Kubernetes Secreta s novom lozinkom
kubectl create secret generic app-secrets \
  --from-literal=POSTGRES_PASSWORD=nova_lozinka \
  --from-literal=REDIS_PASSWORD=redis_lozinka \
  --dry-run=client -o yaml | kubectl apply -f -

# Korak 3 — Pokretanje novog rollout-a da podovi preuzmu novi Secret
kubectl rollout restart deployment/api -n ticketing
kubectl rollout restart deployment/worker -n ticketing

# Korak 4 — Praćenje statusa rollout-a
kubectl rollout status deployment/api -n ticketing
kubectl rollout status deployment/worker -n ticketing
```

### 3.5 Validacija

```bash
# Provjera da podovi rade s novim Secretom
kubectl get pods -l app=api -n ticketing
kubectl get pods -l app=worker -n ticketing
# Očekivani izlaz: STATUS = Running, READY = 1/1 za oba

# Provjera readiness sonde (potvrđuje vezu s bazom)
kubectl exec -it <api-pod-naziv> -n ticketing -- \
  wget -qO- http://localhost:8080/readyz
# Očekivani izlaz: {"status":"ok","postgres":"ok","redis":"ok"}

# Provjera dnevnika — ne smije biti grešaka autentikacije
kubectl logs -l app=api -n ticketing --tail=20
# Očekivani izlaz: uspješno pokretanje bez poruka o grešci

# Test end-to-end toka (kupnja ulaznice + provjera zapisa u bazi)
curl -X POST http://<ingress-adresa>/api/tickets/purchase \
  -H "Content-Type: application/json" \
  -d '{"eventId":1,"email":"test@example.com"}'
# Očekivani izlaz: HTTP 202 s orderId
```

### 3.6 Prevencija

- Nikad ne mijenjati lozinku samo u bazi ili samo u Secretu — uvijek ažurirati
  oba mjesta unutar iste promjene (dokumentirati kao standardni postupak)
- Koristiti alate za upravljanje tajnama (npr. HashiCorp Vault ili Sealed Secrets)
  koji automatski sinkroniziraju Kubernetes Secrets s izvorom istine
- Rotaciju lozinki provoditi izvan radnog vremena uz prethodnu najavu radi
  minimiziranja utjecaja na korisnike

---

## Incident 4 — CI quality gate blokira isporuku zbog HIGH ranjivosti

### 4.1 Simptom

- GitHub Actions pipeline završava s greškom na koraku "Trivy skeniranje ranjivosti"
- Kontejnerska slika **nije** objavljena u registar
- Izlazni kod pipelina je `1` — isporuka je blokirana
- U Actions dnevniku vidljiva je poruka:

```
CRITICAL  1
HIGH      3
2025-xx-xx [FATAL] exit status 1
```

### 4.2 Dijagnoza

```bash
# Lokalna reprodukcija skeniranja (ista naredba kao u CI)
trivy image --severity HIGH,CRITICAL --ignore-unfixed \
  ghcr.io/<owner>/devops-algebra/api:scan-target

# Pregled detaljnog izvještaja s CVE oznakama
trivy image --severity HIGH,CRITICAL --ignore-unfixed \
  --format table ghcr.io/<owner>/devops-algebra/api:scan-target

# Provjera npm ovisnosti lokalno
cd api && npm audit --omit=dev

# Provjera koji paket uvodi ranjivost (stablo ovisnosti)
npm ls <naziv-paketa>
```

U izlazu tražiti:
- Naziv paketa i verziju s ranjivošću
- CVE oznaku (npr. `GHSA-w5hq-g745-h8pq`)
- Verziju koja ispravlja ranjivost (`Fixed Version`)

### 4.3 Uzrok

Trivy skeniranje pronašlo je paket s poznatom HIGH ili CRITICAL ranjivošću
za koju postoji dostupan ispravak (konfiguracija `ignore-unfixed: true` znači
da se prijavljuju samo ranjivosti koje imaju fiksiranu verziju).

Ranjivosti u Node.js projektima najčešće dolaze iz:
- Direktnih ovisnosti navedenih u `package.json` (npr. `uuid`, `express`)
- Tranzitivnih ovisnosti koje direktni paketi uvlače kao svoje ovisnosti

**Stvarni primjer iz ovog projekta**: Trivy je blokirao objavu `api` slike zbog
ranjivosti `GHSA-w5hq-g745-h8pq` u paketu `uuid@10.x` (nedostatak provjere
granica međuspremnika). Ranjivost je ispravljena nadogradnjom na `uuid@14.0.0`.

### 4.4 Ispravak

**Korak 1 — Identificirati paket i dostupnu ispravku** (iz Trivy izvještaja)

**Korak 2 — Ažurirati paket u `package.json`:**

```bash
# Primjer: ažuriranje uuid na sigurnu verziju
cd api
# Promijeniti u package.json: "uuid": "^10.0.0" → "uuid": "^14.0.0"
npm install
npm audit --omit=dev
# Očekivani izlaz: "found 0 vulnerabilities"
```

**Korak 3 — Ako ažuriranje nije moguće unutar semver raspona, koristiti override:**

```json
"overrides": {
  "vulnerable-package": ">=sigurna-verzija"
}
```

**Korak 4 — Commitati ažurirani `package.json` i `package-lock.json`:**

```bash
git add api/package.json api/package-lock.json
git commit -m "fix: ažuriraj <paket> zbog <CVE-oznaka>"
git push
```

### 4.5 Validacija

```bash
# Provjera lokalnog audit izvještaja
cd api && npm audit --omit=dev
# Očekivani izlaz: "found 0 vulnerabilities"

# Provjera GitHub Actions pipelina nakon pusha
# Očekivani izlaz: sve zeleno, slika objavljena u registar

# Provjera objavljene slike u ghcr.io
# GitHub → Packages → api → provjeriti novi tag s aktualnim SHA
```

Potvrda uspjeha: GitHub Actions korak "Trivy skeniranje ranjivosti" završava
bez grešaka i sljedeći korak "Objava slike u registar" se izvršava.

### 4.6 Prevencija

- Dependabot je konfiguriran za tjednu provjeru npm ovisnosti — automatski
  otvara pull request pri svakom sigurnosnom ažuriranju paketa
- Svaki Dependabot PR prolazi kroz isti CI/CD quality gate uključujući Trivy
  skeniranje — PR se ne spaja dok Trivy ne potvrdi 0 ranjivosti razine HIGH
  ili CRITICAL
- Koristiti `npm overrides` u `package.json` za forsiranje sigurne verzije
  tranzitivne ovisnosti kada direktni paket još nije objavio ispravak
