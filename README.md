<!-- ./README.md -->

# Docker Stacks

Kolekcija spremnih Docker Compose stack-ova za servise koje najcesce koristim. Svaki servis ima svoj folder sa `docker-compose.yml`, `.env.example` i `README.md` sa uputstvom.

Cilj je da instalacija bilo kog servisa bude tri komande:

```bash
cd <servis>
cp .env.example .env
docker compose up -d
```

---

## Sadrzaj repozitorijuma

| Folder | Servis | Opis |
|---|---|---|
| [`zabbix 6.0`](./zabbix%206.0) | Zabbix 6.0 LTS | Monitoring sistem — server, MySQL, frontend, Java gateway, agent |
| [`zabbix 7.0`](./zabbix%207.0) | Zabbix 7.0 LTS | Novija LTS verzija, dodatno sa web service-om za PDF izvestaje |
| [`portainer`](./portainer) | Portainer CE | Web interfejs za upravljanje Docker okruzenjem |
| [`haproxy`](./haproxy) | HAProxy | Load balancer i reverse proxy |

Lista se dopunjuje. U planu: MySQL, PostgreSQL, Grafana, Nginx Proxy Manager, Uptime Kuma, Vaultwarden.

> **Napomena o imenima foldera**
> Dva Zabbix foldera imaju razmak u imenu (`zabbix 6.0`, `zabbix 7.0`). Zbog toga ih u shell-u uvek navodi pod navodnicima:
> ```bash
> cd "zabbix 7.0"
> ```
> Bez navodnika shell ce to protumaciti kao dva odvojena argumenta i komanda nece raditi. Isto vazi za `cp`, `tar`, `rm` i sve ostalo sto prima putanju.

---

## Instalacija Docker-a

### Ubuntu / Debian

**1. Ukloni stare pakete**

```bash
for pkg in docker.io docker-doc docker-compose podman-docker containerd runc; do
  sudo apt-get remove -y $pkg
done
```

**2. Dodaj zvanicni Docker repozitorijum**

```bash
sudo apt-get update
sudo apt-get install -y ca-certificates curl gnupg

sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update
```

> Za **Debian** zameni `ubuntu` sa `debian` na oba mesta u komandama iznad.

**3. Instaliraj Docker**

```bash
sudo apt-get install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
```

### Rocky Linux / AlmaLinux / RHEL / CentOS

```bash
sudo dnf remove -y docker docker-client docker-common docker-engine podman runc
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo dnf install -y docker-ce docker-ce-cli containerd.io \
  docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
```

### Windows i macOS

Preuzmi **Docker Desktop** sa [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop/). Compose je ukljucen u instalaciju.

Na Windows-u se preporucuje WSL2 backend. Fajlove iz ovog repo-a drzi unutar WSL2 fajl sistema (`/home/korisnik/...`), a ne na `C:\` disku — na montiranim Windows putanjama performanse su znatno losije, a dozvole na fajlovima prave probleme.

---

## Podesavanja posle instalacije

**Pokretanje Docker-a pri startu sistema**

```bash
sudo systemctl enable --now docker
```

**Rad bez `sudo`**

```bash
sudo usermod -aG docker $USER
```

Odjavi se i prijavi ponovo (ili `newgrp docker`) da bi promena vazila.

> Clanstvo u `docker` grupi je ekvivalent root pristupu na toj masini. Dodaj samo korisnike kojima verujes.

**Rotacija logova**

Podrazumevani `json-file` log driver raste bez ogranicenja dok ne popuni disk. Ovo je najcesci uzrok kvara na masinama koje rade mesecima bez nadzora. Postavi globalni limit u `/etc/docker/daemon.json`:

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

Zatim `sudo systemctl restart docker`. Limit vazi za kontejnere kreirane posle restarta — postojece treba ponovo kreirati sa `docker compose up -d --force-recreate`.

**Provera da sve radi**

```bash
docker --version
docker compose version
docker run --rm hello-world
```

Ako poslednja komanda ispise pozdravnu poruku, instalacija je uspesna.

> Vazno: koristi se `docker compose` (bez crtice), to je Compose v2 kao plugin. Stara `docker-compose` komanda sa crticom je v1 i vise se ne odrzava. Ako ti sistem prijavi da komanda ne postoji, nedostaje ti paket `docker-compose-plugin`.

---

## Kako se koristi ovaj repozitorijum

**1. Kloniraj repo**

```bash
git clone https://github.com/alennikolic/docker.git
cd docker
```

**2. Udji u folder servisa koji ti treba**

```bash
cd portainer
```

Za Zabbix foldere koristi navodnike:

```bash
cd "zabbix 7.0"
```

**3. Napravi `.env` fajl i popuni ga**

```bash
cp .env.example .env
nano .env
```

Svaki folder ima `.env.example` sa placeholder vrednostima. Lozinke tipa `promeni_me_...` obavezno zameni pravim.

**4. Pokreni**

```bash
docker compose up -d
```

**5. Procitaj README tog foldera**

U njemu su portovi, podrazumevani kredencijali, backup procedura i resavanje najcescih problema. Svaki servis ima svoje specificnosti.

---

## Konvencije u repozitorijumu

Svi folderi prate ista pravila, tako da kada naucis jedan, znas i ostale:

- **`docker-compose.yml`** — standardno ime, pa `docker compose up -d` radi bez dodatnih parametara.
- **`.env.example`** — sablon sa promenljivim vrednostima. Kopira se u `.env` koji Compose automatski ucitava.
- **`name:`** na vrhu compose fajla — fiksira ime projekta nezavisno od imena foldera. Ovde nije stvar kozmetike: ime projekta se inace izvodi iz imena foldera, a imena foldera sa razmakom i tackom Compose mora da sanitizuje. Bez `name:` linije ne znas unapred kako ce se zvati mreza ni kontejneri. Sa njom je deterministicki.
- **`./data/`** — svi trajni podaci idu tu kao bind mount, pa je backup celog servisa prosto arhiviranje tog foldera. Kreira se automatski pri prvom pokretanju.
- **Portovi kroz promenljive** — nijedan port nije zakucan u compose fajlu, sve se menja iz `.env`.
- **Prva linija svakog fajla je komentar sa njegovom putanjom** u repozitorijumu.
- **`.env` i `data/` nikada ne idu u git** — pokriveni su `.gitignore` fajlom u root-u.

---

## Imena Docker mreza

Kada jedan stack treba da vidi kontejnere iz drugog (npr. HAProxy koji prosledjuje saobracaj ka Zabbix frontendu), potrebno ti je tacno ime mreze. Ono se sastavlja kao `<name>_<mreza>`, gde `<name>` dolazi iz `name:` linije compose fajla:

| Stack | `name:` | Ime mreze |
|---|---|---|
| Zabbix 6.0 | `zabbix-60` | `zabbix-60_zabbix-net` |
| Zabbix 7.0 | `zabbix-70` | `zabbix-70_zabbix-net` |
| Portainer | `portainer` | `portainer_portainer-net` |
| HAProxy | `haproxy` | `haproxy_haproxy-net` |

Uvek potvrdi stvarno stanje sa:

```bash
docker network ls
```

---

## Osnovne Docker Compose komande

Sve se pokrecu iz foldera u kome se nalazi `docker-compose.yml`.

```bash
docker compose up -d           # pokreni u pozadini
docker compose ps              # status kontejnera
docker compose logs -f         # prati logove
docker compose logs -f <servis> # logovi jednog servisa
docker compose restart         # restart
docker compose stop            # zaustavi (kontejneri ostaju)
docker compose down            # zaustavi i obrisi kontejnere
docker compose pull            # povuci novije verzije slika
docker compose exec <servis> sh # shell u kontejneru
docker compose config          # prikazi konacnu konfiguraciju sa uvrscenim .env
```

Korisne opste komande:

```bash
docker ps -a                   # svi kontejneri na masini
docker stats                   # potrosnja resursa uzivo
docker network ls              # spisak mreza
docker system df               # koliko prostora zauzima Docker
docker system prune -a         # ciscenje neiskoriscenih slika i kontejnera
```

> `docker system prune -a` brise sve slike koje trenutno ne koristi nijedan kontejner. Zaustavljeni stack-ovi ce posle toga morati ponovo da povuku slike.

---

## Portovi po servisima

Da se ne bi sudarali kada vise stack-ova radi na istoj masini:

| Servis | Podrazumevani portovi | Promenljive |
|---|---|---|
| Zabbix 6.0 | `8080` (web), `10051` (server) | `ZBX_WEB_PORT`, `ZBX_SERVER_PORT` |
| Zabbix 7.0 | `8090` (web), `10061` (server) | `ZBX_WEB_PORT`, `ZBX_SERVER_PORT` |
| Portainer | `9443` (HTTPS web), `8000` (edge agent) | `PORTAINER_HTTPS_PORT`, `PORTAINER_EDGE_PORT` |
| HAProxy | `80`, `443`, `8404` (stats) | `HAPROXY_HTTP_PORT`, `HAPROXY_HTTPS_PORT`, `HAPROXY_STATS_PORT` |

Svi se menjaju kroz `.env` fajl odgovarajuceg foldera.

---

## Resavanje opstih problema

**`permission denied while trying to connect to the Docker daemon socket`**

Korisnik nije u `docker` grupi. Uradi `sudo usermod -aG docker $USER`, pa se odjavi i prijavi.

**`docker: 'compose' is not a docker command`**

Nedostaje Compose v2 plugin. Instaliraj `docker-compose-plugin` iz zvanicnog Docker repozitorijuma.

**`bind: address already in use`**

Neki drugi proces drzi taj port. Pronadji ga sa `sudo ss -tlnp | grep :<port>` pa ga zaustavi ili promeni port u `.env` fajlu.

**`no such file or directory` pri ulasku u Zabbix folder**

Ime foldera sadrzi razmak. Koristi `cd "zabbix 7.0"` sa navodnicima.

**Kontejner se stalno restartuje**

```bash
docker compose logs --tail=50 <servis>
```

Log skoro uvek sadrzi tacan razlog — najcesce greska u konfiguraciji ili nedostajuca promenljiva iz `.env`.

**Promene u `.env` nemaju efekta**

`docker compose restart` ne cita `.env` ponovo. Koristi:

```bash
docker compose up -d --force-recreate
```

**Kontejner ne moze da pise u `./data/` podfolder**

Docker kreira nedostajuce bind mount foldere kao `root:root`, a procesi u Zabbix kontejnerima rade pod neprivilegovanim korisnikom. Ako u logu vidis `permission denied` na `export` ili `snmptraps`, ispravi vlasnistvo:

```bash
sudo chown -R 1997:1997 data/export data/snmptraps
```

**Nema slobodnog prostora na disku**

```bash
docker system df
docker system prune -a --volumes
```

---

## Pre izlaganja u produkciju

Ovaj repo je pisan kao skup radnih sablona. Pre nego sto bilo sta od ovoga stavi na masinu dostupnu spolja, prodji kroz listu:

**Obavezno**

- Zameni sve podrazumevane lozinke iz `.env.example`.
- Promeni podrazumevane admin kredencijale servisa (npr. Zabbix `Admin` / `zabbix`).
- Proveri da `.env` i `data/` nisu u gitu: `git ls-files | grep -E '(^|/)\.env$|/data/'`. Ako je nesto vec commitovano, samo `.gitignore` to nece ukloniti — mora `git rm --cached`, a lozinka ostaje u istoriji i treba je promeniti.
- Ne izlazi administrativne interfejse direktno. Stavi ih iza VPN-a ili reverse proxy-ja sa autentifikacijom, ili ih vezi na loopback:
  ```yaml
  ports:
    - "127.0.0.1:${PORTAINER_HTTPS_PORT}:9443"
  ```

**Preporuceno**

- Fiksiraj verzije slika umesto `latest` tagova, da nadogradnja bude svesna odluka a ne posledica `docker compose pull`.
- Dodaj `mem_limit` i `cpus` svakom servisu, da jedan odbegli proces ne izgladni ceo host.
- Dodaj healthcheck servisima koji ga nemaju. Bez njega `restart: unless-stopped` ne pomaze kada proces zivi ali ne odgovara.
- Za MySQL: `--skip-log-bin` znaci da nemas point-in-time recovery, samo dnevni dump. Ako gubitak podataka od jednog dana nije prihvatljiv, ukloni tu opciju i podesi binlog backup.
- Testiraj restore proceduru pre nego sto ti zatreba. Backup koji nikad nije vracen nije backup.

---

## Licenca

MIT
