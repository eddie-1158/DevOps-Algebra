# Arhitektura sustava — Secure Event Ticketing Platform

## 1. Pregled projekta

Secure Event Ticketing Platform je višeslojna web-aplikacija za prodaju ulaznica za događaje.
Platforma je izgrađena kao skup međusobno neovisnih servisa koji komuniciraju kroz definirana
sučelja, a cijeli sustav se isporučuje kroz kontejnerski pristup koristeći Podman/Docker za
lokalni razvoj i Kubernetes za produkcijsko okruženje.

---

## 2. Arhitekturni dijagram

```
                         KORISNIK (web-preglednik)
                                    │
                                    │ HTTPS (port 443)
                                    ▼
                    ┌───────────────────────────────┐
                    │        INGRESS / NGINX         │
                    │   (usmjeravanje prometa)       │
                    └──────────┬────────────┬────────┘
                               │            │
                    /          │            │  /api/*
                               ▼            ▼
          ┌────────────────────────┐   ┌────────────────────────┐
          │    WEB FRONTEND        │   │    API SERVIS           │
          │    Node.js             │◀──│    Node.js / Express    │
          │    port 3000           │   │    port 8080            │
          │                        │   │                        │
          │  Servira statički HTML │   │  GET  /events          │
          │  Poziva API za podatke │   │  POST /tickets/purchase│
          │  /healthz              │   │  GET  /tickets/orders  │
          │  /config               │   │  /healthz  /readyz     │
          └────────────────────────┘   └────────┬───────┬───────┘
                                                │       │
                              SQL upiti         │       │  RPUSH (dodaj u red)
                                                ▼       ▼
                               ┌──────────────────┐  ┌─────────────────────┐
                               │   POSTGRESQL     │  │   REDIS             │
                               │   port 5432      │  │   port 6379         │
                               │                  │  │                     │
                               │  ticket_orders   │  │  red: ticket_orders │
                               │  (trajna pohrana)│  │  (predmemorija)     │
                               └──────────────────┘  └──────────┬──────────┘
                                       ▲                         │
                                       │ INSERT narudžbe         │ BRPOP (čitaj iz reda)
                                       │                         ▼
                                       │            ┌────────────────────────┐
                                       └────────────│   WORKER               │
                                                    │   Node.js              │
                                                    │                        │
                                                    │  Čita iz Redis reda    │
                                                    │  Sprema u PostgreSQL   │
                                                    │  Šalje e-mail potvrde  │
                                                    └────────────────────────┘
```

### Tok zahtjeva — kupnja ulaznice

```
 1. Korisnik ──▶ Frontend  (odabir događaja, unos e-mail adrese)
 2. Frontend ──▶ API       POST /api/tickets/purchase
 3. API      ──▶ Redis     RPUSH ticket_orders (narudžba u red)
 4. API      ──▶ Korisnik  HTTP 202 Accepted (orderId)
 5. Worker   ──▶ Redis     BRPOP (preuzima narudžbu iz reda)
 6. Worker   ──▶ PostgreSQL INSERT ticket_orders (sprema narudžbu)
 7. Worker   ──▶ (e-mail)  Šalje potvrdu kupnje
```

---

## 3. Usporedba kontejnera i virtualnih strojeva

### 3.1 Usporedna tablica

| Kriterij                  | Virtualni stroj (VM)                                        | Kontejner                                                    |
|---------------------------|-------------------------------------------------------------|--------------------------------------------------------------|
| Izolacija             | Hardverska izolacija kroz hypervisor (KVM, VMware, Hyper-V) | Izolacija na razini procesa (Linux namespaces + cgroups)     |
| Dijeljenje jezgre     | Svaki VM pokreće vlastitu kopiju jezgre OS-a                | Svi kontejneri dijele jezgru domaćinskog OS-a               |
| Veličina slike        | 1–40 GB (cijeli OS + aplikacija)                            | 5–500 MB (samo aplikacija i ovisnosti)                       |
| Vrijeme pokretanja    | 30 sekundi – nekoliko minuta (pokretanje OS-a)              | < 1 sekunda (pokretanje procesa)                             |
| Overhead memorije     | Visok — svaki VM rezervira memoriju za vlastiti OS          | Nizak — nema dupliciranja OS-a između kontejnera             |
| Portabilnost          | Ovisi o hypervisoru; migracija između providera je složena  | Ista slika radi identično lokalno, u CI/CD i produkciji      |
| Sigurnost             | Jača izolacija; kompromitiranje jednog VM-a ne utječe na druge | Manja izolacija; napad na jezgru može probiti granice   |
| Skalabilnost          | Sporo horizontalno skaliranje (pokretanje novog VM-a = minute) | Brzo skaliranje (novi kontejner = sekunde)                |
| Reproducibilnost      | VM snapshot nije prenosiv između hypervisora                | Kontejnerska slika je nepromjenjiva i verzionirana           |
| Upravljanje           | Ručno ili kroz alate poput Ansible/Terraform                | Automatsko kroz Kubernetes (self-healing, auto-scaling)      |

### 3.2 Portabilnost

Kontejneri rješavaju klasični problem razvojnog inženjerstva: "radi na mom računalu, ne radi na
produkciji". Kontejnerska slika enkapsulira aplikaciju zajedno s točno određenim verzijama svih
ovisnosti (Node.js 20.x, express 4.18.2, pg 8.11.3). Ta ista slika se pokreće identično na:
- razvojnom računalu razvojnog inženjera (podman/docker)
- GitHub Actions CI okruženju
- produkcijskom Kubernetes klasteru

Virtualni strojevi ne garantiraju ovu razinu konzistentnosti jer konfiguracija OS-a, instalirani
paketi i sistemske biblioteke variraju između okruženja.

### 3.3 Sigurnost

Virtualni strojevi nude jaču izolaciju na razini hardvera — kompromitiranje jednog VM-a ne
ugrožava izravno susjedne VM-ove jer svaki ima zasebnu jezgru OS-a.

Kontejneri dijele jezgru domaćina, što znači da ranjivost na razini jezgre može potencijalno
probiti izolaciju. Ovaj rizik se u ovom projektu ublažava sljedećim mjerama:

- Non-root korisnik — svi kontejneri pokreću procese s UID 10000+ (bez root privilegija)
- Minimalan base image — `node:20-alpine` smanjuje napadačku površinu eliminacijom
  nepotrebnih sistemskih alata
- Uklanjanje npm iz runtime sloja — npm nije potreban u produkcijskom sloju, a ima poznate
  ranjivosti u `node:20-alpine`
- Kubernetes RBAC — ServiceAccount s minimalnim dozvolama (princip najmanjih ovlasti)
- NetworkPolicy — segmentacija prometa između servisa (worker ne može direktno zvati frontend)

### 3.4 Overhead i učinkovitost resursa

Na poslužitelju s 32 GB RAM-a i 8 jezgri:

- VM pristup (10 VM-ova): svaki VM zauzima ~2 GB RAM-a za OS → 20 GB za OS, samo 12 GB
  za aplikacije; pokretanje novog VM-a traje 1–3 minute
- Kontejnerski pristup (10 kontejnera): OS se ne duplicira → 30+ GB dostupno za aplikacije;
  pokretanje novog kontejnera traje < 1 sekundu

### 3.5 Zaključak odabira — zašto kontejneri za ovaj projekt

Za Secure Event Ticketing Platform kontejneri su ispravniji odabir jer:

1. Mikroservisna arhitektura zahtijeva neovisno skaliranje — u periodu visoke potražnje
   (koncerti, sportski događaji) worker servis treba 5× više instanci od frontenda; kontejneri
   to omogućuju bez mijenjanja ostatka sustava
2. CI/CD pipeline — GitHub Actions gradi sliku jednom i isporučuje je na bilo koje okruženje
3. Lokalni razvoj — jednom naredbom (`podman compose up`) pokreće se identičan stack kao u
   produkciji, što eliminira greške uzrokovane razlikama u okruženju
4. Troškovna učinkovitost — manji overhead od VM-ova znači više servisa na istom hardveru

---

## 4. Odabir servisa i njihove uloge

### 4.1 Web Frontend — `frontend/` (port 3000)

Uloga: Jedina točka kontakta s korisnikom. Servira statičke HTML/CSS/JS datoteke i
omogućuje pregled događaja te kupnju ulaznica.

Ključne značajke:
- `/healthz` — provjera zdravlja (liveness sonda)
- `/config` — vraća API URL (omogućuje konfiguraciju bez ponovnog izgradnje slike)
- Statički sadržaj serviran kroz Express

Međuservisna komunikacija: HTTP pozivi prema API servisu (`/api/events`, `/api/tickets/purchase`)

### 4.2 API Servis — `api/` (port 8080)

Uloga: Srce poslovne logike. Jedina komponenta koja smije pisati u PostgreSQL i dodavati
zadatke u Redis red.

Ključni endpointi:
- `GET /healthz` — liveness sonda
- `GET /readyz` — readiness sonda (provjerava PostgreSQL i Redis vezu)
- `GET /events` — popis dostupnih događaja
- `POST /tickets/purchase` — kupnja ulaznice (dodaje narudžbu u Redis red)
- `GET /tickets/orders` — pregled narudžbi (čita iz PostgreSQL)

Međuservisna komunikacija: PostgreSQL (SQL upiti), Redis (RPUSH u red zadataka)

### 4.3 Background Worker — `worker/` (bez izloženog porta)

Uloga: Asinkrona obrada narudžbi. Rasterećuje API servis od dugotrajnih operacija
i osigurava da se svaka narudžba obradi čak i pri privremenom nedostupnosti PostgreSQL.

Princip rada:
- `BRPOP ticket_orders 0` — blokirajuće čekanje na novi zadatak iz Redis reda
- `INSERT INTO ticket_orders` — trajno zapisuje narudžbu u PostgreSQL
- U produkciji: slanje e-mail potvrde kupcu

Zašto poseban servis: Odvajanje asinkronih operacija poboljšava odzivnost API servisa
(HTTP 202 vraća se trenutno) i omogućuje neovisno skaliranje workera prema opterećenju.

### 4.4 PostgreSQL — `infra/postgres/` (port 5432)

Uloga: Trajna relacijska pohrana podataka aplikacije.

Shema: Tablica `ticket_orders` s indeksom na `created_at` za brze upite.

Zašto PostgreSQL: ACID transakcije osiguravaju konzistentnost — sprječavaju dvostruku
obradu iste narudžbe (`ON CONFLICT (order_id) DO NOTHING`).

### 4.5 Redis (port 6379)

Uloga (dvije funkcije):
1. Red zadataka — API servis dodaje narudžbe (`RPUSH`), worker ih preuzima (`BRPOP`)
2. Predmemorija sesija — brzo dohvaćanje često traženih podataka

Zašto Redis: In-memory pohrana pruža mikrosekundne latencije, idealno za asinkroni
red zadataka i privremenu predmemoriju.

---

## 5. Međuservisna komunikacija

```
┌────────────────────────────────────────────────────────────────────┐
│                   Protokoli međuservisne komunikacije              │
├─────────────────┬──────────────────┬──────────────────────────────┤
│ Izvor           │ Odredište        │ Protokol / operacija         │
├─────────────────┼──────────────────┼──────────────────────────────┤
│ Korisnik        │ Frontend         │ HTTP/HTTPS (port 3000)       │
│ Frontend        │ API              │ HTTP/REST — JSON (port 8080) │
│ API             │ PostgreSQL       │ TCP — PostgreSQL protokol    │
│ API             │ Redis            │ TCP — RESP protokol (RPUSH)  │
│ Worker          │ Redis            │ TCP — RESP protokol (BRPOP)  │
│ Worker          │ PostgreSQL       │ TCP — PostgreSQL protokol    │
└─────────────────┴──────────────────┴──────────────────────────────┘
```

Sigurnosna načela komunikacije:
- PostgreSQL, Redis i worker nisu izloženi javnoj mreži — dostupni samo unutar
  interne Docker/Kubernetes mreže
- API servis je jedina komponenta koja ima pravo pisanja u obje pohrane podataka
- Frontend ne komunicira direktno s bazom podataka ni s Redisom

---

## 6. Usklađenost arhitekture s projektnim ciljevima

| Projektni cilj                       | Kako arhitektura ispunjava cilj                                              |
|--------------------------------------|------------------------------------------------------------------------------|
| Sigurna višeslojna aplikacija        | Svaki sloj ima jasnu ulogu i ograničen pristup; nema direktnog pristupa bazi |
| Kontejnerizacija svih servisa        | Svaki servis ima vlastiti Containerfile s multi-stage buildom i non-root UID |
| Lokalni razvoj jednom naredbom       | `podman compose up` pokreće svih 5 servisa s hot-reload podrškom             |
| CI/CD automatizacija                 | GitHub Actions gradi, testira i objavljuje slike pri svakom git push         |
| Kubernetes orkestracija              | Manifesti definiraju isporuku, sonde zdravlja, resurse i mrežne politike     |
| DevSecOps sigurnosne provjere        | Trivy skenira slike, RBAC ograničava pristup, Secrets štite lozinke i ključeve |
| Asinkrona obrada narudžbi            | Worker servis preuzima zadatke iz Redis reda neovisno o API servisu          |
