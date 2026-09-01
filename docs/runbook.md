# Runbook

Dokument sadrži postupke za dijagnostiku i otklanjanje najčešćih kvarova.

Zbog bržeg snalaženja, preporuča se postaviti kontekst na namespace `ticketing`

```
kubectl config set-context --current --namespace=ticketing
```

## Predkoraci

Osnovna provjera logova i stanja

1. `kubectl get pods` - provjeriti koji pod nije Running ili Ready
2. `kubectl describe pod <naziv_poda>` - **Events** odjeljak prikazuje problem
3. `kubectl logs <ime> --previous` — zašto je prethodni kontejner ugašen
4. `kubectl get events --sort-by=.lastTimestamp` — kada nije jasno koji je pod problematičan

`describe` objašnjava zašto se pod ne pokreće, `logs` zašto se ruši nakon pokretanja.

---

## Tablica pokrivenih simptoma

| Simptom | Scenarij |
|---|---|
| CrashLoopBackOff na api ili workeru | S1 |
| ImagePullBackOff / ErrImagePull | S2 |
| CreateContainerConfigError | S3 |
| /readyz vraća 503, lista događaja radi | S4 |
| Kupnja vraća 202, narudžbe nema u bazi | S5 |
| failed calling webhook pri kubectl apply | S6 |
| Brisanje poda workera traje ~10 s | S7 |

---

## Scenarij

### S1 - Api i worker u CrashLoop

Simptom. Podovi se restartaju u petlji i RESTARTS se povećava. Frontend će se učitati ali neće imati liste događaja

#### Dijagnostika

```
kubectl get pods
kubectl logs deploy/api --previous
kubectl get pods -l app=redis
```

#### Uzrok

Ukoliko se radi o novome klasteru, Redis treba nekoliko sekundi da se pokrene i počne prihvaćati konekcije. Ovo stanje se mora smiriti u nekoliko sekundi, ili je problem u NetworkPolicy.

#### Mjera

```
kubectl get pods -l app=redis                    # je li Redis Running
kubectl get networkpolicy                        # postoji li dopuštenje za Redis
kubectl exec deploy/api -- nc -z redis 6379      # prolazi li veza
```

Ako Redis radi a veza ne prolazi, usporediti oznake u politici `ticketing-allow-redis-from-app` s oznakama podova (`kubectl get pods --show-labels`). Oznake moraju 
biti iste!

**Napomena** Ovo ponašanje se smatra "normalni", zbog nedostka proba za stanje redis i worker servisa.

---

### S2 - ImagePullBackOff nakon deploya

Simptom. Novi pod ostaje u ImagePullBackOff ili ErrImagePull. Stari pod i dalje radi, aplikacija je dostupna.

#### Dijagnostika

```
kubectl describe pod <naziv_poda> | tail -20
```

U Events stoji `Failed to pull image ... manifest unknown` ili `unauthorized`.

#### Uzrok

Tag ne postoji, ili je paket u GHCR-u privatan. GHCR pakete objavljene kroz GITHUB_TOKEN postavlja kao privatne, pa ih klaster bez imagePullSecret ne može povući.

#### Mjera

```
docker logout ghcr.io
docker pull ghcr.io/kemilab/ticketing-api:v1.0.1
kubectl rollout undo deployment/api
```

Ako slika povuče bez prijave, paket je javan i greška je u tagu.

#### Validacija

```
kubectl rollout status deployment/api
curl http://localhost/api/healthz
```

**Prevencija.** Deploy koristi nepromjenjive sha- tagove. Rolling update ne gasi stari pod dok novi ne prođe readiness probu, pa loš tag ne ruši aplikaciju nego samo zaustavi rollout.

---

### S3 - CreateContainerConfigError

Simptom. Pod ne prelazi u Running. Logova nema jer se kontejner nikad nije pokrenuo.

#### Dijagnostika

```
kubectl describe pod <naziv_poda> | tail -20
kubectl get configmap,secret
```

U Events: `couldn't find key POSTGRES_PASSWORD in Secret` ili `secret not found`.

#### Uzrok

Kontejner traži ključ kojeg u ConfigMapu ili Secretu nema.

#### Mjera

```
cp k8s/02-secrets-example.yml k8s/02-secrets.yml
kubectl apply -f k8s/02-secrets.yml
```

#### Validacija

```
kubectl get pods
kubectl exec deploy/api -- env | grep POSTGRES_
```

---

### S4 - Baza nedostupna

Simptom. /healthz vraća ok, /readyz vraća 503. Lista događaja radi, /tickets/orders vraća grešku. Kupnja i dalje vraća 202.

#### Dijagnostika

```
kubectl get pods -l app=postgres
kubectl logs database-0
kubectl get pvc
```

#### Uzrok

Postgres nije spreman ili PVC nije povezan.


#### Mjera

```
kubectl describe pod database-0 | tail -20
kubectl get pvc                    
kubectl delete pod database-0
```
 PVC status mora biti **Bound**

#### Validacija

```
kubectl exec database-0 -- pg_isready -U ticketing_user -d ticketing
```

---

### S5 - Narudžba potvrđena ali je nema u bazi

Simptom. POST /tickets/purchase vraća 202 i orderId, ali narudžba se ne pojavljuje u /tickets/orders. Izvana sve izgleda zdravo.

#### Dijagnostika

```
kubectl exec deploy/redis -- redis-cli LLEN ticket_orders
kubectl logs deploy/worker --tail=50
kubectl get pods -l app=worker
```

Raste li LLEN, worker ne prazni red.

#### Uzrok

Dvije različite situacije.

Worker ne radi ili ne može do Redisa.

Korisnik je upisao enormno veliki broj ulaznica. Worker se nakon toga
ne može sam oporaviti i potrebo ga je ponovno podići.



#### Mjera

```
kubectl rollout restart deployment/worker
```

#### Validacija

```
curl http://localhost/api/tickets/orders
```

---

### S6 - failed calling webhook pri primjeni Ingressa

Simptom.

```
Internal error occurred: failed calling webhook "validate.nginx.ingress.kubernetes.io":
dial tcp 10.96.x.x:443: connect: connection refused
```

#### Dijagnostika

```
kubectl get pods -n ingress-nginx
kubectl get endpoints ingress-nginx-controller-admission -n ingress-nginx
```

#### Uzrok

Admission webhook ingress-nginxa nije dostupan.

#### Mjera

```
kubectl apply -f k8s/04-ingress-controller.yml
```

#### Validacija

```
kubectl get ingress
curl http://localhost/
```

## Oporavak i eskalacija

| Situacija | Postupak |
|---|---|
| Vraćanje na prethodnu verziju | `kubectl rollout undo deployment/<ime>` |
| Pregled povijesti verzija | `kubectl rollout history deployment/<ime>` |
| Podaci nakon brisanja StatefulSeta | PVC se ne briše automatski, podaci ostaju |
| Potpuni reset okruženja | `kubectl delete namespace ticketing` pa `kubectl apply -f k8s/` |
