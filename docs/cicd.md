# CI/CD pipeline

WorkFlow dateteka: `.github/workflows/publish.yml`

## Okidači

Prilikom svakoga push u repo ili otvaranja pull requesta, radi se skeniranje koda sa Trivy alatom.
Međutim, objava u registar događa se **samo pri izdanju verzije**.


## Struktura

Aplikacija ima tri servisa s identičnim koracima izgradnje, pa se koristi
matrica umjesto tri kopije istog bloka:

```yaml
strategy:
  fail-fast: false
  matrix:
    service: [api, frontend, worker]
```

`fail-fast: false` - ukoliko dođe do prekida izviđenja 
jednoga elemenat u matrici, drugi se i daje nastavljaju.

Dozvole su ograničene na razinu:

```yaml
permissions:
  contents: read
  packages: write
```

## Redoslijed koraka

Action započinje sa checkoutom koda. Izrade se sve docker slik i tagiraju se.
Trivy radi prvu analizu, u kojoj se samo prikazuje ranjivosti ali ne blokira 
pipeline. Trivy gate koji sljedi iza toga, ukoliko nađe ranivost koja se može zakrpati, blokira danji proces i sprječava da se slika objavi u registar.


## Sigurnosne provjere

Skeniranje je podijeljeno na dva koraka s različitom svrhom:

| Korak | Razine | Izlazni kod | Učinak |
|---|---|---|---|
| Trivy analiza | CRITICAL, HIGH, MEDIUM | 0 | prijavljuje, ne blokira |
| Trivy gate | CRITICAL, `--ignore-unfixed` | 1 | zaustavlja pipeline |

Prag za blokiranje postavljen je na CRITICAL uz `--ignore-unfixed`, dakle samo
na kritične ranjivosti za koje zakrpa postoji.

Trivy je zadržan na fiksnoj verziji radi ponovljivosti — s pomičnim tagom 

Detaljni nalazi i poduzete mjere: [sigurnosno izvješće](security/image_scan_reports.md).

## Objava i tagiranje

Registar je GitHub Container Registry. Prijava koristi ugrađeni `GITHUB_TOKEN`,
pa u repozitoriju nema dodatnih tajni koje bi se morale održavati ili rotirati.

Svaka objavljena slika dobiva dva taga:

| Tag | Namjena |
|---|---|
| `v<semver>` | verzija čitljiva ljudima, iz naziva releasea |
| `sha-<commit>` | nepromjenjiv, sljediv do točnog commita |

Tag `latest` se ne koristi za deployment! Moguće ga je korsiti samo za 
lokalno okruženje.

## Lanac opskrbe

Koriste  se isključivo akcije službenih izvora (`actions/checkout`,
`docker/login-action`, `aquasecurity/trivy-action`), s pinanom verzijom.

## Učinak na brzinu isporuke

| | Prije | Sada |
|---|---|---|
| Izgradnja tri slike | ručno, tri naredbe | automatski, paralelno |
| Sigurnosna provjera | ručno ili nikako | pri svakoj promjeni |
| Objava u registar | ručno `docker push` | automatski pri releaseu |
| Vrijeme od commita do slike u registru | minute ručnog rada | ~2 min bez intervencije |