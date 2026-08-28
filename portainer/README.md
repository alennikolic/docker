<!-- ./portainer/README.md -->

# Portainer CE (Docker Compose)

Portainer je web interfejs za upravljanje Docker okruzenjem — kontejneri, slike, mreze, volumeni, stack-ovi, logovi i konzola, sve iz browsera.

---

## Sadrzaj stack-a

| Servis | Image | Uloga |
|---|---|---|
| `portainer` | `portainer/portainer-ce` | Web interfejs i API za upravljanje Docker-om |

Portainer upravlja lokalnim Docker-om preko `/var/run/docker.sock` koji mu je montiran u kontejner.

---

## Bezbednosno upozorenje

Montiranje Docker socket-a daje kontejneru punu kontrolu nad Docker daemon-om, sto je prakticno root pristup hostu. Zato:

- Nikada ne izlazi Portainer direktno na internet bez reverse proxy-ja sa autentifikacijom.
- Koristi jaku admin lozinku (minimum 12 karaktera).
- Ako ti treba samo pregled bez izmena, razmisli o read-only socket proxy-ju (`tecnativa/docker-socket-proxy`).

---

## Preduslovi

- Docker Engine 20.10 ili noviji
- Docker Compose v2 (`docker compose`, bez crtice)
- Slobodni portovi na hostu: `9443` (web) i `8000` (edge agent)

---

## Instalacija

**1. Udji u folder**

```bash
cd portainer
```

**2. Napravi `.env` fajl**

```bash
cp .env.example .env
```

Po potrebi izmeni verziju ili portove:

```bash
nano .env
```

**3. Pokreni stack**

```bash
docker compose up -d
```

**4. Otvori frontend**

```
https://<ip-adresa-servera>:9443
```

Obrati paznju na **https**, ne http. Portainer koristi self-signed sertifikat pa ce browser prikazati upozorenje — prihvati ga.

**5. Napravi admin nalog**

Pri prvom otvaranju Portainer trazi da postavis admin korisnika i lozinku. Ovo mora da se uradi u roku od 5 minuta od pokretanja kontejnera, inace se iz bezbednosnih razloga inicijalizacija zakljucava. Ako propustis rok:

```bash
docker compose restart
```

Zatim odmah otvori stranicu ponovo.

Nakon prijave izaberi **Get Started** da bi Portainer preuzeo lokalno Docker okruzenje.

---

## Promenljive iz `.env`

| Promenljiva | Podrazumevano | Opis |
|---|---|---|
| `PORTAINER_VERSION` | `2.21.4` | Tag verzije. Moze i `latest`, ali fiksna verzija je sigurnija |
| `PORTAINER_HTTPS_PORT` | `9443` | Port na hostu za web interfejs |
| `PORTAINER_EDGE_PORT` | `8000` | Port za tunel ka Edge agentima |

Nakon izmene `.env` fajla uradi `docker compose up -d` da se kontejner ponovo kreira.

---

## Struktura podataka

```
portainer/
├── docker-compose.yml
├── .env
├── .env.example
├── README.md
└── data/
    └── portainer/        # baza, sertifikati, podesavanja
```

Folder `data/` se kreira automatski pri prvom pokretanju. Ceo Portainer state je u njemu — backup tog foldera je backup cele instalacije.

---

## Svakodnevne komande

```bash
# Status kontejnera
docker compose ps

# Logovi
docker compose logs -f

# Restart
docker compose restart

# Zaustavljanje (podaci ostaju)
docker compose stop

# Zaustavljanje i brisanje kontejnera (podaci u ./data ostaju)
docker compose down

# Nadogradnja verzije
# 1. promeni PORTAINER_VERSION u .env
# 2. zatim:
docker compose pull
docker compose up -d
```

---

## Backup i restore

**Backup:**

```bash
docker compose stop
tar czf portainer-backup-$(date +%F).tar.gz ./data
docker compose start
```

**Restore:**

```bash
docker compose down
rm -rf ./data
tar xzf portainer-backup-2026-01-15.tar.gz
docker compose up -d
```

Portainer ima i ugradjeni backup: **Settings -> Backup Portainer**, sa opcijom enkripcije lozinkom.

---

## Upravljanje udaljenim serverima

Portainer moze da upravlja i drugim masinama. Na udaljenom serveru pokreni agenta:

```bash
docker run -d \
  -p 9001:9001 \
  --name portainer_agent \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:2.21.4
```

Zatim u Portainer-u: **Environments -> Add environment -> Docker Standalone -> Agent**, i upisi `<ip-udaljenog-servera>:9001`.

Za masine koje nisu direktno dostupne (iza NAT-a) koristi **Edge Agent**, koji se sam povezuje nazad na port `8000`.

---

## Resavanje problema

**Browser prijavljuje da veza nije bezbedna**

Ocekivano — self-signed sertifikat. Prihvati upozorenje ili postavi svoj sertifikat kroz **Settings -> SSL certificate**.

**"Your Portainer instance timed out for security purposes"**

Proslo je vise od 5 minuta od starta a admin nalog nije kreiran. Uradi `docker compose restart` i odmah otvori stranicu.

**Zaboravljena admin lozinka**

```bash
docker compose stop
docker run --rm -v $(pwd)/data/portainer:/data portainer/helper-reset-password
docker compose start
```

Komanda ispise privremenu lozinku u terminalu.

**Portainer ne vidi kontejnere**

Proveri da li je socket montiran i da li korisnik ima pristup:

```bash
docker compose exec portainer ls -l /var/run/docker.sock
```

**Port 9443 je zauzet**

Promeni `PORTAINER_HTTPS_PORT` u `.env` i pokreni `docker compose up -d`.

---

## Korisni linkovi

- [Zvanicna Portainer dokumentacija](https://docs.portainer.io/)
- [Portainer CE na Docker Hub-u](https://hub.docker.com/r/portainer/portainer-ce)
