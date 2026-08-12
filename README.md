# Secure Event Ticketing Platform

Ovaj repozitorij je referentni uzorak aplikacije za kolegij **Uvod u DevOps - DevSecOps**.

## Arhitektura

Detaljan pregled dostupan je na [architecture.md](docs/architecture.md)

- `frontend` - web UI za pregled evenata i kupnju karata
- `api` - REST API za evente, narudzbe i health provjere
- `worker` - pozadinska obrada queue poruka
- `postgres` - trajna pohrana narudzbi
- `redis` - red (queue)

## Podizanje testnoga okruženja

Preduvjet: Docker i Docker Compose

### Postavljanje varijabli

` cp .env.example .env`

Za **testno** okruženje nije potrebno mijenjati varijable

### Pokretanje servisa

` docker compose up -d --build`

Slike i kontejneri optimizirani su za **HotReload**

### Testiranje aplikacije

Frontend URI: ` http://localhost:3000`

Api URI: `http://localhost:8080`

### Brza validacija funkcionalnosti

1. Health API:

   ```bash
   curl http://localhost:8080/healthz
   curl http://localhost:8080/readyz
   ```
2. Dohvati evente:

   ```bash
   curl http://localhost:8080/events
   ```
3. Pošalji narudzbu:

   ```bash
   curl -X POST http://localhost:8080/tickets/purchase \
     -H "Content-Type: application/json" \
     -d '{"eventId":"evt-1001","customerEmail":"student@example.com","quantity":2}'
   ```
4. Provjeri obradene narudžbe:

   ```bash
   curl http://localhost:8080/tickets/orders
   ```
5. Testiranje skaliranja

   ```
   docker compose up -d --scale worker=4
   ```

### Praćenje statusa servisa

```
docker compose ps
docker compose logs -f
```

### Zaustavljanje

Postgress kontejner ima trajni volumen u kojemu se nalaze sve narudže. Ukoliko je potrebno kratkovremeno zastaviti projekt bez gubitna podataka:

- ` docker compose down`

Za potpuno brisanje svih podataka i volumena:

- ` docker compose down -v`

## Sigurnosni elementi

- Multi-stage Docker build i non-root runtime korisnik
- Secret + ConfigMap odvojena konfiguracija
- Liveness/Readiness probe
- Resource requests/limits
- ServiceAccount + RBAC
- NetworkPolicy segmentacija
- Trivy skeniranje slika u CI pipelineu

Detalji skeniranja: `docs/security/image-scan-report.md`
