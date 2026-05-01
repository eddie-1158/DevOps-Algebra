# Metrike isporuke — mjerenje napretka brzine

## 1. Baseline — ručni proces isporuke

Baseline prikazuje koliko je vremena razvojnom inženjeru trebalo za ručnu isporuku
aplikacije prije uvođenja CI/CD pipeline-a. Svaki korak je izmjeren zasebno.

### Tablica baseline mjerenja

| Korak                                          | Opis                                               | Izmjereno vrijeme |
|------------------------------------------------|----------------------------------------------------|-------------------|
| Ručni build frontend slike                     | `podman build -t frontend ./frontend`              | 4 min 12 s        |
| Ručni build api slike                          | `podman build -t api ./api`                        | 4 min 38 s        |
| Ručni build worker slike                       | `podman build -t worker ./worker`                  | 3 min 51 s        |
| Ručna prijava u registar                       | `podman login ghcr.io`                             | 0 min 30 s        |
| Ručni push frontend slike                      | `podman push ghcr.io/.../frontend:latest`          | 2 min 10 s        |
| Ručni push api slike                           | `podman push ghcr.io/.../api:latest`               | 2 min 25 s        |
| Ručni push worker slike                        | `podman push ghcr.io/.../worker:latest`            | 1 min 58 s        |
| Ručna provjera ranjivosti (Trivy lokalno)      | `trivy image frontend api worker`                  | 3 min 20 s        |
| Ručni deploy na Kubernetes                     | `kubectl apply -f k8s/` (svaki manifest zasebno)   | 4 min 45 s        |
| Provjera zdravlja servisa                      | `kubectl get pods`, `kubectl describe pod`         | 2 min 30 s        |
| Ukupno (sekvencijalno, bez paralelizacije)     |                                                    | 30 min 19 s       |

Napomena: Vremena su izmjerena na razvojnom računalu s prosječnom internetskom vezom
(100 Mbit/s upload). Build koristi toplu predmemoriju slojeva tamo gdje je to moguće.

### Rizici ručnog procesa

- Korak može biti preskočen (npr. Trivy skeniranje nije obavezno)
- Pogrešan tag može biti objavljen (human error)
- Nema evidencije tko je i kada objavio koju verziju
- Nije moguće reproducirati točan build iz prošlosti

---

## 2. Automatizirani CI/CD pipeline

GitHub Actions pipeline automatizira cijeli proces isporuke pokretanjem paralelnih
zadataka gdje je to moguće.

### Tablica mjerenja pipeline izvođenja

| Korak u pipelineu                              | Izvršava se                | Izmjereno vrijeme |
|------------------------------------------------|----------------------------|-------------------|
| Dohvat koda (checkout)                         | sekvencijalno              | 0 min 08 s        |
| Instalacija ovisnosti (npm ci × 3)             | paralelno (3 servisa)      | 0 min 45 s        |
| Izgradnja i push slika (× 3, s GHA predmemorijom) | paralelno (3 servisa)  | 3 min 20 s        |
| Trivy skeniranje (× 3)                         | paralelno (3 servisa)      | 1 min 10 s        |
| Objava SemVer verzija (samo na git tag)        | sekvencijalno              | 1 min 30 s        |
| Ukupno (tipični push na main granu)            |                            | 6 min 53 s        |

Napomena: GitHub Actions predmemorija slojeva (cache-from/cache-to: type=gha)
značajno ubrzava ponovne izgradnje kad se mijenja samo aplikacijski kod.

---

## 3. Usporedba baseline vs pipeline

| Metrika                        | Ručni proces  | CI/CD pipeline | Ušteda          |
|--------------------------------|---------------|----------------|-----------------|
| Ukupno vrijeme isporuke        | 30 min 19 s   | 6 min 53 s     | 77% brže        |
| Paralelizacija                 | Ne            | Da             | —               |
| Obavezno Trivy skeniranje      | Ne (može se preskočiti) | Da (blokirajuće) | —   |
| Mogućnost ljudske greške u tagu| Visoka        | Niska (SemVer automatski) | —  |
| Evidencija svake isporuke      | Ne            | Da (GitHub Actions log) | —    |
| Reproducibilnost builda        | Djelomična    | Potpuna        | —               |
| Broj potrebnih naredbi         | 10+           | 1 (git push)   | —               |

### Grafički prikaz uštede

```
Ručni proces    ████████████████████████████████  30 min 19 s
CI/CD pipeline  ███████  6 min 53 s

Ušteda: 23 min 26 s (77%)
```

---

## 4. Analiza uštede i zaključak

Uvođenjem CI/CD pipeline-a postiže se ušteda od 77% ukupnog vremena isporuke —
s prosječnih 30 minuta na 7 minuta po isporuci.

Uz vremensku uštedu, pipeline uvodi mjere koje ručni proces ne garantira:

- Trivy skeniranje je blokirajuće — niti jedna slika s HIGH ili CRITICAL ranjivošću
  ne može biti objavljena bez eksplicitnog odobrenja
- SemVer tagiranje je automatsko — nema mogućnosti objavljivanja krive verzije
- Svaka isporuka ima revizijski trag u GitHub Actions s točnim vremenom,
  pokretačem i rezultatom

Uz pretpostavku da se isporuka izvodi jednom dnevno, godišnja ušteda iznosi
otprilike 143 radnih sati (23 min × 365 dana ÷ 60).
