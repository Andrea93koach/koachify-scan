# Koachify Scan — Quiz Funnel Interattivo

**Ultimo aggiornamento:** 29.06.2026  
**Versione:** v1.7  
**Stato:** In sviluppo attivo — Punti 1/2/3/4/5/6 completati, Supabase attivo, Punto 8 (OpenAI) e Punto 9 (SVG map) da fare

---

## Cos'è

Check-up professionale per coach, terapeuti e consulenti. L'utente risponde a domande adattive organizzate in 9 pianeti tematici e ottiene una **Clarity Map** personalizzata con il suo archetipo professionale. Strumento di lead generation verso la Clarity Call gratuita Koachify.

---

## File del progetto

```
Quiz Koachify Scan/
├── index.html          ← tutto il quiz (CSS + HTML + JS vanilla, file unico)
├── logo-scuro.svg      ← logo ufficiale Koachify (navy, per sfondo chiaro)
├── logo-bianco.svg     ← logo ufficiale Koachify (bianco, per sfondo navy)
└── KOACHIFY-SCAN.md    ← questo file
```

---

## Architettura tecnica

- **File unico** `index.html` — nessun framework, nessun backend, nessun build tool
- **Logica:** JS vanilla con state machine
- **Storage:** localStorage (risposta persistente tra refresh)
- **Offline:** tutti i calcoli lato client

---

## Struttura moduli

| # | Schermata | Contenuto |
|---|-----------|-----------|
| Welcome | Screen iniziale | Hero navy, logo bianco, CTA "Inizia il check-up" |
| Modulo 0 | Profilazione iniziale | 5 domande (nome, attività, fatturato, obiettivo, sfida aperta) |
| Moduli 1–9 | 9 Pianeti | 3 pilastri per pianeta, 2 domande per pilastro (percezione + verifica adattiva) |
| Bridge | Schermata di transizione | Dopo ogni pianeta: titolo + testo + image placeholder hero |
| Modulo 10 | Clarity Map | Risultato finale con archetipo, griglia semafori, priorità, CTA |

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
state._backAvailable   // true = l'utente può tornare indietro di 1 step
state.history[]        // stack navigazione per goBack()
```

---

## Design system

### Brand colors (CSS variables)
```css
--navy:       #283e4a   /* colore principale testi/header */
--green:      #88c057   /* accento principale */
--green-dark: #6da03f   /* verde scuro */
--green-soft: #eef6e6   /* verde chiarissimo (bg card selected) */
--bg:         #f4f7f5   /* sfondo app */
--bg-outer:   #dce8e0   /* sfondo pagina attorno al box quiz */
--white:      #ffffff
```

### Layout
- **App box:** `width: min(560px, 100vw)` — fisso 560px su desktop, full-width su mobile
- **Altezza:** `100dvh` / `100vh` — sempre full viewport, no scroll su domande
- **No scroll domande:** flex column con `option-card { flex: 1 }` — le card si distribuiscono nello spazio disponibile senza overflow
- **Scroll permesso:** solo su welcome, bridge, clarity map (via CSS `:has()`)
- **Shadow desktop:** `box-shadow: 0 4px 60px rgba(0,0,0,0.14)` quando viewport > 560px

### Componenti principali
- **Topbar:** logo SVG (40px) + badge modulo + freccia back (quando disponibile)
- **Progress bar:** gradiente verde, visibile su moduli 1–9
- **Module card:** sfondo navy, emoji sistema + nome pianeta + tag
- **Option cards:** badge lettera + testo + checkmark; `flex: 1` per riempire spazio senza scroll
- **Bridge card:** hero image placeholder (190px, gradiente navy→verde) + body con icon/titolo/testo
- **Clarity Map:** hero navy, griglia 3×3 semafori colorati, result-row archetipi, CTA

---

## UX: comportamenti speciali

### Auto-advance (implementato v1.2)
- Click su opzione → animazione selezione (360ms) → avanzamento automatico
- **Solo** domande a scelta multipla; le domande open text e i bridge mantengono il pulsante "Continua"

### Back limitato a 1 step (implementato v1.2)
- Dopo ogni avanzamento: appare freccia `<` in topbar + link "← Cambia risposta precedente" sotto Continua
- Un solo uso consentito consecutivo — dopo il back, entrambi i controlli scompaiono
- Al prossimo avanzamento si riabilitano
- Funzione: `goBackOne()` → setta `_backAvailable = false` → chiama `goBack()`

### Logo adattivo
- **Schermate chiare** (domande, bridge, clarity): `logo-scuro.svg`
- **Welcome** (sfondo navy): `logo-bianco.svg`
- Swap automatico in `showScreen()` via JS

---

## Asset da produrre (TODO)

### Immagini bridge (9 pianeti × 3 varianti semaforo = 27 immagini)
Ogni bridge mostra un'immagine hero 560×190px (o equivalente ratio 2.95:1) coordinata con il design del quiz. Il placeholder attuale è il gradiente navy con l'emoji del pianeta al centro.

Struttura placeholder attuale in HTML:
```html
<div class="bridge-img-placeholder">
  <div class="bridge-img-icon-large" id="bridge-img-icon">🌍</div>
  <!-- icona camera semitrasparente in alto a destra -->
</div>
```

Per sostituire con immagine reale:
```html
<!-- sostituire .bridge-img-placeholder con: -->
<img src="bridge/m1-green.jpg" alt="Pianeta Attrazione" class="bridge-hero-img">
```
```css
.bridge-hero-img {
  width: 100%;
  height: 190px;
  object-fit: cover;
  border-radius: var(--radius-lg) var(--radius-lg) 0 0;
}
```

---

## Roadmap PDF — Stato implementazione

Documento di riferimento: `KoachifyScan_Prompt_ClaudeCode.pdf`

| # | Punto | Stato | Note |
|---|-------|-------|------|
| 1 | UX/UI card risposta + breadcrumb + 189 sottotitoli | ✅ Completato | Push GitHub 26.06.2026 |
| 2 | Progress bar con riga dinamica + percentuale | ✅ Completato | 29.06.2026 |
| 3 | Bridge 1/2/3 nelle 3 posizioni con strutture indicate | ✅ Completato | 27.06.2026 |
| 4 | Gate email (Nome/Email/Telefono + microcopy + trust) | ✅ Completato | 27.06.2026 |
| 5 | Schermata elaborazione 8.5s con checklist animata | ✅ Completato | 27.06.2026 |
| 6 | Clarity Map finale: score sistemi + reveal animato + mappa SVG | ✅ Completato | 29.06.2026 |
| 7 | Integrazione Supabase | ✅ Attivo | Fix cripto.randomUUID, archetipo, 4 tabelle popolate |
| 8 | Integrazione OpenAI GPT-4o | 🔲 Da fare | API key da fornire |
| 9 | Mappa SVG Universo con semafori + calibrazione coords | 🔲 Da fare | — |

---

## Dettaglio tecnico — punti da implementare

### Punto 2 — Progress bar migliorata
- Riga dinamica sotto la barra: **"Stai analizzando il Pianeta [Nome] — Sistema [Nome Sistema]"**
- Percentuale visibile accanto/dentro la barra (es. `42%`)
- Gradiente barra: da `#6BA03E` a `#88C057` (esplicito, non solo variabili CSS)

### Punto 3 — Bridge restructure (MAJOR)
**Cambio architettura:** da 9 bridge (uno per pianeta) a **3 bridge di sistema** posizionati:
- **Bridge 1:** dopo Pianeta 3 (Viaggio) — fine Sistema Prosperità — **DOPPIA SCHERMATA**
  - Pagina A: badge "Sistema Prosperità analizzato" + stat 68% + anticipazione
  - Pagina B: testimonianza PLACEHOLDER (foto + citazione + contesto "prima di...")
- **Bridge 2:** dopo Pianeta 6 (Moltiplicazione) — fine Sistema Libertà — **SINGOLA**
  - Badge "Sistema Libertà analizzato" + stat 2.3x + anticipazione ultima sezione
- **Bridge 3:** dopo Pianeta 9 (Scuderia) — fine Sistema Serenità — **DOPPIA SCHERMATA**
  - Pagina A: "La tua Clarity Map è pronta" + griglia 3×3 con blur CSS 6px su semafori + CTA gate email
  - Pagina B: testimonianza finale PLACEHOLDER + CTA "Sblocca la mia Clarity Map"

Struttura visiva standard tutti i bridge:
- Sfondo `#e8f4e8` (diverso dalle domande)
- Badge verde validazione + Titolo rispecchiamento + Box stat verde + Anticipazione + CTA "Continua l'analisi"
- Progress bar visibile in basso

### Punto 4 — Gate email
- Headline: "La tua Clarity Map è pronta. Dove te la mandiamo?"
- Sottotitolo: "Abbiamo identificato i tuoi blocchi e i tuoi punti di forza..."
- Campi: Nome / Email professionale / WhatsApp o telefono
- CTA verde: "Sblocca la mia Clarity Map"
- Microcopy: "La ricevi anche via email così puoi rileggere quando vuoi. Nessuno spam, mai."
- Trust signal: "I tuoi dati sono al sicuro · Privacy Policy"

### Punto 5 — Schermata elaborazione
- Titolo: "Analizzando i tuoi risultati..."
- Barra progresso animata progressiva
- Checklist sequenziale con delay: [check] 27 pilastri elaborati → [check] Archetipo identificato → [animazione] Generando Clarity Map...
- Testo corsivo grigio sotto
- Durata: 8-12 secondi (o fino a risposta IA al Punto 8)

### Punto 6 — Clarity Map aggiornata
- Header: "La Clarity Map di [Nome]"
- Sottotitolo: "Analisi completata il [data] — Archetipo: [Nome Archetipo]"
- Griglia 3×3 con reveal animato: ogni cella si illumina con delay 150ms (come scansione)
- Score numerico per sistema sotto ogni riga: `XX/100` con colore semaforo prevalente
- Box insight IA (150-200 parole, tono diagnostico)
- Sezione "I tuoi Punti di Forza" — max 3 pianeti verdi con icona + nome
- Sezione "Le tue Priorità d'Azione" — max 3 pianeti rossi con azione concreta
- CTA finale verde: "Prenota la tua Clarity Call gratuita"
- Sottotitolo CTA: "Gratuita · 30 minuti · Senza impegno"
- Microcopy CTA: "Il nostro team ha già accesso alla tua Clarity Map. La call parte da dove il check-up si è fermato."

### Punto 7 — Supabase (⏸ attesa credenziali)
4 tabelle: `quiz_sessions`, `quiz_answers`, `quiz_leads`, `clarity_map`
- All'avvio: creare riga `quiz_sessions` + UUID in localStorage
- Ad ogni risposta: salvare in `quiz_answers` con pianeta/pilastro/tipo/risposta/semaforo
- Al completamento: aggiornare `quiz_sessions` con `completed_at`, archetipo, score
- Al submit gate email: inserire in `quiz_leads` e `clarity_map` per ogni pianeta

### Punto 8 — OpenAI GPT-4o (⏸ attesa API key)
- Chiamata API al submit gate email (trasparente all'utente)
- System prompt incluso nel PDF (diagnostico, in italiano, JSON output)
- User message con semafori pianeta, score sistemi, archetipo, fatturato dichiarato
- Fallback: report pre-scritto per archetipo se API fallisce/timeout 15s
- Output JSON: `archetipo_titolo`, `archetipo_descrizione`, `blocco_principale`, `punti_di_forza[]`, `priorita_azione[]`, `report_narrativo`, `prossimo_passo`

### Punto 9 — Mappa SVG con semafori
- Overlay cerchi SVG colorati sull'immagine PNG mappa Universo
- Funzione `renderSemaforiMappa(clarityMapData)`
- Colori: `#E53935` rosso / `#F0A500` giallo / `#88C057` verde
- Animazione fade-in sequenziale (100ms delay per cerchio)
- Oggetto `PILASTRI_COORDS` con x,y per tutti i 27 pilastri (da calibrare)
- Modalità debug temporanea: click sull'immagine → mostra coordinate

---

## TODO operativi

- [ ] **CTA Clarity Call** — collegare URL reale `https://koachify.it/clarity-call`
- [ ] **Testimonianze bridge** — inserire testi reali quando disponibili (ora PLACEHOLDER)
- [ ] **Immagine mappa Universo** — fornire PNG per il Punto 9
- [ ] **Credenziali Supabase** — da fornire all'utente quando si arriva al Punto 7
- [ ] **API Key OpenAI** — da fornire all'utente quando si arriva al Punto 8
- [ ] **Deploy dominio** — pubblicare su dominio/subdominio Koachify quando approvato

---

## Dettaglio tecnico — Supabase (⚠️ debug pendente)

**Progetto:** `Quiz Koachify Scan` su account Koachify  
**URL:** `https://qwxbuqabguvmpzcebygr.supabase.co`  
**Anon key:** `eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6InF3eGJ1cWFiZ3V2bXB6Y2VieWdyIiwicm9sZSI6ImFub24iLCJpYXQiOjE3ODI1Njg5MDksImV4cCI6MjA5ODE0NDkwOX0.yFQ0dMj9_L3YNDQEI6WF7LnUBHRXtmpCmIwsHBJrpiE`

**Tabelle create:** `quiz_sessions`, `quiz_answers`, `quiz_leads`, `clarity_map`  
**RLS:** abilitato con policy INSERT anon su tutte le tabelle + UPDATE anon su quiz_sessions  
**Problema:** quiz completato ma nessun dato arriva nelle tabelle — da debuggare  
**Ipotesi:** il client Supabase JS potrebbe non essere inizializzato correttamente prima dell'uso (timing CDN); verificare con F12 → Console per errori

**Hook JS implementati:**
- `startQuiz()` → `createSession()` — crea riga in quiz_sessions, salva UUID in localStorage
- Risposta percezione A → `saveAnswer(pianeta, pilastro, 'percezione', letter, 'red')`
- Risposta verifica → `saveAnswer(pianeta, pilastro, 'verifica', letter, sem)`
- `showClarityMap()` → `completeSession()` — aggiorna completed_at, archetipo, score sistemi
- `submitGateEmail()` → `saveLeadData()` — inserisce in quiz_leads + clarity_map per ogni pianeta

---

## Flusso schermate attuale (v1.5)

```
Welcome
  → Profilazione (Modulo 0) — 5 domande
  → Pianeta 1 Attrazione (3 pilastri × 2 domande)
  → Pianeta 2 Conversione
  → Pianeta 3 Viaggio
  → BRIDGE 1 SISTEMA PROSPERITÀ (doppia schermata)
      Pag A: badge + titolo + stat 68% + anticipazione + progress 33%
      Pag B: headline "Non sei solo..." + testimonianza PLACEHOLDER
  → Pianeta 4 Architettura
  → Pianeta 5 Laboratorio
  → Pianeta 6 Moltiplicazione
  → BRIDGE 2 SISTEMA LIBERTÀ (schermata singola)
      Badge + titolo + stat 2.3x + anticipazione + progress 67%
      Back arrow → torna a Pianeta 6
  → Pianeta 7 Bussola
  → Pianeta 8 Villaggio
  → Pianeta 9 Scuderia
  → BRIDGE 3 SISTEMA SERENITÀ (doppia schermata)
      Pag A: badge + "La tua Clarity Map è pronta" + griglia blur 6px + CTA
      Pag B: headline "Non sei solo..." + testimonianza PLACEHOLDER
  → GATE EMAIL: Nome / Email / Telefono + trust signal
  → ELABORAZIONE: 8.5s animata con checklist sequenziale (sfondo navy)
  → CLARITY MAP: archetipo + griglia 3×3 semafori + punti forza/priorità + CTA
```

---

## Copy testimonianze (da inserire quando disponibili)

**Bridge 1 Page B** — tema Prosperità/fatturato:
> Headline: "Non sei solo... e abbiamo analizzato moltissimi casi come il tuo"
> Sub: "Guarda cosa dice chi si trovava nella tua stessa situazione sul fronte del fatturato e dell'acquisizione clienti — prima di fare il check-up e farsi affiancare dal team Koachify"
> Placeholder: `[TESTIMONIANZA PROSPERITÀ]` + `[NOME CLIENTE]` + `[Ruolo]`
> Contesto: "Questa era la situazione prima di lavorare sul Sistema Prosperità."

**Bridge 3 Page B** — tema Clarity Map/diagnosi:
> Headline: "Non sei solo... e abbiamo analizzato moltissimi casi come il tuo"
> Sub: "Guarda cosa dice chi si trovava esattamente dove sei tu ora — prima di ricevere la sua Clarity Map e farsi affiancare dal team Koachify"
> Placeholder: `[TESTIMONIANZA FORTE]` + `[NOME CLIENTE]` + `[Ruolo]`
> Contesto: "Ecco cosa ha visto nella sua Clarity Map — e cosa è cambiato dopo."

---

## GitHub

**Repository:** https://github.com/Andrea93koach/koachify-scan  
**Live:** https://andrea93koach.github.io/koachify-scan/  
**Branch:** master → auto-deploy su GitHub Pages  
**Ultimo push:** 26.06.2026 (v1.3 — Punto 1)  
**Da pushare:** v1.4-v1.5 (Bridge 1/2/3 + Gate Email + Elaborazione + Supabase)

---

## TODO operativi

- [ ] **Punto 8 PDF** — OpenAI GPT-4o: analisi personalizzata nel box "Il tuo blocco principale" (API key da fornire)
- [ ] **Punto 9 PDF** — Calibrare PILASTRI_COORDS: inserire universo-koachify.png nella cartella, aprire quiz, cliccare sulla mappa in debug mode e annotare le 27 coordinate dalla Console
- [ ] **Testimonianze** — inserire testi reali in Bridge 1 e Bridge 3 (ora PLACEHOLDER)
- [ ] **Deploy dominio** — pubblicare su dominio/subdominio Koachify quando approvato

---

## Storico versioni

| Versione | Data | Modifiche principali |
|----------|------|---------------------|
| v1.0 | 26.06.2026 | Build iniziale completa: 9 moduli, logica adattiva, 4 archetipi, Clarity Map |
| v1.1 | 26.06.2026 | UI/UX app-like, logo ufficiale SVG integrato, no-scroll domande, box larghezza fissa 560px |
| v1.2 | 26.06.2026 | Auto-advance su click opzione (360ms), back limitato a 1 step, bridge redesign con image placeholder hero |
| v1.3 | 26.06.2026 | Punto 1 PDF: card con subtitoli (189 testi), breadcrumb contestuale, hover/selected states, push GitHub Pages |
| v1.4 | 27.06.2026 | Bridge 1 (doppia — Prosperità), Bridge 2 (singola — Libertà), Bridge 3 (doppia — Serenità + griglia blur) |
| v1.5 | 27.06.2026 | Gate Email, Schermata Elaborazione 8.5s, Supabase integrato (debug pendente), copy testimonianze aggiornato |
| v1.6 | 29.06.2026 | Supabase fix (crypto.randomUUID, archetipo pre-calcolato), progress bar dinamica (Punto 2), serve.ps1 |
| v1.7 | 29.06.2026 | Clarity Map redesign completo (Punto 6): header personalizzato, griglia 3x3 con score e reveal, score sistemi, mappa SVG + debug, insight box, CTA aggiornata |
