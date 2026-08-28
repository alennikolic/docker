# Zabbix 7.0 LTS (Docker Compose)

Kompletan Zabbix 7.0 stack u Docker kontejnerima: server, MySQL baza, web frontend (nginx + PHP), Java gateway, web service za izvestaje i agent2.

---

## Sadrzaj stack-a

| Servis | Image | Uloga |
|---|---|---|
| `mysql-server` | `mysql:8.0-oracle` | Baza podataka |
| `zabbix-server` | `zabbix/zabbix-server-mysql:alpine-7.0-latest` | Jezgro sistema, prikuplja i obradjuje podatke |
| `zabbix-web` | `zabbix/zabbix-web-nginx-mysql:alpine-7.0-latest` | Web interfejs |
| `zabbix-java-gateway` | `zabbix/zabbix-java-gateway:alpine-7.0-latest` | JMX monitoring (Java aplikacije) |
| `zabbix-web-service` | `zabbix/zabbix-web-service:alpine-7.0-latest` | Generisanje PDF izvestaja i browser provere |
| `zabbix-agent` | `zabbix/zabbix-agent2:alpine-7.0-latest` | Monitoring samog host servera |

Svi kontejneri su na zajednickoj `zabbix-net` bridge mrezi i medjusobno komuniciraju preko imena servisa.

---

## Sta je novo u odnosu na 6.0

- **`zabbix-web-service`** je dodat u stack. Sluzi za planirane PDF izvestaje (*Reports -> Scheduled reports*) i za novi `browser` tip stavke koji izvrsava JavaScript scenarije preko headless Chromium-a. Zbog Chromium sandbox-a kontejneru je dodat `cap_add: SYS_ADMIN`.
- **`agent2` umesto klasicnog agenta.** Agent2 je pisan u Go-u, podrzava plugin sistem i persistent buffer za aktivne provere. Ako ti treba klasican agent, zameni image sa `zabbix/zabbix-agent:alpine-7.0-latest`.
- **`name: zabbix-70`** je definisan na vrhu compose fajla, a svi kontejneri imaju `zabbix70-` prefiks, tako da ovaj stack moze da radi paralelno sa 6.0 stack-om na istoj masini.
- **Drugi portovi po defaultu** (`8090` za web, `10061` za server) iz istog razloga.
- Proxy sada podrzava memorijski buffer, a serveru je dodata podrska za asinhrone poller-e — nista od toga ne zahteva izmenu u compose fajlu.

---

## Preduslovi

- Docker Engine 20.10 ili noviji
- Docker Compose v2 (`docker compose`, bez crtice)
- Minimum 2 GB RAM-a i 10 GB slobodnog prostora
- Slobodni portovi na hostu: `8090` (web) i `10061` (server)

Provera da li je sve na mestu:

```bash
docker --version
docker compose version
```

---

## Instalacija

**1. Udji u folder**

```bash
cd zabbix-7.0
```

**2. Napravi `.env` fajl**

```bash
cp .env.example .env
```

Otvori ga i promeni lozinke:

```bash
nano .env
```

**3. Pokreni stack**

```bash
docker compose up -d
```

**4. Prati start**

Prvi start traje 1-3 minuta jer se importuje sema baze. Sacekaj poruku da je server startovan:

```bash
docker compose logs -f zabbix-server
```

Kada vidis `Zabbix Server started`, sve je spremno. Prekini pracenje sa `Ctrl+C`.

**5. Otvori frontend**

```
http://<ip-adresa-servera>:8090
```

Pocetni podaci za prijavu:

- Korisnik: `Admin`
- Lozinka: `zabbix`

> Promeni lozinku odmah nakon prve prijave: **Users -> Users -> Admin -> Change password**.

---

## Promenljive iz `.env`

| Promenljiva | Podrazumevano | Opis |
|---|---|---|
| `MYSQL_DATABASE` | `zabbix` | Ime baze |
| `MYSQL_USER` | `zabbix` | Korisnik baze koji koristi Zabbix |
| `MYSQL_PASSWORD` | - | Lozinka tog korisnika |
| `MYSQL_ROOT_PASSWORD` | - | Root lozinka MySQL-a |
| `ZBX_WEB_PORT` | `8090` | Port na hostu za web interfejs |
| `ZBX_SERVER_PORT` | `10061` | Port na hostu za komunikaciju sa agentima |
| `ZBX_SERVER_NAME` | `Zabbix 7.0` | Naziv koji se prikazuje u frontendu |
| `PHP_TZ` | `Europe/Belgrade` | Vremenska zona frontenda |

Nakon izmene `.env` fajla obavezno uradi `docker compose up -d` da se kontejneri ponovo kreiraju sa novim vrednostima.

---

## Ukljucivanje PDF izvestaja

Web service je vec povezan sa serverom, ali frontend mora da zna svoju adresu:

1. Idi na **Administration -> General -> Other**.
2. U polje **Frontend URL** upisi `http://zabbix-web:8080/` (interno ime servisa, ne javni URL).
3. Sacuvaj i probaj **Reports -> Scheduled reports -> Create report -> Test**.

Ako test vrati gresku o konekciji, proveri da li se ime `zabbix-server` nalazi u `ZBX_ALLOWEDIP` promenljivoj web-service kontejnera.

---

## Struktura podataka

```
zabbix-7.0/
├── docker-compose.yml
├── .env
├── .env.example
├── README.md
└── data/
    ├── mysql/            # fajlovi baze
    ├── alertscripts/     # custom skripte za notifikacije
    ├── externalscripts/  # eksterne check skripte
    ├── export/           # export istorijskih podataka
    └── snmptraps/        # SNMP trap fajlovi
```

Folder `data/` se kreira automatski pri prvom pokretanju.

---

## Svakodnevne komande

```bash
# Status kontejnera
docker compose ps

# Logovi svih servisa
docker compose logs -f

# Logovi jednog servisa
docker compose logs -f zabbix-server

# Restart celog stack-a
docker compose restart

# Restart jednog servisa
docker compose restart zabbix-server

# Zaustavljanje (podaci ostaju)
docker compose stop

# Zaustavljanje i brisanje kontejnera (podaci u ./data ostaju)
docker compose down

# Ulazak u shell kontejnera
docker compose exec zabbix-server sh

# Azuriranje na najnoviji 7.0.x patch
docker compose pull
docker compose up -d
```

---

## Backup i restore

**Backup baze:**

```bash
docker compose exec mysql-server \
  mysqldump -u root -p"LOZINKA" --single-transaction zabbix \
  > backup-$(date +%F).sql
```

**Restore:**

```bash
cat backup-2026-01-15.sql | docker compose exec -T mysql-server \
  mysql -u root -p"LOZINKA" zabbix
```

---

## Dodavanje hosta za monitoring

Na masini koju zelis da monitorises instaliraj Zabbix agenta i u njegovoj konfiguraciji podesi:

```
Server=<ip-tvog-zabbix-servera>
ServerActive=<ip-tvog-zabbix-servera>:10061
Hostname=<ime-te-masine>
```

Obrati paznju na port `10061` u `ServerActive` — to je port koji je ovaj stack izlozio na hostu.

Zatim u frontendu: **Data collection -> Hosts -> Create host**, dodaj IP adresu i povezi template (npr. *Linux by Zabbix agent*).

---

## Resavanje problema

**Server se ne pokrece, u logu pise da ne moze da se poveze na bazu**

Sacekaj - MySQL healthcheck moze trajati do minut. Ako i dalje ne radi, proveri da li se lozinke u `.env` poklapaju sa onim sto je bilo pri prvom kreiranju baze. MySQL pamti lozinku iz prvog starta u `./data/mysql`; naknadna izmena `.env` fajla je nece promeniti.

**Frontend prijavljuje "Database error"**

Zabbix server jos uvek importuje semu. Sacekaj i osvezi stranicu.

**Web service pada ili izvestaji ne rade**

Chromium unutar kontejnera zahteva `SYS_ADMIN` capability. Ako tvoj host ima strog seccomp/AppArmor profil, proveri log sa `docker compose logs zabbix-web-service`.

**Port je zauzet**

Promeni `ZBX_WEB_PORT` ili `ZBX_SERVER_PORT` u `.env` i pokreni `docker compose up -d`.

**Zelim cist start (brise sve podatke)**

```bash
docker compose down
sudo rm -rf ./data
docker compose up -d
```

---

## Nadogradnja sa 6.0 na 7.0

Ovo je zaseban stack sa svojom bazom. Ako zelis da prebacis postojece podatke iz 6.0 instalacije:

1. Napravi backup baze iz 6.0 foldera.
2. Zaustavi 6.0 stack (`docker compose down`).
3. Restore-uj dump u bazu 7.0 stack-a **pre** nego sto pokrenes `zabbix-server` kontejner.
4. Pokreni stack — Zabbix server ce sam odraditi migraciju seme pri prvom startu i to ce upisati u log.

Migracija seme je jednosmerna. Zadrzi backup dok ne potvrdis da sve radi.

---

## Korisni linkovi

- [Zvanicna Zabbix 7.0 dokumentacija](https://www.zabbix.com/documentation/7.0/en)
- [Zabbix Docker images na Docker Hub-u](https://hub.docker.com/u/zabbix)
