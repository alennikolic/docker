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
| [`zabbix-6.0`](./zabbix-6.0) | Zabbix 6.0 LTS | Monitoring sistem — server, MySQL, frontend, Java gateway, agent |
| [`zabbix-7.0`](./zabbix-7.0) | Zabbix 7.0 LTS | Novija LTS verzija, dodatno sa web service-om za PDF izvestaje |
| [`portainer`](./portainer) | Portainer CE | Web interfejs za upravljanje Docker okruzenjem |
| [`haproxy`](./haproxy) | HAProxy | Load balancer i reverse proxy |

Lista se dopunjuje. U planu: MySQL, PostgreSQL, Grafana, Nginx Proxy Manager, Uptime Kuma, Vaultwarden.

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
cd <repo>
```

**2. Udji u folder servisa koji ti treba**

```bash
cd portainer
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
- **`name:`** na vrhu compose fajla — fiksira ime projekta nezavisno od imena foldera.
- **`./data/`** — svi trajni podaci idu tu kao bind mount, pa je backup celog servisa prosto arhiviranje tog foldera. Kreira se automatski pri prvom pokretanju.
- **Portovi kroz promenljive** — nijedan port nije zakucan u compose fajlu, sve se menja iz `.env`.
- **Prva linija svakog fajla je komentar sa njegovom putanjom** u repozitorijumu.

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

| Servis | Podrazumevani portovi |
|---|---|
| Zabbix 6.0 | `8080` (web), `10051` (server) |
| Zabbix 7.0 | `8090` (web), `10061` (server) |
| Portainer | `9443` (HTTPS web), `8000` (edge agent) |
| HAProxy | `80`, `443`, `8404` (stats) |

Svi se menjaju kroz `.env` fajl odgovarajuceg foldera.

---

## Resavanje opstih problema

**`permission denied while trying to connect to the Docker daemon socket`**

Korisnik nije u `docker` grupi. Uradi `sudo usermod -aG docker $USER`, pa se odjavi i prijavi.

**`docker: 'compose' is not a docker command`**

Nedostaje Compose v2 plugin. Instaliraj `docker-compose-plugin` iz zvanicnog Docker repozitorijuma.

**`bind: address already in use`**

Neki drugi proces drzi taj port. Pronadji ga sa `sudo ss -tlnp | grep :<port>` pa ga zaustavi ili promeni port u `.env` fajlu.

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

**Nema slobodnog prostora na disku**

```bash
docker system df
docker system prune -a --volumes
```

---

## Napomena o bezbednosti

Fajlovi `.env.example` u ovom repo-u sadrze placeholder lozinke koje su svima vidljive. Namenjeni su kao sablon, ne za produkciju.

Pre nego sto bilo sta od ovoga izlozis internetu:

- Zameni sve podrazumevane lozinke.
- Promeni podrazumevane admin kredencijale servisa (npr. Zabbix `Admin`/`zabbix`).
- Ne izlazi administrativne interfejse direktno — stavi ih iza VPN-a ili reverse proxy-ja sa autentifikacijom.
- Ako pravis sopstveni fork sa pravim lozinkama, dodaj `.gitignore` sa `.env` i `data/`.

---

## Licenca

MIT
