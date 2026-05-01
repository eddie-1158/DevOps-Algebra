# Politika upravljanja kontejnerskim slikama

## 1. Prakse sigurnosti pri izradi slika

Sve kontejnerske slike u ovom projektu grade se prema sljedećim sigurnosnim načelima:

### Multi-stage build

Svaki Containerfile koristi dvostupanjski postupak izgradnje:

- Stupanj izgradnje (builder) — instalira sve ovisnosti i priprema aplikacijski kod
- Stupanj izvođenja (runtime) — sadrži samo ono što je potrebno za pokretanje; nema
  alata za izgradnju, nema razvojnih ovisnosti

Prednost: završna slika ne sadrži npm, gcc, make ni druge alate koji povećavaju
napadačku površinu i veličinu slike.

### Uklanjanje npm iz runtime sloja

node:20-alpine uključuje npm koji ima poznate ranjivosti (identificirane Trivy skeniranjem).
Budući da npm nije potreban za izvođenje aplikacije, uklanja se odmah na početku runtime stupnja:

    RUN apk del npm

### Instalacija samo produkcijskih ovisnosti

U builder stupnju koristi se zastavica --omit=dev kako bi se razvojne ovisnosti
(nodemon i sl.) isključile iz slike:

    RUN npm ci --omit=dev

### Non-root korisnik

Svi kontejneri pokreću procese kao neprivilegirani korisnik s UID 10000 i GID 10000:

    RUN addgroup -g 10000 appgroup && adduser -u 10000 -G appgroup -D appuser
    USER appuser

Pokretanje kao root korisnik unutar kontejnera je sigurnosni rizik — u slučaju proboja
izolacije napadač bi imao root privilegije na domaćinu.

---

## 2. Pregled slika i bazni image

| Servis   | Bazni image      | Veličina (approx.) | Izloženi port |
|----------|------------------|--------------------|---------------|
| frontend | node:20-alpine   | ~60 MB             | 3000          |
| api      | node:20-alpine   | ~65 MB             | 8080          |
| worker   | node:20-alpine   | ~55 MB             | —             |

node:20-alpine odabran je kao minimalni bazni image jer:
- Alpine Linux je manji od 10 MB (za razliku od node:20 koji je ~350 MB)
- Sadrži samo neophodne sistemske biblioteke
- Aktivno se održava i prima sigurnosne zakrpe

---

## 3. Tagiranje slika — SemVer politika

Svaka kontejnerska slika objavljuje se s tri oznake (taga) pri svakom izdanju:

    ghcr.io/matej-basic/devops-project-app/frontend:v1.2.3   <- puna SemVer verzija
    ghcr.io/matej-basic/devops-project-app/frontend:v1.2     <- minor verzija
    ghcr.io/matej-basic/devops-project-app/frontend:latest   <- zadnja stabilna

### Shema verzioniranja (SemVer)

Verzija se zapisuje u obliku vMAJOR.MINOR.PATCH:

- MAJOR — nekompatibilne promjene API-ja ili arhitekture (npr. v1.0.0 → v2.0.0)
- MINOR — nove funkcionalnosti koje su kompatibilne s prethodnom verzijom (npr. v1.1.0 → v1.2.0)
- PATCH — ispravci grešaka bez novih funkcionalnosti (npr. v1.2.0 → v1.2.1)

### Primjeri oznaka

    v1.0.0   — prvo stabilno izdanje
    v1.0.1   — ispravak greške u obradi narudžbi
    v1.1.0   — dodana podrška za više valuta
    v2.0.0   — nova verzija API-ja (nekompatibilna s v1.x)

Oznaka latest uvijek pokazuje na zadnju stabilnu verziju objavljenu s main grane.
Privremene slike iz pull request grana oznacavaju se SHA sažetkom commita i ne
objavljuju se pod latest.

### Automatizacija u CI/CD

GitHub Actions pipeline automatski određuje i primjenjuje oznake:

    - git tag v1.2.3          # razvojni inženjer označi commit
    - git push origin v1.2.3  # pokreće pipeline
    - pipeline gradi sliku i objavljuje je s oznakama v1.2.3, v1.2 i latest

---

## 4. Politika zadržavanja slika (Retention Policy)

### Pravila zadržavanja

| Vrsta oznake      | Koliko zadržati | Razlog                                          |
|-------------------|-----------------|-------------------------------------------------|
| latest            | uvijek          | Potrebna za produkcijski deploy                 |
| vMAJOR.MINOR.PATCH| zadnjih 10      | Omogućuje rollback na prethodnu stabilnu verziju|
| SHA commit oznake | 30 dana         | Kratkotrajne slike iz CI/CD procesa             |
| PR privremene     | 7 dana          | Slike za testiranje pull requestova             |

### Automatizacija brisanja

GitHub Container Registry (ghcr.io) podržava automatsko brisanje starih slika kroz
GitHub Actions workflow koji se pokreće jednom tjedno:

    - zadržava zadnjih 10 verzioniranih slika po servisu
    - briše SHA i PR slike starije od definiranog roka
    - nikad ne briše slike s oznakom latest ili aktivnom produkcijskom oznakom

### Zašto je retention policy važna

Bez politike zadržavanja, registar se s vremenom puni stotinama nepotrebnih slika što:
- povećava troškove pohrane
- otežava pronalazak ispravne verzije za rollback
- narušava sigurnost jer stare slike s poznatim ranjivostima ostaju dostupne

---

## 5. Skeniranje ranjivosti (Trivy)

Svaka slika prolazi obvezno skeniranje ranjivosti pomoću alata Trivy prije objave
u registar. Skeniranje je integrirano u GitHub Actions CI/CD pipeline.

### Konfiguracija skeniranja

- Alat: Trivy (aquasecurity/trivy-action)
- Razina blokiranja: HIGH i CRITICAL ranjivosti (exit-code: 1)
- Skeniraju se: OS paketi i Node.js ovisnosti (package-lock.json)
- Format izvještaja: SARIF (prikazuje se u GitHub Security tabu)

### Quality gate

Ako Trivy pronađe ranjivost razine HIGH ili CRITICAL, pipeline se prekida s
izlaznim kodom 1 i slika se NE objavljuje u registar. Ovo je blokirajuća provjera.
