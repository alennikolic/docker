# Zabbix 6.0 LTS (Docker Compose)

Kompletan Zabbix 6.0 stack u Docker kontejnerima: server, MySQL baza, web frontend (nginx + PHP), Java gateway i agent.

---

## Sadrzaj stack-a

| Servis | Image | Uloga |
|---|---|---|
| `mysql-server` | `mysql:8.0-oracle` | Baza podataka |
| `zabbix-server` | `zabbix/zabbix-server-mysql:alpine-6.0-latest` | Jezgro sistema, prikuplja i obradjuje podatke |
| `zabbix-web` | `zabbix/zabbix-web-nginx-mysql:alpine-6.0-latest` | Web interfejs |
| `zabbix-java-gateway` | `zabbix/zabbix-java-gateway:alpine-6.0-latest` | JMX monitoring (Java aplikacije) |
| `zabbix-agent` | `zabbix/zabbix-agent:alpine-6.0-latest` | Monitoring samog host servera |

Svi kontejneri su na zajednickoj `zabbix-net` bridge mrezi i medjusobno komuniciraju preko imena servisa.

---

## Preduslovi

- Docker Engine 20.10 ili noviji
- Docker Compose v2 (`docker compose`, bez crtice)
- Minimum 2 GB RAM-a i 10 GB slobodnog prostora
- Slobodni portovi na hostu: `8080` (web) i `10051` (server)

Provera da li je sve na mestu:

```bash
docker --version
docker compose version
```

---

## Instalacija

**1. Udji u folder**

```bash
cd zabbix
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

Prvi start traje 1-3 minuta jer se importuje sema baze (oko 170 tabela). Sacekaj poruku da je server startovan:

```bash
docker compose logs -f zabbix-server
```

Kada vidis `Zabbix Server started`, sve je spremno. Prekini pracenje sa `Ctrl+C`.

**5. Otvori frontend**

```
http://<ip-adresa-servera>:8080
```

Pocetni podaci za prijavu:

- Korisnik: `Admin`
- Lozinka: `zabbix`

> Promeni lozinku odmah nakon prve prijave: **User settings -> Profile -> Change password**.

---

## Promenljive iz `.env`

| Promenljiva | Podrazumevano | Opis |
|---|---|---|
| `MYSQL_DATABASE` | `zabbix` | Ime baze |
| `MYSQL_USER` | `zabbix` | Korisnik baze koji koristi Zabbix |
| `MYSQL_PASSWORD` | - | Lozinka tog korisnika |
| `MYSQL_ROOT_PASSWORD` | - | Root lozinka MySQL-a |
| `ZBX_WEB_PORT` | `8080` | Port na hostu za web interfejs |
| `ZBX_SERVER_NAME` | `Zabbix 6.0` | Naziv koji se prikazuje u frontendu |
| `PHP_TZ` | `Europe/Belgrade` | Vremenska zona frontenda |

Nakon izmene `.env` fajla obavezno uradi `docker compose up -d` da se kontejneri ponovo kreiraju sa novim vrednostima.

---

## Struktura podataka

```
zabbix/
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

# Azuriranje na najnoviji 6.0.x patch
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
ServerActive=<ip-tvog-zabbix-servera>
Hostname=<ime-te-masine>
```

Zatim u frontendu: **Data collection -> Hosts -> Create host**, dodaj IP adresu i povezi template (npr. *Linux by Zabbix agent*).

---

## Resavanje problema

**Server se ne pokrece, u logu pise da ne moze da se poveze na bazu**

Sacekaj - MySQL healthcheck moze trajati do minut. Ako i dalje ne radi, proveri da li se lozinke u `.env` poklapaju sa onim sto je bilo pri prvom kreiranju baze. MySQL pamti lozinku iz prvog starta u `./data/mysql`; naknadna izmena `.env` fajla je nece promeniti.

**Frontend prijavljuje "Database error"**

Zabbix server jos uvek importuje semu. Sacekaj i osvezi stranicu.

**Port 8080 je zauzet**

Promeni `ZBX_WEB_PORT` u `.env` na npr. `8081` i pokreni `docker compose up -d`.

**Zelim cist start (brise sve podatke)**

```bash
docker compose down
sudo rm -rf ./data
docker compose up -d
```

---

## Korisni linkovi

- [Zvanicna Zabbix 6.0 dokumentacija](https://www.zabbix.com/documentation/6.0/en)
- [Zabbix Docker images na Docker Hub-u](https://hub.docker.com/u/zabbix)
