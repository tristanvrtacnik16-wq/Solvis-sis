# Šolski informacijski sistem (SIS)

Celotno razvojno okolje in aplikacija za teden 3–4: Docker okolje,
podatkovna baza s testnimi podatki in aplikacija s tremi vlogami
(administrator, učitelj, učenec).

## Zagon (en ukaz, na kateremkoli računalniku skupine)

```bash
cp .env.example .env
docker compose up
```

Po nekaj sekundah (ob prvem zagonu baza inicializira shemo, aplikacija pa
počaka nanjo in samodejno napolni testne podatke):

- **Aplikacija**: http://localhost:8080
- **Adminer** (orodje za administracijo baze): http://localhost:8081
  - Sistem: MySQL, Strežnik: `db`, Uporabnik/geslo/baza: glej `.env`
- **MySQL** je dosegljiv tudi neposredno na `localhost:3306` (za orodja kot DBeaver/MySQL Workbench)

### Testni računi (ustvarjeni samodejno ob prvem zagonu)

| Vloga | E-pošta | Geslo |
|---|---|---|
| Administrator | admin@sola.si | admin123 |
| Učitelj | poljubna e-pošta iz tabele `ucitelji` (glej Adminer) | ucitelj123 |
| Učenec | registrira se sam prek `/register.php` | po izbiri |

## Arhitektura / struktura docker-compose.yml

Tri ločene storitve v skupnem Docker omrežju (`solvis-net`):

- **db** – MySQL 8.0 v lastnem vsebniku, s trajnim volumnom `db_data`
  (podatki preživijo ponovni zagon). Ob prvem zagonu samodejno izvede
  `sql/schema.sql`.
- **app** – PHP 8.2 + Apache (lastna slika, `docker/app/Dockerfile`),
  koda je montirana iz `./src`, kar omogoča takojšnje spreminjanje brez
  ponovne gradnje slike. Ob zagonu počaka na bazo in požene `seed.php`.
- **adminer** – lahko orodje za pregled/urejanje baze neposredno prek
  brskalnika.

Aplikacija se z bazo povezuje prek imena storitve `db` (Docker DNS znotraj
skupnega omrežja), ne prek javnega IP-ja – zato je baza ločena od
aplikacije, a dosegljiva samo znotraj Docker omrežja (in dodatno izpostavljena
na `localhost:3306` zaradi razvojnega udobja).

## Struktura projekta

```
solvis-sis/
├── docker-compose.yml
├── .env.example
├── GIT-WORKFLOW.md
├── docker/app/            # Dockerfile + entrypoint za aplikacijski vsebnik
├── sql/schema.sql         # Normalizirana shema baze (glej komentarje v datoteki)
└── src/                   # Izvorna koda aplikacije (mount v vsebnik "app")
    ├── config.php         # Povezava z bazo, seje, avtorizacija
    ├── seed.php           # Testni podatki (10 predmetov, 20 učiteljev, 100 učencev)
    ├── login.php / register.php / logout.php / index.php
    ├── download.php       # Nadzorovan prenos gradiv/oddaj
    ├── admin/             # Administratorski pogled (CRUD + relacije)
    ├── api/                # REST končne točke za admin CRUD (JSON)
    ├── teacher/            # Učiteljski pogled (gradiva, oddaje učencev)
    └── student/            # Učenčev pogled (profil, predmeti, oddaja nalog)
```

## Funkcionalnost po vlogah

**Administrator** – vpis/urejanje/brisanje predmetov, učiteljev in učencev;
določanje relacij učitelj↔predmet in učenec↔predmet prek uporabniškega
vmesnika (`/admin`).

**Učitelj** – vidi seznam predmetov, ki jih poučuje; nalaga in briše
gradiva samo za te predmete; pregleduje oddane naloge učencev pri teh
predmetih (`/teacher`).

**Učenec** – registracija in urejanje profila; določanje/urejanje seznama
predmetov (vsaj 2); vpogled v gradiva vsakega vpisanega predmeta; oddaja
nalog – datoteka se shrani pod imenom `Priimek Ime – Naslov naloge`,
ponovna oddaja z istim naslovom prepiše prejšnjo datoteko šele po izrecni
potrditvi (`/student`).

## Normalizacija baze (kratek povzetek)

Shema je normalizirana do 3NF: vsaka entiteta (predmet, učitelj, učenec,
uporabniški račun, gradivo, oddaja) ima svojo tabelo, many-to-many relacije
(učitelj↔predmet, učenec↔predmet) so razvezane v ločenih veznih tabelah s
tujimi ključi in `UNIQUE` omejitvijo na paru ključev, kar prepreči podvojene
povezave. Podrobni komentarji so v `sql/schema.sql`.

## Git delovni tok

Glej `GIT-WORKFLOW.md` za vejno strategijo (`main` / `develop` /
`feature/...`) in pravila za vlečne zahtevke.
