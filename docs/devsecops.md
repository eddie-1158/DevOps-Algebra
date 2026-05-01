# DevSecOps metodologija — Secure Event Ticketing Platform

## 1. Pregled DevSecOps pristupa

DevSecOps je metodologija koja integrira sigurnosne provjere u svaki korak razvojnog
i isporučnog procesa — umjesto da sigurnost bude naknadna misao, ona je dio svakog
git push-a i svake isporuke.

U ovom projektu sigurnost je ugrađena na tri razine:

    Kod  →  Kontejnerska slika  →  Infrastruktura
     |              |                     |
  Dependabot     Trivy scan          Kubernetes
  npm audit      exit-code 1         Secrets + RBAC

---

## 2. Sigurnosne provjere u CI/CD toku

### 2.1 Pregled svih provjera

| Provjera                    | Alat              | Kada se izvršava         | Blokirajuća |
|-----------------------------|-------------------|--------------------------|-------------|
| Instalacija ovisnosti       | npm ci            | Svaki push / PR          | Da          |
| Skeniranje ranjivosti slike | Trivy             | Nakon izgradnje slike    | Da          |
| Automatsko ažuriranje paketa| Dependabot        | Svaki ponedjeljak        | Da (PR)     |
| Provjera tajni u kodu       | GitHub Secret scan| Svaki push               | Da          |

### 2.2 Trivy skeniranje kontejnerskih slika

Trivy skenira svaku izgrađenu kontejnersku sliku prije nego što se objavi u registar.
Skeniraju se dvije kategorije:

- OS paketi (Alpine Linux apk paketi)
- Ovisnosti aplikacije (Node.js paketi iz package-lock.json)

Konfiguracija u CI/CD:

    exit-code: '1'          # pipeline se prekida ako se pronađe ranjivost
    severity: HIGH,CRITICAL # skeniraju se samo visoke i kritične razine
    ignore-unfixed: true    # ignoriraju se ranjivosti bez dostupnog ispravka

Bez ove konfiguracije (exit-code: '0') skeniranje bi samo prijavilo nalaze ali ih ne
bi blokiralo — slika bi bila objavljena čak i s kritičnim ranjivostima.

### 2.3 npm ci umjesto npm install

Svi Containerfilevi koriste npm ci koji:
- Instalira točno verzije iz package-lock.json (reproducibilni buildovi)
- Briše node_modules i reinstalira od nule (nema zaostalih paketa)
- Brži je od npm install u CI okruženju

---

## 3. Quality gate prije objave slika

Quality gate je skup automatiziranih provjera koje moraju proći prije nego što
kontejnerska slika bude objavljena u registar i isporučena na produkcijsko okruženje.

### Tijek quality gatea

    git push
        │
        ▼
    [1] Build i test (npm ci)
        │  neuspjeh → pipeline se prekida, slika se NE gradi
        ▼
    [2] Izgradnja kontejnerske slike
        │  neuspjeh → pipeline se prekida, slika se NE objavljuje
        ▼
    [3] Trivy skeniranje (exit-code: '1')
        │  HIGH/CRITICAL ranjivost → pipeline se prekida, slika se NE objavljuje
        ▼
    [4] Objava slike u ghcr.io registar
        │
        ▼
    [5] SemVer označavanje (samo na git tag)

Svaki korak mora uspješno završiti da bi se prešlo na sljedeći. Ovo garantira
da u produkciju nikad ne dospije slika koja nije prošla sve provjere.

---

## 4. Upravljanje tajnama i konfiguracijom

### 4.1 Bez hardkodiranih vrijednosti

Nijedna lozinka, API ključ ni token nije upisana direktno u kod ili Containerfile.
Sve osjetljive vrijednosti dolaze iz okruženja:

| Vrijednost              | Lokalni razvoj        | Produkcija (Kubernetes)  |
|-------------------------|-----------------------|--------------------------|
| Lozinka baze podataka   | .env (lokalno)        | Kubernetes Secret        |
| Redis lozinka           | .env (lokalno)        | Kubernetes Secret        |
| GitHub token za registar| GitHub Actions Secret | Kubernetes Secret        |

### 4.2 .env.example kao dokumentacija

Fajl .env.example sadrži popis svih varijabli okruženja s opisnim primjernim
vrijednostima — bez stvarnih lozinki. Razvojni inženjeri kopiraju fajl u .env
i upisuju lokalne vrijednosti. Fajl .env je uvijek u .gitignore.

### 4.3 Kubernetes Secrets

U produkcijskom okruženju sve tajne se pohranjuju kao Kubernetes Secret objekti:

    kubectl create secret generic ticketing-secrets \
      --from-literal=POSTGRES_PASSWORD=<vrijednost> \
      --from-literal=REDIS_PASSWORD=<vrijednost>

Kubernetes Secrets su base64 kodirani i dostupni samo podovima koji ih eksplicitno
referenciraju kroz envFrom ili env.valueFrom.secretKeyRef.

---

## 5. Alati i obrazloženje odabira

### Trivy — skeniranje ranjivosti

Trivy je open-source alat tvrtke Aqua Security za skeniranje kontejnerskih slika,
datotečnih sustava i git repozitorija na poznate ranjivosti (CVE).

Odabran jer:
- Integrira se kao GitHub Actions korak bez dodatne infrastrukture
- Skenira i OS pakete i Node.js ovisnosti u jednom prolazu
- Podržava SARIF format za prikaz u GitHub Security tabu
- Aktivno se održava s dnevnim ažuriranjima baze ranjivosti

### Dependabot — automatsko ažuriranje ovisnosti

Dependabot je GitHub-ova ugrađena usluga koja svaki ponedjeljak provjerava postoje
li novije verzije npm paketa i GitHub Actions i automatski otvara pull request
s ažuriranom verzijom.

Odabran jer:
- Eliminira ručno praćenje sigurnosnih zakrpa
- Svaki Dependabot PR prolazi kroz isti CI/CD quality gate (Trivy + testovi)
- Promjene su male i lako pregledljive (jedan paket po PR-u)

### npm ci — reproducibilni buildovi

npm ci koristi package-lock.json kao jedini izvor istine za verzije ovisnosti.
Za razliku od npm install koji može nadograditi pakete unutar semver raspona,
npm ci instalira točno ono što je zapisano u lockfileu.

---

## 6. Nalazi i korektivne mjere

### 6.1 Pronađena ranjivost: npm u node:20-alpine runtime sloju

Simptom: Trivy prijavljuje ranjivost razine HIGH u npm paketu prisutnom u
node:20-alpine baznoj slici.

Uzrok: node:20-alpine uključuje npm globalno instaliran u slici, a npm u toj
verziji ima poznatu ranjivost koja je popravljena u novijim verzijama.

Korektivna mjera (automatizirana):
Uklanjanje npm iz runtime sloja u svakom Containerfileu odmah nakon bazne slike:

    FROM node:20-alpine AS runtime
    RUN apk del npm

Ova mjera je trajna i automatska — svaki novi build runtime sloja ne sadrži npm.

### 6.2 Zastarjele ovisnosti Node.js paketa

Simptom: Trivy ili npm audit prijavljuju ranjivost u tranzitivnoj ovisnosti
(npr. ranjiva verzija express ili redis klijenta).

Uzrok: Tranzitivne ovisnosti se ne ažuriraju automatski ako nije eksplicitno
konfigurirano praćenje.

Korektivne mjere (automatizirane):

1. Dependabot otvara PR s novom verzijom paketa svaki ponedjeljak
2. Ako Dependabot ne može automatski riješiti konflikt, koristi se npm override
   u package.json za forsiranje sigurne verzije tranzitivne ovisnosti:

        "overrides": {
          "vulnerable-package": ">=2.1.0"
        }

3. PR prolazi kroz CI/CD pipeline koji uključuje Trivy skeniranje — ako nova
   verzija i dalje ima ranjivost, PR se ne spaja (merge) dok problem nije riješen.

### 6.3 Hardkodirana vrijednost u kodu

Simptom: GitHub Secret scanning pronalazi token ili lozinku commitanu u repozitorij.

Uzrok: Razvojni inženjer je slučajno commitao .env fajl ili upisao vrijednost
direktno u kod.

Korektivna mjera (automatizirana):
GitHub Secret scanning blokira push automatski ako detektira poznati format tajne
(AWS ključ, GitHub token, lozinka u connection stringu i sl.). Vrijednost se mora:

1. Ukloniti iz git povijesti (git filter-repo)
2. Odmah poništiti (rotirati) na usluzi koja je izdala tajnu
3. Premjestiti u .env (lokalno) ili Kubernetes Secret (produkcija)

### 6.4 Stvarni nalaz: ranjivost uuid paketa (GHSA-w5hq-g745-h8pq)

Simptom: Trivy skeniranje blokiralo je objavu api slike s izlaznim kodom 1.
Prijavljena ranjivost razine HIGH u paketu uuid verzije 10.x.

Uzrok: Paket uuid < 14.0.0 sadrži nedostatak provjere granica međuspremnika
(buffer bounds check) u funkcijama v3/v5/v6 kada se koristi s parametrom buf.
Ranjivost je identificirana u bazi CVE pod oznakom GHSA-w5hq-g745-h8pq.

Korektivna mjera (automatizirana):
Paket je ažuriran na verziju ^14.0.0 u api/package.json, a package-lock.json
je ažuriran naredbom npm install:

    "uuid": "^14.0.0"

Verifikacija: Trivy skeniranje nakon ažuriranja prijavljuje 0 ranjivosti.
Pipeline je nastavljen i slika je uspješno objavljena u registar.

Prevencija: Dependabot je konfiguriran za tjednu provjeru npm ovisnosti i
automatski otvara pull request pri svakom novom sigurnosnom ažuriranju paketa.
