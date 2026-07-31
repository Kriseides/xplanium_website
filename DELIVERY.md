# XPlanium — Delivery del sito xplanium.it

Documento operativo per la pubblicazione e la manutenzione del sito pubblico
**xplanium.it**. Rivolto a chi metterà in produzione i file e configurerà DNS,
hosting e SMTP.

---

## 1. Cosa si consegna

Repository: `xplanium_website/` (branch `develop`).

Il sito è **completamente statico** (HTML + CSS + JS inline, zero build step).
I file da pubblicare sono:

```
xplanium_website/
├── index.html              ← home / landing principale
├── privacy.html            ← informativa privacy (GDPR)
├── cookie.html             ← cookie policy
├── accessibilita.html      ← dichiarazione di accessibilità (WCAG 2.1 AA)
├── favicon.svg             ← favicon vettoriale (browser moderni)
├── favicon.png             ← fallback favicon (Safari, apple-touch-icon)
├── sitemap.xml             ← sitemap per motori di ricerca
├── robots.txt              ← direttive crawler
└── assets/
    ├── logo-xplanium.svg
    ├── logo-kriseides.svg
    └── photos/             ← screenshot e foto reali (jpg/png)
```

**Non serve** un build step: si copia tutto così com'è sul server web.

**Da NON pubblicare**: la cartella `.git/`, questo `DELIVERY.md`, `README.md`,
e qualunque file `.zip` che dovesse rimanere in cartella.

---

## 2. Prerequisiti operativi

Prima del deploy servono:

1. **Dominio registrato**: `xplanium.it` — con accesso al pannello DNS del registrar
2. **Hosting statico**: DigitalOcean App Platform (Static Site), Netlify,
   Cloudflare Pages, o equivalente. Il sito è statico, qualunque CDN va bene.
   Scelta consigliata: **DigitalOcean App Platform (Static Site tier gratuito)**
   per allinearsi allo stack Kriseides già in uso.
3. **Certificato HTTPS**: fornito automaticamente da tutti gli hosting
   moderni (Let's Encrypt). Non serve acquistare nulla.
4. **Endpoint API per il form contatti**: vedi sezione 4.
5. **Casella email `info@xplanium.it`** attiva (destinataria delle richieste)
   e **`no-reply@xplanium.it`** configurabile come mittente su Brevo (SPF,
   DKIM, DMARC del dominio devono puntare a Brevo). Vedi sezione 4.

---

## 3. Pubblicazione del sito statico

### 3.1 DigitalOcean App Platform (raccomandato)

1. Dal pannello DO → Apps → Create App → **Static Site**
2. Sorgente: connetti il repository Git di `xplanium_website` (branch `main`
   o `develop`, allineare con il branch di produzione scelto)
3. **Build command**: lasciare vuoto (nessun build)
4. **Output directory**: `/` (root del repo)
5. **HTTP routes**: `/` (default) — App Platform gestisce automaticamente
   `sitemap.xml`, `robots.txt` e i redirect delle pagine `.html`
6. **Custom domain**: aggiungere `xplanium.it` e `www.xplanium.it`
7. DO chiederà di aggiungere record DNS (vedi 3.3)

### 3.2 Alternativa: Netlify / Cloudflare Pages

Stessa logica:
- Connetti il repo Git
- Build command: vuoto
- Publish directory: `.` (root)
- Custom domain: `xplanium.it`

### 3.3 DNS (in tutti i casi)

Al registrar del dominio `xplanium.it` configurare:

| Tipo   | Nome  | Valore                           | Note                         |
|--------|-------|----------------------------------|------------------------------|
| A      | @     | (IP fornito dall'hosting)        | Punta a root domain          |
| CNAME  | www   | xplanium.it                      | Redirect www → root          |

L'hosting fornirà l'IP esatto (o un CNAME target come
`ondigitalocean.app` / `netlify.app`).

**Propagazione**: fino a 48h. Di solito < 1h.

### 3.4 HTTPS

Automatico su tutti gli hosting moderni: al primo hit su HTTPS viene
generato un certificato Let's Encrypt. Verificare che l'hosting abbia
attivo il **redirect HTTP → HTTPS** (opzione standard, quasi sempre
attiva di default).

Test dopo la pubblicazione: aprire `http://xplanium.it` deve reindirizzare
a `https://xplanium.it`.

---

## 4. Endpoint del form contatti

Il form di richiesta dimostrazione (sezione `#contatti`) invia i dati con
`fetch()` a un endpoint HTTP configurato nel JS di `index.html`
(riga 4831, costante `CONTACT_API_URL`):

```
https://api.kriseides.com/api/lead/xplanium
```

**Il backend che serve questo endpoint esiste già** ed è il backend Fastify
condiviso del gruppo Kriseides:

- **Repo**: `Kriseides/site/server/` (sorella di `XPlanium/`)
- **Deploy**: vedi `Kriseides/site/server/DELIVERY.md` per istruzioni complete
- **Endpoint per XPlanium**: `POST /api/lead/xplanium` (rotta parametrica
  `/api/lead/:brand` con `brand=xplanium` già presente in `BRAND_CONFIG`)
- **CORS**: `xplanium.it` e `www.xplanium.it` sono già whitelistati
  nel `.env.example` del server

Il backend deve essere in produzione **prima** del go-live del sito, altrimenti
il form restituirà errore. Ordine consigliato: **deploy backend Kriseides →
verifica endpoint → deploy sito xplanium.it**.

### 4.1 Contratto dell'endpoint

- **Metodo**: `POST`
- **Content-Type**: `application/json`
- **CORS**: deve accettare richieste da `https://xplanium.it` e
  `https://www.xplanium.it`

Payload atteso (JSON):

```json
{
  "name":    "string (2-100 char, required)",
  "company": "string (2-150 char, required)",
  "email":   "string (email valid, required)",
  "phone":   "string (opt, max 40 char)",
  "sector":  "string (required, enum — vedi sotto)",
  "modules": ["Vendite", "Marketing", "Social", "Cliente", "Statistiche"],
  "message": "string (10-4000 char, required)",
  "privacy": true,
  "website": "honeypot (deve essere vuoto — se valorizzato, scartare)"
}
```

**Valori enum `sector`**:
`Impianti elettrici` · `Fotovoltaico` · `Idraulica e climatizzazione` ·
`Videosorveglianza` · `Domotica e automatismi` · `Infissi e serramenti` ·
`Edilizia e finiture` · `Piscine e SPA` · `Altro`

**Valori enum `modules`** (max 5): `Vendite`, `Statistiche`, `Cliente`,
`Marketing`, `Social`. Il modulo Core è sempre incluso di default e NON
viene mai inviato dal form.

Response attesa:
- `200 OK` + `{ "ok": true }` → notifica al cliente "Grazie, ti ricontattiamo"
- Qualsiasi altro status → banner rosso "Impossibile inviare, riprova"

### 4.2 Comportamento atteso lato backend

Alla ricezione, il backend Kriseides deve:

1. **Validare** il payload contro l'enum
2. **Scartare silenziosamente** (200 OK senza fare nulla) se `website` è
   valorizzato (honeypot anti-bot)
3. **Inviare email interna** a `info@xplanium.it` con tutti i dati del lead
4. **Inviare email di conferma** al richiedente (`body.email`) —
   soggetto: "Abbiamo ricevuto la tua richiesta — XPlanium"
5. **Rate-limit per IP** (consigliato: 5 richieste / 10 min)

### 4.3 Alternativa: usare il backend XPlanium

Il repo `xplanium_backend/` contiene un endpoint alternativo compatibile
(`POST /api/v1/public/contact` in `src/modules/publicContact/`) che si
può usare se in futuro si vuole disaccoppiare il lead di XPlanium dal
backend condiviso Kriseides. Non è la strada scelta oggi: si mantiene
il flusso Kriseides per centralizzare i lead di tutti i brand.

Se in futuro si decidesse di switchare:
1. Cambiare `CONTACT_API_URL` in `index.html` riga 4831-4833
2. Configurare le env vars SMTP del backend XPlanium
3. Aggiungere CORS per `xplanium.it` nella config del backend XPlanium

### 4.4 Configurazione dominio email su Brevo

Perché le email non finiscano in spam serve autenticare il dominio
`xplanium.it` su Brevo (una tantum):

1. Su Brevo → Senders & IP → Domains → **Add domain**: `xplanium.it`
2. Brevo fornisce 3 record da aggiungere al DNS di `xplanium.it`:
   - **SPF** (TXT): `v=spf1 include:spf.brevo.com ~all`
   - **DKIM** (TXT): valore specifico fornito da Brevo (es. `brevo1._domainkey`)
   - **DMARC** (TXT, consigliato): `v=DMARC1; p=quarantine; rua=mailto:dmarc@xplanium.it`
3. Attendere validazione Brevo (5-30 minuti dopo la propagazione DNS)

Verificare con `https://mxtoolbox.com/spf.aspx` che SPF e DKIM siano validi
prima di aprire il traffico al form.

---

## 5. Checklist di verifica post-deploy

Da fare **subito dopo** il go-live, in ordine:

**Frontend statico:**
- [ ] `https://xplanium.it` carica la home in < 2 secondi
- [ ] `http://xplanium.it` redirige a `https://`
- [ ] `https://www.xplanium.it` redirige a `https://xplanium.it` (o viceversa,
      basta essere coerenti — scegliere un canonical)
- [ ] Favicon visibile nel tab del browser (verificare su Chrome + Safari + Firefox)
- [ ] Le 4 pagine si aprono: `/`, `/privacy.html`, `/cookie.html`, `/accessibilita.html`
- [ ] Le immagini `assets/photos/*.png` caricano tutte (aprire DevTools →
      Network → filtro `img` → nessun 404)
- [ ] `https://xplanium.it/sitemap.xml` e `https://xplanium.it/robots.txt`
      restituiscono 200

**Form contatti (end-to-end):**
- [ ] Compilare il form dalla pagina live con dati reali → deve arrivare
      un'email a `info@xplanium.it` entro 60 secondi
- [ ] L'email di conferma arriva anche al richiedente (verificare cartella spam)
- [ ] Testare validazione: inviare form vuoto → banner rosso, focus sul
      primo campo mancante
- [ ] Testare honeypot: aprire DevTools, valorizzare il campo nascosto
      `website`, inviare → nessuna email arriva ma la UI mostra successo

**SEO / accessibilità:**
- [ ] Lighthouse (Chrome DevTools): Performance ≥ 90, Accessibility ≥ 95,
      Best Practices ≥ 95, SEO ≥ 95
- [ ] Google Search Console: aggiungere la property `xplanium.it`,
      submit di `sitemap.xml`
- [ ] Verificare rich snippet: incollare `https://xplanium.it` su
      https://search.google.com/test/rich-results (deve trovare
      `SoftwareApplication`, `Organization`, `FAQPage`)

**Cookie / GDPR:**
- [ ] Alla prima visita compare il banner cookie (bottom-right)
- [ ] Cliccando "Solo essenziali" il banner sparisce e in `localStorage`
      c'è la chiave `xp-cookie-consent` con `analytics: false, marketing: false`
- [ ] La pagina Cookie ha un pulsante per **revocare** il consenso

**Legali:**
- [ ] Il footer contiene la ragione sociale corretta (**Kriseides S.r.l.**),
      P.IVA, indirizzo, PEC
- [ ] Link a Privacy, Cookie, Accessibilità funzionanti
- [ ] Zero riferimenti residui a **Demontech** (vecchia ragione sociale)

---

## 6. Manutenzione e update

### Aggiornamenti di contenuto

Il sito è versionato in Git. Per pubblicare modifiche:

1. Editare i file localmente
2. `git commit` + `git push` sul branch di produzione
3. Se l'hosting è collegato al repo (raccomandato), il redeploy è
   **automatico** entro 1-2 minuti
4. Verificare in privato: aprire in modalità incognito (per evitare cache)

### Rotazione delle credenziali SMTP

Se cambiate credenziali Brevo:
- Se si usa il backend Kriseides (`api.kriseides.com`) → aggiornare lì
- Se si usa `xplanium_backend` → aggiornare `SMTP_USER` e `SMTP_PASS` nelle
  env vars del server, restart del servizio

### Backup

Il codice è su Git (backup implicito). L'unico stato dinamico è nelle email
inviate (log del server SMTP Brevo). Nessun database da backuppare per il
sito pubblico.

### Aggiornamento sitemap

Quando si aggiungono nuove sezioni con id anchor (`#nuova-sezione`),
aggiornare `sitemap.xml` inserendo la URL e ripubblicare.

### Monitoring consigliato

- **Uptime**: UptimeRobot (gratuito) o Pingdom → check ogni 5 min
  su `https://xplanium.it` e sull'endpoint del form
- **Errori JS**: opzionale, Sentry o simile (attualmente non integrato,
  aggiungere solo se si osservano problemi)
- **Traffico**: se si vuole analytics, aggiungerlo rispettando il
  consent (attivare solo se `analytics: true` in `localStorage` —
  la struttura del cookie manager è già predisposta)

---

## 7. Contatti tecnici / riferimenti

- **Repo sito**: `xplanium_website/` (branch `develop`)
- **Repo backend fallback**: `xplanium_backend/`, modulo
  `src/modules/publicContact/`
- **Backend gruppo Kriseides** (produzione): `api.kriseides.com` —
  responsabile team Kriseides
- **Casella lead in ingresso**: `info@xplanium.it`
- **Provider email**: Brevo (SMTP relay)
- **Hosting consigliato**: DigitalOcean App Platform

Per problemi tecnici sul form contatti verificare in ordine:
1. Endpoint API risponde (curl al `POST` con payload di test)
2. Credenziali SMTP valide (test da pannello Brevo)
3. Record SPF/DKIM/DMARC del dominio ancora attivi (mxtoolbox)
4. CORS del backend accetta `https://xplanium.it`
