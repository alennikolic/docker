<!-- ./haproxy/README.md -->

# HAProxy (Docker Compose)

HAProxy je load balancer i reverse proxy za HTTP i TCP saobracaj. Ovaj stack ga pokrece sa primerom konfiguracije koja radi rutiranje po domenu, health check-ove backend servera i stats stranicu.

---

## Sadrzaj stack-a

| Servis | Image | Uloga |
|---|---|---|
| `haproxy` | `haproxy:3.0-alpine` | Load balancer / reverse proxy |

---

## Preduslovi

- Docker Engine 20.10 ili noviji
- Docker Compose v2 (`docker compose`, bez crtice)
- Slobodni portovi na hostu: `80`, `443` i `8404` (stats)

Ako na hostu vec radi nginx ili Apache na portu 80, zaustavi ga ili promeni portove u `.env`.

---

## Instalacija

**1. Udji u folder**

```bash
cd haproxy
```

**2. Napravi `.env` fajl**

```bash
cp .env.example .env
nano .env
```

Obavezno promeni `HAPROXY_STATS_PASSWORD`.

**3. Prilagodi konfiguraciju**

Otvori `config/haproxy.cfg` i zameni primere svojim domenima i IP adresama backend servera:

```bash
nano config/haproxy.cfg
```

**4. Proveri sintaksu pre pokretanja**

```bash
docker run --rm -v $(pwd)/config:/usr/local/etc/haproxy:ro \
  -e HAPROXY_STATS_USER=test -e HAPROXY_STATS_PASSWORD=test \
  haproxy:3.0-alpine haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
```

Ocekivan odgovor je `Configuration file is valid`.

**5. Pokreni stack**

```bash
docker compose up -d
docker compose logs -f
```

**6. Otvori stats stranicu**

```
http://<ip-adresa-servera>:8404
```

Prijavi se kredencijalima iz `.env` fajla.

---

## Promenljive iz `.env`

| Promenljiva | Podrazumevano | Opis |
|---|---|---|
| `HAPROXY_VERSION` | `3.0-alpine` | Tag verzije |
| `HAPROXY_HTTP_PORT` | `80` | Port na hostu za HTTP |
| `HAPROXY_HTTPS_PORT` | `443` | Port na hostu za HTTPS |
| `HAPROXY_STATS_PORT` | `8404` | Port na hostu za stats stranicu |
| `HAPROXY_STATS_USER` | `admin` | Korisnik za stats stranicu |
| `HAPROXY_STATS_PASSWORD` | - | Lozinka za stats stranicu |

HAProxy podrzava `${PROMENLJIVA}` sintaksu direktno u `haproxy.cfg`, pa se vrednosti iz `.env` prosledjuju kontejneru kroz `environment` sekciju i koriste u konfiguraciji. Tako lozinka nikada ne stoji u samom config fajlu.

---

## Struktura foldera

```
haproxy/
├── docker-compose.yml
├── .env
├── .env.example
├── README.md
├── config/
│   └── haproxy.cfg       # glavna konfiguracija
└── certs/                # PEM sertifikati za HTTPS
```

Folder `certs/` napravi rucno kada ti zatreba HTTPS:

```bash
mkdir -p certs
```

---

## HTTPS i sertifikati

HAProxy ocekuje **jedan PEM fajl** koji sadrzi i sertifikat i privatni kljuc, spojene jedan za drugim.

Ako koristis Let's Encrypt:

```bash
cat /etc/letsencrypt/live/primer.rs/fullchain.pem \
    /etc/letsencrypt/live/primer.rs/privkey.pem \
    > certs/primer.rs.pem
chmod 600 certs/primer.rs.pem
```

Za test sa self-signed sertifikatom:

```bash
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout /tmp/key.pem -out /tmp/cert.pem \
  -subj "/CN=primer.rs"
cat /tmp/cert.pem /tmp/key.pem > certs/primer.rs.pem
```

Zatim otkomentarisi `https_in` frontend blok u `config/haproxy.cfg` i ponovo ucitaj konfiguraciju.

---

## Ponovno ucitavanje konfiguracije

Nakon izmene `haproxy.cfg` **nije potreban restart**. HAProxy podrzava graceful reload signalom, bez prekida postojecih konekcija:

```bash
docker compose kill -s HUP haproxy
```

Pre toga uvek proveri sintaksu:

```bash
docker compose exec haproxy haproxy -c -f /usr/local/etc/haproxy/haproxy.cfg
```

Ako je konfiguracija neispravna, HAProxy ce zadrzati staru i nastaviti da radi.

---

## Svakodnevne komande

```bash
# Status kontejnera
docker compose ps

# Logovi (pristupni log ide na stdout)
docker compose logs -f

# Reload konfiguracije bez prekida
docker compose kill -s HUP haproxy

# Restart
docker compose restart

# Zaustavljanje
docker compose stop

# Zaustavljanje i brisanje kontejnera
docker compose down

# Nadogradnja verzije (prvo promeni HAPROXY_VERSION u .env)
docker compose pull
docker compose up -d
```

---

## Rutiranje ka kontejnerima iz drugih stack-ova

Da bi HAProxy prosledjivao saobracaj ka npr. Zabbix web interfejsu iz susednog foldera, poveži ga na tu mrezu. U `docker-compose.yml` dodaj:

```yaml
    networks:
      - haproxy-net
      - zabbix-net

networks:
  haproxy-net:
    driver: bridge
  zabbix-net:
    external: true
    name: zabbix-70_zabbix-net
```

Zatim u `haproxy.cfg` koristi ime servisa umesto IP adrese:

```
backend zabbix_servers
    server zbx1 zabbix-web:8080 check
```

Tacno ime mreze proveri sa `docker network ls`.

---

## Balansiranje - dostupni algoritmi

| Algoritam | Kada koristiti |
|---|---|
| `roundrobin` | Podrazumevani, ravnomerno redom po serverima |
| `leastconn` | Kada konekcije traju dugo (baze, WebSocket) |
| `source` | Sticky sesije bazirane na IP adresi klijenta |
| `uri` | Keširanje - isti URL uvek ide na isti server |

Menja se linijom `balance <algoritam>` u backend sekciji.

---

## Resavanje problema

**Kontejner se ne pokrece**

Skoro uvek je greska u konfiguraciji. Pogledaj log:

```bash
docker compose logs haproxy
```

HAProxy prijavljuje tacan broj linije u kojoj je problem.

**Greska "cannot bind socket" na portu 80**

Ili je port zauzet na hostu (`sudo ss -tlnp | grep :80`), ili `sysctls` linija nije primenjena. Novije HAProxy slike rade kao non-root korisnik, pa im je potreban `net.ipv4.ip_unprivileged_port_start=0` da bi otvorile portove ispod 1024.

**Stats stranica trazi lozinku a ne prihvata je**

Proveri da su `HAPROXY_STATS_USER` i `HAPROXY_STATS_PASSWORD` prosledjeni kontejneru:

```bash
docker compose exec haproxy env | grep HAPROXY
```

Ako su prazni, kontejner je startovan pre nego sto je `.env` napravljen — uradi `docker compose up -d --force-recreate`.

**Svi backend serveri su DOWN**

Na stats stranici pogledaj kolonu sa razlogom. Najcesce je health check putanja pogresna (`http-check send meth GET uri /`) ili backend server ne odgovara sa statusom 200. Probaj privremeno da ukloniš `option httpchk` da bi potvrdio da je problem u proveri, a ne u mrezi.

**Backend server ne vidi pravu IP adresu klijenta**

`option forwardfor` je vec ukljucen i salje `X-Forwarded-For` zaglavlje. Aplikacija na backend serveru mora da bude podesena da ga cita.

---

## Korisni linkovi

- [Zvanicna HAProxy dokumentacija](https://docs.haproxy.org/)
- [HAProxy na Docker Hub-u](https://hub.docker.com/_/haproxy)
