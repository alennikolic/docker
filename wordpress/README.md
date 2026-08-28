<!-- ./wordpress/README.md -->

# WordPress (Docker Compose)

WordPress sa MariaDB bazom, uvecanim PHP limitima za upload i opcionim phpMyAdmin-om.

---

## Sadrzaj stack-a

| Servis | Image | Uloga |
|---|---|---|
| `wordpress` | `wordpress:6.7-php8.3-apache` | Sam WordPress (Apache + PHP) |
| `db` | `mariadb:11.4` | Baza podataka |
| `phpmyadmin` | `phpmyadmin:5-apache` | Web pristup bazi — **opciono** |

phpMyAdmin se ne pokrece podrazumevano. Ukljucen je u `tools` profil, pa se startuje samo kada ti zatreba.

---

## Preduslovi

- Docker Engine 20.10 ili noviji
- Docker Compose v2 (`docker compose`, bez crtice)
- Slobodni portovi na hostu: `8081` (sajt) i `8082` (phpMyAdmin, opciono)

---

## Instalacija

**1. Udji u folder**

```bash
cd wordpress
```

**2. Napravi `.env` fajl**

```bash
cp .env.example .env
nano .env
```

Promeni obe lozinke. Ako zelis drugaciji prefiks tabela od `wp_`, promeni ga **sada** — posle instalacije se ne moze menjati bez rucne intervencije u bazi.

**3. Pokreni stack**

```bash
docker compose up -d
```

**4. Prati start**

```bash
docker compose logs -f wordpress
```

Prvi start traje desetak sekundi jer se WordPress fajlovi raspakuju u `./data/wordpress`.

**5. Otvori sajt**

```
http://<ip-adresa-servera>:8081
```

Pojavice se instalacioni carobnjak: izbor jezika, naziv sajta, admin korisnik i lozinka. Za admin korisnicko ime izbegni `admin` — to je prvo sto boti probaju.

**6. Po potrebi pokreni phpMyAdmin**

```bash
docker compose --profile tools up -d
```

Dostupan je na `http://<ip-adresa-servera>:8082`, prijava kredencijalima iz `.env` fajla. Kada zavrsis, ugasi ga:

```bash
docker compose stop phpmyadmin
```

---

## Promenljive iz `.env`

| Promenljiva | Podrazumevano | Opis |
|---|---|---|
| `WP_VERSION` | `6.7-php8.3-apache` | Tag WordPress slike sa fiksiranom PHP verzijom |
| `MARIADB_VERSION` | `11.4` | Tag MariaDB slike (LTS grana) |
| `WP_DB_NAME` | `wordpress` | Ime baze |
| `WP_DB_USER` | `wordpress` | Korisnik baze |
| `WP_DB_PASSWORD` | - | Lozinka tog korisnika |
| `WP_DB_ROOT_PASSWORD` | - | Root lozinka baze |
| `WP_TABLE_PREFIX` | `wp_` | Prefiks tabela, postavlja se pre prve instalacije |
| `WP_PORT` | `8081` | Port na hostu za sajt |
| `PMA_PORT` | `8082` | Port na hostu za phpMyAdmin |

Nakon izmene `.env` fajla uradi `docker compose up -d` da se kontejneri ponovo kreiraju.

---

## Struktura podataka

```
wordpress/
├── docker-compose.yml
├── .env
├── .env.example
├── README.md
├── config/
│   └── uploads.ini       # PHP limiti za upload i memoriju
└── data/
    ├── wordpress/        # kompletan sajt: wp-content, wp-config.php, jezgro
    └── db/               # fajlovi baze
```

Ceo sajt je u `./data/wordpress`, ukljucujuci teme, plugin-ove i upload-ovane fajlove. Backup tog foldera plus dump baze je kompletan backup sajta.

---

## PHP limiti

Podrazumevani PHP limit za upload je 2 MB, sto je premalo za vecinu tema. Fajl `config/uploads.ini` ga podize na 256 MB i uvecava memoriju i vreme izvrsavanja.

Nakon izmene tog fajla restartuj kontejner:

```bash
docker compose restart wordpress
```

Trenutne vrednosti proveri u **Alati -> Zdravlje sajta -> Info -> Server**.

---

## Svakodnevne komande

```bash
# Status kontejnera
docker compose ps

# Logovi
docker compose logs -f wordpress

# Restart
docker compose restart

# Zaustavljanje (podaci ostaju)
docker compose stop

# Zaustavljanje i brisanje kontejnera (podaci u ./data ostaju)
docker compose down

# Shell u kontejneru
docker compose exec wordpress bash

# Nadogradnja (prvo promeni WP_VERSION u .env)
docker compose pull
docker compose up -d
```

---

## WP-CLI

WordPress slika nema ugradjen WP-CLI, ali moze da se pokrene kao poseban kontejner nad istim podacima:

```bash
docker run --rm \
  --network wordpress_wordpress-net \
  --volumes-from wordpress \
  -u 33:33 \
  wordpress:cli wp plugin list
```

Korisni primeri:

```bash
# Lista i azuriranje plugin-ova
wp plugin list
wp plugin update --all

# Promena admin lozinke
wp user update admin --user_pass=nova_lozinka

# Zamena URL-a posle preseljenja sajta
wp search-replace 'http://staro.rs' 'https://novo.rs' --skip-columns=guid

# Ciscenje revizija postova
wp post delete $(wp post list --post_type=revision --format=ids) --force
```

Zameni samo deo posle `wordpress:cli` u komandi iznad.

---

## Backup i restore

**Backup (baza + fajlovi):**

```bash
docker compose exec db \
  mariadb-dump -u root -p"LOZINKA" --single-transaction wordpress \
  > backup-db-$(date +%F).sql

tar czf backup-files-$(date +%F).tar.gz ./data/wordpress
```

**Restore:**

```bash
cat backup-db-2026-01-15.sql | docker compose exec -T db \
  mariadb -u root -p"LOZINKA" wordpress

tar xzf backup-files-2026-01-15.tar.gz
docker compose restart
```

Backup baze bez fajlova je nepotpun — teme, plugin-ovi i slike nisu u bazi.

---

## Rad iza reverse proxy-ja

Ako WordPress stavis iza HAProxy-ja ili Nginx-a sa HTTPS sertifikatom, WordPress ce i dalje misliti da radi na HTTP-u i generisati mesovite (mixed content) linkove. Resenje je da mu prosledis informaciju iz `X-Forwarded-Proto` zaglavlja.

U `docker-compose.yml` otkomentarisi `WORDPRESS_CONFIG_EXTRA` blok, pa:

```bash
docker compose up -d
```

Zatim u **Podesavanja -> Opsta** promeni obe adrese sajta u `https://tvoj-domen.rs`.

Da bi proxy iz drugog foldera video ovaj kontejner, poveži ga na `wordpress_wordpress-net` mrezu (postupak je opisan u `haproxy/README.md`), i u backend sekciji koristi `server wp1 wordpress:80 check`.

---

## Resavanje problema

**"Error establishing a database connection"**

Baza jos nije spremna ili se lozinke ne poklapaju. MariaDB pamti lozinku iz prvog starta u `./data/db`; naknadna izmena `.env` fajla je nece promeniti. Proveri log:

```bash
docker compose logs db
```

**Bela stranica bez ikakve poruke**

Skoro uvek plugin ili tema. Ukljuci prikaz gresaka privremeno kroz `WORDPRESS_CONFIG_EXTRA`:

```yaml
      WORDPRESS_CONFIG_EXTRA: |
        define('WP_DEBUG', true);
        define('WP_DEBUG_LOG', true);
```

Log se onda pise u `./data/wordpress/wp-content/debug.log`. Iskljuci ovo kada zavrsis.

**WordPress trazi FTP podatke pri instalaciji plugin-a**

Problem sa dozvolama nad `./data/wordpress`. Fajlovi moraju da pripadaju korisniku pod kojim radi Apache (UID 33):

```bash
sudo chown -R 33:33 ./data/wordpress
```

**Permalink-ovi vracaju 404**

U **Podesavanja -> Stalne veze** samo klikni Sacuvaj — WordPress regenerise `.htaccess`. Apache u zvanicnoj slici vec ima ukljucen `mod_rewrite`.

**Upload veci od 2 MB ne prolazi**

`config/uploads.ini` nije montiran ili nije ucitan. Proveri:

```bash
docker compose exec wordpress php -i | grep upload_max_filesize
```

**Port 8081 je zauzet**

Promeni `WP_PORT` u `.env` i pokreni `docker compose up -d`.

**Zelim cist start (brise sajt i bazu)**

```bash
docker compose down
sudo rm -rf ./data
docker compose up -d
```

---

## Bezbednosne preporuke

- Admin korisnicko ime ne sme biti `admin`, `wordpress` ili ime domena.
- Ukljuci automatska azuriranja jezgra i redovno azuriraj plugin-ove — zastareo plugin je najcesci nacin provale u WordPress.
- Obrisi teme i plugin-ove koje ne koristis; i neaktivni mogu biti ranjivi.
- Stavi sajt iza reverse proxy-ja sa HTTPS-om pre nego sto ga izlozis internetu.
- Ogranici pristup `wp-login.php` po IP adresi ili dodaj plugin za rate limiting.

---

## Korisni linkovi

- [Zvanicna WordPress dokumentacija](https://wordpress.org/documentation/)
- [WordPress na Docker Hub-u](https://hub.docker.com/_/wordpress)
- [WP-CLI komande](https://developer.wordpress.org/cli/commands/)
