# Sigurnosno izvješće

## Opseg i alat

Skeniraju se sve tri slike koje projekt gradi i objavljuje: `ticketing-api`,
`ticketing-frontend` i `ticketing-worker`. Postgres i Redis se povlače kao
službene slike, pa se njih ne skenira.

Alat je **Trivy**, zadržan na verziji `0.74.0` kako bi rezultat skeniranja bio
ponovljiv.

Skenira se u dva konteksta:

- **lokalno**, tijekom razvoja, radi brze provjere prije commita
- **u CI pipelineu**, pri svakom buildu, kao uvjet za objavu slike

## Postupak

Prije skeniranja slike treba ponovno sagraditi kako bi uključile ažurirane
pakete:

```bash
docker build --target prod -t ticketing-api:1.0.0 ./api
docker build --target prod -t ticketing-frontend:1.0.0 ./frontend
docker build --target prod -t ticketing-worker:1.0.0 ./worker
```

Skeniranje:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v trivy-cache:/root/.cache/ \
  aquasec/trivy:0.74.0 image ticketing-api:1.0.0
```

Isto za `ticketing-frontend:1.0.0` i `ticketing-worker:1.0.0`.

Socket Docker daemona potreban je da bi Trivy vidio lokalno sagrađene slike.
Volumen `trivy-cache` trajno pohranjuje bazu ranjivosti, prvo skeniranje je
dosta sporije jer je mora preuzeti u cijelosti.

## Rezultati

Stanje na dan **23.08.26**, verzija slika **1.0.0**, Trivy `0.74.0`.

| Slika | CRITICAL | HIGH | MEDIUM | LOW | Tajne |
|---|---|---|---|---|---|
| ticketing-api | 1 | 0 | 0 | 0 | 0 |
| ticketing-frontend | 0 | 0 | 0 | 0 | 0 |
| ticketing-worker | 0 | 0 | 0 | 0 | 0 |

Sirovi ispisi nalaze se u prilogu ovog dokumenta.

## Nalazi i mjere

### N1 — CVE-2026-59873, `tar` 7.5.11 (CRITICAL)

**Nalaz.** Trivy je u slici `ticketing-api:sha-c0ee751` prijavio jednu kritičnu
ranjivost: `tar` 7.5.11, DoS kroz izrađenu gzip bombu. Zakrpa
postoji u verziji 7.5.19.

**Lokacija.** `/usr/local/lib/node_modules/npm/node_modules/tar` — kopija koju
sa sobom nosi npm ugrađen u baznu sliku `node:22-alpine`.  Nije ovisnost aplikacija

**Kada je otkriven.** U CI pipelineu, prije objave. Quality gate je zaustavio
build s izlaznim kodom 1 i slika nije objavljena u registar.

**Mjera.** Budući da paket ne dolazi kroz `package.json` aplikacije,
ažuriranje verzije nije bilo primjenjivo. Umjesto zakrpe uklonjen je uzrok: iz
produkcijske faze izbačeni su npm, corepack i yarn.

```dockerfile
RUN rm -rf /usr/local/lib/node_modules/npm \
           /usr/local/lib/node_modules/corepack \
           /usr/local/bin/npm /usr/local/bin/npx /usr/local/bin/corepack \
           /opt/yarn-v1.22.22 /usr/local/bin/yarn /usr/local/bin/yarnpkg
```

Runtime izvršava isključivo `node src/server.js` i alati za instalaciju
paketa mu nisu potrebni.

**Validacija.** Ponovljeni sken: 0 CRITICAL. Pipeline zelen, slika uspješno
objavljena. Veličina slike smanjena za **10 MB**.

**Napomena.** Mjera uz uklanjanje ranjivosti smanjuje i površinu napada, u
produkcijskoj slici više nema alata za instalaciju paketa, pa ih ne može
iskoristiti ni netko tko dobije pristup kontejneru.

### Čisti rezultati

Uz nalaz N1, ostatak skeniranja nije prijavio ništa:

- Alpine 3.24.1, 18 OS paketa, 0 ranjivosti
- Aplikacijske ovisnosti (`app/node_modules`), 0 ranjivosti
- Skeniranje tajni je čisto. Dockerignore, blokira .env tajne da dospiju u sliku

## Quality gate u CI pipelineu

Pipeline razdvaja **izvještavanje** od **blokiranja**:

| Korak | Prag | Izlazni kod | Učinak |
|---|---|---|---|
| Trivy analiza | CRITICAL, HIGH, MEDIUM | 0 | samo prijavljuje |
| Trivy gate | CRITICAL, `--ignore-unfixed` | 1 | zaustavlja pipeline |

Prag za blokiranje namjerno je postavljen na CRITICAL uz `--ignore-unfixed`.
Gate na razini HIGH obarao bi build i zbog ranjivosti za koje zakrpa ne
postoji, a gate koji se pali bez mogućnosti rješavanja problema u praksi
završi tako da ga se isključi. Zato se prijavljuje sve, a blokira samo ono
kritično što je moguće popraviti.

Redoslijed koraka je bitan: build → skeniranje → gate → prijava u registar →
objava. Padne li gate, koraci objave se ne izvršavaju i ranjiva slika nikad ne
dođe do registra. Skeniranje nakon objave ne bi bilo quality gate nego
izvješće o šteti.

## Politika tagiranja i objave slika

| Tag | Kada nastaje | Namjena |
|---|---|---|
| `sha-<commit>` | svaki build | nepromjenjiv, sljediv do točnog commita |
| `v<semver>` | samo pri GitHub releaseu | verzija čitljiva ljudima |

Objava se događa **isključivo na `release` događaj**, i to iza quality gatea.
Push u `main` i pull requestovi pokreću build i skeniranje, ali ne i objavu,
time registar ostaje čist, a objava je svjesna odluka umjesto nusprodukta
svakog commita.

Tag `latest` se ne koristi za deployment. Pomičan je, pa iz njega nije moguće
zaključiti koja verzija koda zapravo radi u okruženju niti se pouzdano vratiti
na prethodnu. U deployment manifestima koristi se `sha-` tag.

Registar je GitHub Container Registry (`ghcr.io/kemilab/`). Prijava u CI-ju
ide ugrađenim `GITHUB_TOKEN`-om s dozvolom `packages: write`, bez dodatnih
tajni u repozitoriju.

## Prilozi

- `scan-api-1.0.0.txt`
- `scan-frontend-1.0.0.txt`
- `scan-worker-1.0.0.txt`

Ispisi se generiraju preusmjeravanjem izlaza:

```bash
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  -v trivy-cache:/root/.cache/ aquasec/trivy:0.74.0 \
  image ticketing-api:1.0.0 > docs/security/scan-api-1.0.0.txt
```

Verzija slike upisana je u naziv datoteke jer Trivy datum skeniranja ne
zapisuje u sam ispis.