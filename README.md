# URL Threat Checker

**Lucrare de licență — Universitatea Babeș-Bolyai, Cluj-Napoca, Facultatea de Stiinte Economice si Gestiune a Afacerilor**

## Sumar

Aplicație web pentru analiza hibridă a riscului URL-urilor, dezvoltată ca proiect de licență. Un utilizator autentificat introduce un URL, sistemul îl analizează cu un model de învățare automată antrenat local, verifică opțional reputația prin VirusTotal, salvează rezultatul în baza de date și afișează un raport detaliat în interfața web.

Aplicația include autentificare cu doi factori (TOTP, compatibilă cu Google Authenticator) și o integrare opțională cu Telegram, care permite primirea URL-urilor printr-un bot și răspunsul automat cu verdictul calculat.

Proiectul este conceput pentru rulare locală în cadrul prezentării lucrării de licență; integrarea cu un mediu de producție remote nu face parte din versiunea evaluată.

## Mod de funcționare

```text
Utilizatorul introduce un URL
  -> backend-ul extrage trăsături numerice din URL
  -> modelul Random Forest local prezice tipul URL-ului
  -> reguli euristice corectează fals-pozitivele evidente
  -> VirusTotal este interogat dacă este configurat
  -> verdictul final este salvat și afișat în dashboard
  -> rezultatele modelului local pot fi comparate cu datele de referință VirusTotal
```

## Stivă tehnologică

- FastAPI (Python) cu validare Pydantic v2
- SQLAlchemy 2.0
- SQLite pentru persistență locală
- scikit-learn (model Random Forest)
- API VirusTotal v3 pentru îmbogățire externă
- Next.js 16, React 19, TypeScript strict, Tailwind CSS
- Autentificare 2FA prin TOTP (Google Authenticator)
- Integrare opțională cu Telegram Bot API

## Rulare backend

```bash
cd backend
uv sync
uv run reset-demo
uv run uvicorn url_threat_checker.main:app --host 127.0.0.1 --port 8001 --reload
```

Credențiale implicite pentru dezvoltare locală:

```text
admin / admin123
```

Aceste credențiale sunt destinate exclusiv mediului local de dezvoltare. Pentru orice rulare în afara acestui context, parola implicită trebuie schimbată, iar secretul TOTP trebuie configurat conform documentației.

## Rulare frontend

```bash
cd frontend
pnpm install
BACKEND_INTERNAL_URL=http://127.0.0.1:8001 pnpm dev
```

Aplicația poate fi accesată la:

```text
http://localhost:3000
```

Browser-ul comunică cu API-ul prin rewrite-ul `/backend` definit în Next.js, astfel încât cookie-urile de autentificare rămân same-origin pe parcursul testării locale.

## Artefactul modelului

Repository-ul include `models/model_card.json`, dar nu include fișierul antrenat al modelului, deoarece artefactul este de dimensiune mare:

```text
models/url_classifier.skops
```

Pentru demonstrația completă, fișierul `.skops` distribuit separat trebuie plasat exact la calea de mai sus înainte de pornirea backend-ului. În lipsa acestui fișier, aplicația pornește, însă starea modelului este raportată ca `unavailable`.

## Resetarea datelor pentru demonstrație

Înainte de capturile de ecran sau de prezentare, datele pot fi resetate determinist:

```bash
cd backend
uv run reset-demo
```

`reset-demo` șterge rapoartele locale și cache-ul VirusTotal, apoi creează un set curat de exemple intenționate: sigure, suspecte, periculoase și pseudo-whitelist, fără apeluri live către VirusTotal.

`seed-demo` este o variantă aditivă; se recomandă `reset-demo` pentru un dashboard curat.

Pentru a afișa și comparația dintre modelul local și VirusTotal fără apeluri live:

```bash
uv run reset-demo --with-comparison
```

## Antrenarea modelului

Pentru reantrenare, setul de date este distribuit ca artefact privat:

```text
url-threat-checker-training-data.zip
```

Acesta se dezarhivează în `data/raw/`, după care se rulează:

```bash
mkdir -p data/raw
unzip ~/Downloads/url-threat-checker-training-data.zip -d data/raw

uv --project backend run train-model \
  --input data/raw/malicious_phish.csv \
  --processed data/processed/prepared_urls.csv \
  --model models/url_classifier.skops \
  --card models/model_card.json \
  --augmentation data/raw/curated_benign_trusted_urls.csv
```

Setul original Kaggle este păstrat nemodificat. La antrenare se adaugă `data/raw/curated_benign_trusted_urls.csv`, un set restrâns de corecții revizuite manual pentru URL-uri HTTPS de încredere. Această augmentare corectează dezechilibrul din date, în care URL-urile HTTPS legitime erau subreprezentate, păstrând totodată exemplele negative relevante (Google Forms, Google Sites, redirect-uri și alte suprafețe de abuz). Rândurile curate sunt ponderate în timpul antrenării, astfel încât corecția să fie vizibilă modelului fără a altera setul original.

Fișierul antrenat `models/url_classifier.skops` este tratat ca artefact separat din cauza dimensiunii și trebuie copiat în `models/` înainte de demonstrația completă. Pachetul sursă include `models/model_card.json`, însă nu și fișierul `.skops`.

## Configurarea VirusTotal

Aplicația funcționează și fără VirusTotal. Atunci când nu este configurată o cheie API, rapoartele afișează `not_configured`, iar verdictul rămâne determinat de modelul local și de regulile euristice.

Pentru o verificare live locală, cheia se exportă doar în shell-ul care pornește backend-ul:

```bash
export VIRUSTOTAL_API_KEY="..."
```

Cheile API nu trebuie versionate și nu trebuie scrise în fișierele sursă.

## Generarea pachetului de predare

Pentru o arhivă curată de predare:

```bash
cd backend
uv run create-submission-bundle
```

Pachetul exclude secretele locale, mediile virtuale, `node_modules`, output-ul Next.js build, bazele de date SQLite locale, CSV-urile procesate, cache-urile și artefactul `.skops`.

## Verificarea proiectului

```bash
cd backend
uv run pytest
uv run ruff check

cd ../frontend
pnpm lint
pnpm build
```

Automatizarea testării prin browser (Playwright) nu face parte din această lucrare; testarea interfeței este realizată manual, conform procedurii descrise în raportul de licență.
