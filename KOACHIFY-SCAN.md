# Koachify Scan — Quiz Funnel Interattivo

**Ultimo aggiornamento:** 01.07.2026  
**Versione:** v3.0  
**Stato:** Attivo in produzione — embed su GHL (promo.koachify.it). Pendente: calibrazione PLANET_COORDS.

---

## Cos'è

Check-up professionale per coach, terapeuti e consulenti. L'utente risponde a domande adattive organizzate in 9 pianeti tematici e ottiene una **Clarity Map** personalizzata con il suo archetipo professionale, diagnosi AI e priorità d'azione concrete. Strumento di lead generation verso la Clarity Call gratuita Koachify.

---

## File del progetto

```
Quiz Koachify Scan/
├── index.html              ← tutto il quiz (CSS + HTML + JS vanilla, file unico)
├── universo-koachify.png   ← mappa visiva Universo Koachify (3840×2060)
├── logo-scuro.svg          ← logo ufficiale Koachify (navy, per sfondo chiaro)
├── logo-bianco.svg         ← logo ufficiale Koachify (bianco, per sfondo navy)
├── foto-martina.png        ← foto testimonial Bridge 1 (AI generata)
├── foto-luca.png           ← foto testimonial Bridge 3 (AI generata)
├── serve.ps1               ← server HTTP locale per test (porta 8080)
└── KOACHIFY-SCAN.md        ← questo file
```

**Avviare server locale:**
```powershell
cd "C:\Users\Utente\Desktop\Claude Test Code\Quiz Koachify Scan"
powershell -ExecutionPolicy Bypass -File serve.ps1
# → http://localhost:8080/
```

---

## Architettura tecnica

- **File unico** `index.html` — nessun framework, nessun backend, nessun build tool
- **Logica:** JS vanilla con state machine
- **Storage:** localStorage (sessione persistente tra refresh)
- **Offline:** tutti i calcoli lato client; AI e Supabase sono opzionali (graceful fallback)

---

## Struttura moduli

| # | Schermata | Contenuto |
|---|-----------|-----------|
| Welcome | Screen iniziale | Hero navy + card body arrotondata, CTA "Inizia il check-up" |
| Modulo 0 | Profilazione iniziale | 5 domande (nome, attività, fatturato, obiettivo, sfida aperta) |
| Moduli 1–9 | 9 Pianeti | 3 pilastri per pianeta, 2 domande per pilastro (percezione + verifica adattiva) |
| Bridge 1 | Dopo Pianeta 3 | Doppia schermata — Sistema Prosperità completato |
| Bridge 2 | Dopo Pianeta 6 | Singola schermata — Sistema Libertà completato |
| Bridge 3 | Dopo Pianeta 9 | Doppia schermata — Sistema Serenità + griglia blur |
| Gate Email | Prima della mappa | Nome / Email / Telefono → sblocca Clarity Map |
| Elaborazione | 15s animata | Checklist sequenziale animata, chiamata AI in background |
| Clarity Map | Risultato finale | Archetipo + griglia 3×3 + diagnosi AI + mappa Universo + CTA |

### I 9 Pianeti

| Pianeta | Sistema | Pilastri |
|---------|---------|---------|
| 1 — Attrazione | Prosperità | Pubblico / Guida mercato / Domanda |
| 2 — Conversione | Prosperità | Offerta / Flussi vendita / Ciclo promo |
| 3 — Viaggio | Prosperità | Percorso cliente / PI / Esperienza |
| 4 — Architettura | Libertà | Documenti / Proc. marketing / Proc. interna |
| 5 — Laboratorio | Libertà | CRM / Marketing tools / KPI |
| 6 — Moltiplicazione | Libertà | Team / AI / Task management |
| 7 — Bussola | Serenità | Assistenza h24 / KB / Onboarding |
| 8 — Villaggio | Serenità | Accademia / Community / Networking |
| 9 — Scuderia | Serenità | Tutor / Done With You / Done For You |

---

## Logica adattiva

```
Percezione (A/B/C)
  └─ A → semaforo ROSSO diretto (skip verifica)
  └─ B/C → mostra Verifica (A/B/C/D)
              └─ scoring: scoreSemaforo(perc, verif) → red/yellow/green
              └─ C+A/B → bias flag → domanda aperta condizionale
```

**Score pianeta:** somma 3 semafori pilastro → 0-1=rosso, 2-3=giallo, 4-6=verde

---

## 4 Archetipi

| # | Nome | Fascia fatturato | Pattern semafori |
|---|------|-----------------|-----------------|
| 1 | Il Potenziale Inespresso | 0–3k/mese | Prevalenza rossi |
| 2 | Il Tuttofare Esausto | 1.5k–8k/mese | Mix rossi/gialli |
| 3 | Il Freelance che vuole Scalare | 5k–25k/mese | Prevalenza gialli |
| 4 | L'Imprenditore in Costruzione | 15k+/mese | Molti verdi con buchi |

---

## Variabili JS principali

```js
state.fatturato        // lettera A–F da Q0.3 → asse archetipo
state.pillarSemafori   // "m1p1"…"m9p3" → red/yellow/green
state.planetSemafori   // "p1"…"p9" → semaforo pianeta aggregato
state.biasFlags        // "m1"…"m9" → true se bias rilevato
state.archetipo        // 1/2/3/4
state.leadData         // { nome, email, telefono }
state.sessionId        // UUID Supabase
state._backAvailable   // true = l'utente può tornare indietro di 1 step
state.history[]        // stack navigazione per goBack()
_aiReport              // oggetto JSON da OpenAI (null se fallisce/timeout)
_aiReportPromise       // Promise della chiamata AI (lazy update DOM)
```

---

## Integrazione OpenAI GPT-4o

**Quando:** subito al submit gate email, durante la schermata elaborazione (15s)  
**Modello:** `gpt-4o`, `response_format: {type:'json_object'}`, `max_tokens: 2000`, `temperature: 0.7`  
**Timeout:** 15s AbortController  
**Fallback:** se la chiamata fallisce o scade, la Clarity Map mostra solo i dati locali (archetipo calcolato, score pianeti) senza menzione dell'AI nell'interfaccia  
**Lazy update:** la mappa appare subito a 15s; se l'AI risponde dopo, aggiorna il DOM automaticamente  
**Storage:** il JSON completo del report viene salvato in `quiz_leads.report_text` (JSON.stringify) per consentire il caricamento da URL condiviso

**Struttura JSON output (7 sezioni):**
```json
{
  "archetipo_titolo": "...",
  "archetipo_descrizione": "3 righe personalizzate, max 60 parole",
  "sistema_prosperita": { "score": 0-100, "sintesi": "1 frase situazionale" },
  "sistema_liberta":    { "score": 0-100, "sintesi": "1 frase situazionale" },
  "sistema_serenita":   { "score": 0-100, "sintesi": "1 frase situazionale" },
  "blocco_principale": {
    "pianeta": "...",
    "perche_principale": "2 righe",
    "come_si_manifesta": ["...", "...", "..."],
    "cosa_sblocca": "1 frase"
  },
  "punti_di_forza": [{ "pianeta": "...", "score": 0-100, "significato": "..." }],
  "priorita_azione": [{
    "numero": 1, "pianeta": "...",
    "perche_adesso": "1 frase",
    "azione_concreta": "max 30 parole — cosa fare lunedì mattina",
    "risultato_atteso": "1 frase"
  }],
  "report_narrativo": "180-220 parole, testo continuo",
  "prossimo_passo": "1 frase max 35 parole, ponte verso Clarity Call"
}
```

**Regole tone of voice:** mai usare ecosistema / scalabilità / all-in-one / paradigma / sinergico / ottimizzare / massimizzare / implementare / superlativi. L'utente non deve mai sapere che c'è un'AI nell'interfaccia.

---

## Integrazione Supabase

**Progetto:** `Quiz Koachify Scan`  
**URL:** `https://qwxbuqabguvmpzcebygr.supabase.co`  
**Anon key:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...` (in index.html)

**Tabelle:** `quiz_sessions`, `quiz_answers`, `quiz_leads`, `clarity_map`  
**RLS:** INSERT + UPDATE + SELECT anon attivi (SELECT aggiunto il 01.07.2026 per report condivisi)

**Policy SELECT da aggiungere (se non già fatto):**
```sql
CREATE POLICY "shared_report_select" ON quiz_leads
  FOR SELECT TO anon USING (true);

CREATE POLICY "shared_report_select" ON clarity_map
  FOR SELECT TO anon USING (true);
```

**Hook JS:**
- `startQuiz()` → `createSession()` — UUID in localStorage + riga quiz_sessions
- Ogni risposta → `saveAnswer(pianeta, pilastro, tipo, lettera, semaforo)`
- `showClarityMap()` → `completeSession()` — completed_at + archetipo + score sistemi
- `submitGateEmail()` → `saveLeadData()` + `sendToGHL()` — quiz_leads + clarity_map + webhook GHL
- `generateAIReport()` → UPDATE `quiz_leads.report_text` (JSON completo) + `archetipo`

---

## Integrazione GHL (GoHighLevel)

**Webhook:** `https://services.leadconnectorhq.com/hooks/jqOTU47ZX031Nj5k6Ztn/webhook-trigger/7fdc9dcc-54ce-4ea1-b448-a9fda3d114c9`  
**Trigger:** submit gate email (funzione `sendToGHL()`)

**Payload inviato:**
```json
{
  "firstName":       "Nome del lead",
  "email":           "email@esempio.it",
  "phone":           "+39 333 ...",
  "telefono":        "+39 333 ...",
  "source":          "Koachify Scan",
  "tags":            ["koachify-scan", "archetipo-il-tuttofare-esausto"],
  "archetipo":       "Il Tuttofare Esausto",
  "clarity_map_url": "https://andrea93koach.github.io/koachify-scan/?report=UUID",
  "session_id":      "UUID"
}
```

**URL report condiviso:** `https://andrea93koach.github.io/koachify-scan/?report=UUID`  
Quando aperto, il quiz carica i dati dal DB (quiz_leads + clarity_map) e mostra direttamente la Clarity Map completa del lead senza rifare il quiz.

---

## Embed su GoHighLevel

**Widget HTML da incollare nel page builder GHL:**
```html
<style>
  .hl-quiz-wrap { width:100%; height:100vh; margin:0; padding:0; overflow:hidden; line-height:0; }
  .hl-quiz-wrap iframe { width:100%; height:100%; border:none; display:block; }
</style>
<div class="hl-quiz-wrap">
  <iframe src="https://andrea93koach.github.io/koachify-scan/"
    allow="clipboard-write" title="Koachify Scan" loading="eager"></iframe>
</div>
```

**Impostazioni sezione GHL:** padding 0, min-height 100vh, colonna height 100%.

---

## Mappa Universo

**File:** `universo-koachify.png` (3840×2060)  
**Overlay:** SVG `viewBox="0 0 1000 600" preserveAspectRatio="none"` (mapping 1:1)  
**Renderer:** 9 cerchi colorati per i 9 pianeti, colorati da `state.planetSemafori`

**PLANET_COORDS (da calibrare):**
```js
const PLANET_COORDS = [
  {x: 98, y:200},  // 1 Attrazione   ← stima, da verificare con debug click
  {x:159, y:200},  // 2 Conversione
  {x:128, y:268},  // 3 Viaggio
  {x:349, y:200},  // 4 Architettura
  {x:412, y:200},  // 5 Laboratorio
  {x:379, y:268},  // 6 Moltiplicazione
  {x:599, y:200},  // 7 Bussola
  {x:661, y:200},  // 8 Villaggio
  {x:629, y:268}   // 9 Scuderia
];
```

**Calibrazione:** clicca su un pianeta nella Clarity Map → Console: `[MapDebug] Pianeta ?: x:…, y:…` → aggiorna PLANET_COORDS in index.html.

---

## Design system

### Brand colors
```css
--navy:       #283e4a
--green:      #88c057
--green-dark: #6da03f
--green-soft: #eef6e6
--bg:         #f4f7f5
--white:      #ffffff
```

### Layout
- **App box:** `width: min(680px, 100vw)` — 680px su desktop, full-width su mobile
- **Altezza:** `100dvh` — full viewport, no scroll su domande
- **Scroll permesso:** welcome, bridge, clarity map (via CSS `:has()`)
- **Compact mobile:** `@media (max-height: 700px)` riduce font/padding per stare senza scroll
- **Touch hover fix:** `@media (hover: none)` — disabilita hover sticky su touch (fix pre-selezione)

---

## UX: comportamenti speciali

### Auto-advance
Click su opzione → animazione selezione (360ms) → avanzamento automatico. Solo domande a scelta multipla; open text e bridge mantengono il pulsante "Continua".

### Back limitato a 1 step
Dopo ogni avanzamento: freccia `<` in topbar + link "← Cambia risposta precedente" sotto Continua. Un solo uso consentito consecutivo.

### Logo adattivo
Schermate chiare → `logo-scuro.svg` / Welcome navy → `logo-bianco.svg`. Swap automatico in `showScreen()`.

### Welcome screen
Hero navy senza border-radius + body con `border-radius: 20px 20px 0 0` — effetto card che emerge, nessuna "barra blu" visibile.

---

## Roadmap — Stato implementazione

| # | Punto | Stato | Note |
|---|-------|-------|------|
| 1 | UX/UI card risposta + breadcrumb + 189 sottotitoli | ✅ | 26.06.2026 |
| 2 | Progress bar dinamica con label pianeta + percentuale | ✅ | 29.06.2026 |
| 3 | Bridge 1/2/3 nelle 3 posizioni | ✅ | 27.06.2026 |
| 4 | Gate email (Nome/Email/Telefono + trust) | ✅ | 27.06.2026 |
| 5 | Schermata elaborazione animata | ✅ | Timer 15s |
| 6 | Clarity Map: score sistemi + reveal + mappa SVG | ✅ | 29.06.2026 |
| 7 | Supabase (4 tabelle, RLS, hook completi) | ✅ | Fix crypto.randomUUID |
| 8 | OpenAI GPT-4o — report 7 sezioni + lazy update | ✅ | 29.06.2026 |
| 9 | Calibrazione PLANET_COORDS mappa Universo | ⏳ | Coordinate stimate |
| 10 | Testimonianze reali Bridge 1 e Bridge 3 | ✅ | Foto AI — 01.07.2026 |
| 11 | UX fix: desktop 680px, compact mobile, no-scroll | ✅ | 01.07.2026 |
| 12 | Fix welcome screen barra blu + CTA link | ✅ | 01.07.2026 |
| 13 | GHL webhook + URL report condiviso | ✅ | 01.07.2026 |

---

## TODO operativi

- [ ] **Calibrare PLANET_COORDS** — cliccare sui 9 pianeti nella Clarity Map (debug mode) e aggiornare le 9 coordinate in `index.html`
- [ ] **Deploy dominio** — pubblicare su dominio/subdominio Koachify dedicato quando approvato

---

## Testimonianze

### Bridge 1 Page B — Sistema Prosperità
- **Persona:** Martina R. — Coach nutrizionale e wellness
- **Foto:** `foto-martina.png` (generata con OpenAI gpt-image-1)
- **Testo:** *"Avevo clienti soddisfatti e risultati veri, ma ogni mese ripartivo quasi da zero. Il passaparola funzionava a tratti — poi si fermava. Non riuscivo a capire se il problema fosse l'offerta, il modo in cui mi comunicavo o qualcos'altro. Il check-up ha messo ordine: ho visto esattamente dove si rompeva il mio sistema. Per la prima volta avevo un'analisi chiara, non un'ipotesi."*

### Bridge 3 Page B — Clarity Map
- **Persona:** Luca B. — Business coach per consulenti
- **Foto:** `foto-luca.png` (generata con OpenAI gpt-image-1)
- **Testo:** *"Mi aspettavo un test come tanti. Ho ricevuto invece una diagnosi precisa: il mio blocco principale non era dove pensavo. Vedevo da mesi che qualcosa non funzionava — la Clarity Map mi ha mostrato cosa e perché. Quella chiarezza mi ha permesso di smettere di lavorare su tutto e di concentrarmi su una cosa sola. In tre mesi i risultati sono cambiati."*

---

## GitHub

**Repository:** https://github.com/Andrea93koach/koachify-scan  
**Live:** https://andrea93koach.github.io/koachify-scan/  
**Branch:** master → auto-deploy su GitHub Pages  
**Ultimo push:** 01.07.2026 (v3.0)

---

## Storico versioni

| Versione | Data | Modifiche principali |
|----------|------|---------------------|
| v1.0 | 26.06.2026 | Build iniziale: 9 moduli, logica adattiva, 4 archetipi, Clarity Map |
| v1.1 | 26.06.2026 | UI/UX app-like, logo SVG, no-scroll domande, box 560px |
| v1.2 | 26.06.2026 | Auto-advance, back 1 step, bridge redesign |
| v1.3 | 26.06.2026 | Punto 1 PDF: subtitoli (189 testi), breadcrumb, hover/selected states |
| v1.4 | 27.06.2026 | Bridge 1 (doppia), Bridge 2 (singola), Bridge 3 (doppia + griglia blur) |
| v1.5 | 27.06.2026 | Gate Email, Elaborazione 8.5s, Supabase integrato |
| v1.6 | 29.06.2026 | Supabase fix (crypto.randomUUID, archetipo), progress bar dinamica |
| v1.7 | 29.06.2026 | Clarity Map redesign: header, griglia 3×3, score sistemi, mappa SVG debug |
| v1.8 | 29.06.2026 | OpenAI GPT-4o integrato, report 7 sezioni, lazy DOM update, timer 15s |
| v1.9 | 29.06.2026 | Fix padding option card, universo-koachify.png aggiunto |
| v2.0 | 01.07.2026 | Testimonianze reali Bridge 1/3 con foto profilo AI (Martina R. + Luca B.) |
| v3.0 | 01.07.2026 | Desktop 680px, compact mobile, fix hover touch, fix barra blu welcome, CTA link promo.koachify.it/scan-clarity-call, GHL webhook + URL report condiviso (?report=UUID), Supabase SELECT policy |
