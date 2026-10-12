# Abitudini

App per tracciare abitudini ricorrenti (quotidiane, settimanali, mensili, annuali). Fa parte dell'ecosistema di app personali di Matteo insieme a CaliMat (attività fisica), PesoDigiuno (digiuno/peso) e NextApp (appuntamenti con preparazione).

## Stack

- **Frontend**: un solo file HTML/CSS/JS vanilla, senza bundler e senza dipendenze npm. Librerie da CDN: `supabase-js`, `papaparse@5.4.1`
- **Backend**: Supabase (Postgres + Auth), **lo stesso progetto Supabase di CaliMat e PesoDigiuno**
- **Hosting**: Cloudflare Pages
- **Versionamento**: Git tramite GitKraken
- **Packaging Android**: TWA generata con PWABuilder (`.well-known/assetlinks.json`, pacchetto in `versioni_precedenti/`)

## File del progetto

| File | Scopo |
|---|---|
| `index.html` | Tutta l'app (markup + CSS + JS) |
| `manifest.json` | Manifest PWA (tema `#2dd4bf`, sfondo `#0f1420`) |
| `icon-192.png`, `icon-512.png`, `icon-180.png`, `favicon-32.png` | Icone |
| `.well-known/assetlinks.json` | Verifica del dominio per la TWA Android |
| `abitudini-memoria.md` | Riepilogo delle decisioni prese |

Tutti i file vanno nella stessa cartella/repo, che viene deployata su Cloudflare Pages.

## Credenziali Supabase

`SUPABASE_URL` e `SUPABASE_ANON_KEY` reali sono già scritti in `index.html`. Non vanno mai sostituiti con segnaposto.

## Schema database

Prefisso delle tabelle: `abit_`. La convenzione condivisa è `Cali_`, `Peso_`, `abit_`, `Next_`, per evitare collisioni nello stesso progetto Supabase.

```sql
create table abit_abitudini (
  id uuid primary key default gen_random_uuid(),
  user_id uuid references auth.users default auth.uid(),
  nome text not null,
  tipo text not null default 'checkbox',         -- checkbox | quantita
  unita text,                                    -- solo per 'quantita' (testo libero: km, litri, minuti...)
  target_volte numeric not null default 1,       -- n. volte (checkbox) o obiettivo quantità (quantita)
  target_periodo text not null default 'giorno', -- giorno | settimana | mese | anno
  ordine int not null default 0,                 -- ordinamento "personale"
  colore text default '#2dd4bf',
  archiviata boolean default false,
  created_at timestamptz default now()
);

create table abit_abitudini_log (
  id uuid primary key default gen_random_uuid(),
  habit_id uuid references abit_abitudini(id) on delete cascade,
  user_id uuid references auth.users default auth.uid(),
  fatto_at timestamptz not null default now(),
  valore numeric,                                -- solo per 'quantita'
  nota text,
  created_at timestamptz default now()
);

alter table abit_abitudini enable row level security;
alter table abit_abitudini_log enable row level security;
create policy "own rows" on abit_abitudini for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
create policy "own rows" on abit_abitudini_log for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
```

Le tabelle esistono già nel progetto Supabase condiviso, con questi nomi definitivi. `loadAll()` carica solo le abitudini con `archiviata = false`. Le colonne `colore` e `archiviata` sono nello schema, ma la UI non le gestisce.

## Autenticazione

Email + password tramite Supabase Auth, come in CaliMat e PesoDigiuno. Lo stesso account vale per tutte le app del progetto. Il magic link non si usa.

- `sb.auth.signInWithPassword()`: login
- `sb.auth.signUp()`: registrazione
- `sb.auth.resetPasswordForEmail()`: recupero della password
- `sb.auth.onAuthStateChange()`: unico punto di avvio. Alterna la schermata di login e l'app, e innesca `loadAll()` seguito da `checkDailyReminder()`

## Concetti chiave del codice

### Tipi di abitudine
- **checkbox** (fatto / non fatto): ogni log vale 1 occorrenza. Pulsanti "Fatto ✓" (adesso) e "＋ manuale" (data e ora scelte, nota facoltativa).
- **quantita**: ogni log ha un `valore` numerico e il progresso è la somma dei valori nel periodo. L'unità è testo libero (`unita`). Il pulsante "＋ registra" apre un popup per la quantità.

`countInPeriod(h, periodo)` conta le occorrenze per il tipo checkbox e somma i `valore` per il tipo quantita. `fmtAmount()` formatta il valore insieme all'unità.

### Fine giornata configurabile
`state.dayStartHour` (0–23) si imposta dal menu **Impostazioni** ed è salvato in `localStorage` (`ab_day_start_hour`). Ogni calcolo del tipo "che giorno è" (`localDateKey`, `periodStart`, grafico) sposta l'orologio indietro di queste ore con `shifted()`/`dayShiftMs()`. Così un'attività fatta dopo mezzanotte conta ancora per il giorno precedente.

### Criterio "da fare oggi" (`isDueToday()`)
- **Giornaliere**: compaiono finché non raggiungono il target del giorno.
- **Settimanali/mensili/annuali**: usano il ritmo atteso. Si prende la frazione del periodo di calendario già trascorsa e la si moltiplica per il target: `Math.ceil(target * frazione)` per il tipo checkbox, `target * frazione` per il tipo quantita. L'abitudine compare solo se sei sotto questa soglia.

I periodi seguono il calendario (`periodStart`/`periodEnd`), spostati di `dayStartHour`: giorno, settimana lunedì–domenica, mese, anno.

`habitStatus()` sceglie il colore del chip: `ok` se il target è raggiunto, `warn` se l'abitudine è da fare oggi, `danger` negli altri casi.

### Tab, distinte per scopo
- **Oggi**: solo il riquadro "📋 Da fare oggi" con progresso e pulsante di registrazione rapida. Se non c'è niente da fare mostra "Tutto fatto per oggi".
- **Abitudini**: elenco completo. Ogni card ha nome, frequenza, chip del progresso, barra di avanzamento, "Ultima: X fa" (`fmtAgo`) e un grafico a barre degli ultimi 14 giorni (`renderChart`). Il menu ⋮ offre Modifica, Storico (popup con eliminazione dei singoli log) ed Elimina.
- **Storico**: elenco cronologico grezzo di tutti i log, senza grafici. Serve a correggere o eliminare singole registrazioni.

### Ordinamento
Ci sono tre modalità (`state.sortMode`, salvata in `localStorage` come `ab_sort`, predefinita `personale`):
- `personale`: basata sulla colonna `ordine`. Si riordina con il drag-and-drop HTML5 tramite la maniglia ⠿ (`attachDragHandlers` → `reorderHabits` → `persistOrdineBulk` su Supabase).
- `alfabetico`: ordina con `localeCompare` in italiano.
- `frequenza`: raggruppa per periodo (giorno → anno), poi per target decrescente.

Se al caricamento ci sono valori `ordine` duplicati, `loadAll()` li rinumera una volta e li salva.

### Eliminazioni con undo
Ogni cancellazione, che sia di un'abitudine o di un singolo log, toglie subito l'elemento dalla UI e mostra un toast "Annulla" per 5 secondi. Solo dopo esegue la `delete` reale su Supabase. È lo stesso schema usato in PesoDigiuno.

### Offline
Esiste una coda di sincronizzazione in `localStorage` (`ab_pending_sync`). Se una scrittura su Supabase fallisce, l'azione (`insert_habit`, `update_habit`, `delete_habit`, `insert_log`, `delete_log`) viene accodata. Al ritorno della rete viene ritentata con `flushPending()`, agganciata all'evento `online`, seguita da `loadAll()`. Un pallino di stato nell'header indica la connessione. Il service worker è minimo, registrato da un blob inline, e non fa cache aggressiva.

### Promemoria
`requestNotifPermission()` attiva le notifiche del browser. `checkDailyReminder()` le invia una volta al giorno (`ab_last_reminder`), all'apertura dell'app, con l'elenco delle abitudini da fare oggi. Non c'è Web Push: con l'app chiusa la notifica non arriva.

### Esporta / Importa (CSV, PapaParse)
- **Esporta** (`exportCSV`): scarica `abitudini-backup-YYYY-MM-DD.csv`, un unico CSV con la colonna `tipo_record` (`abitudine` | `log`) e le colonne `id, habit_id, nome, tipo, target_volte, target_periodo, unita, ordine, fatto_at, valore, nota`.
- **Importa** (`triggerImport` → `confirmImport` → `doImport`): legge il CSV, mostra quante abitudini e quanti log ha trovato e chiede conferma. Poi inserisce tutto come **nuove righe**, con nuovi UUID e `habit_id` dei log rimappati. Non sovrascrive mai i dati esistenti.

### Aggiornamento automatico
`setInterval(render, 60000)` aggiorna ogni minuto "Ultima: X fa" e i criteri "da fare".

## Cose da NON fare (vincoli noti)

- Niente frasi motivazionali, card motivazionale, confronto settimanale o badge streak (🔥). Sono stati tolti del tutto: restano solo dati oggettivi (progresso, ultima esecuzione, grafico).
- Niente gamification e niente feature che chiedono input aggiuntivo (journaling, mood tracker).
- Non usare `localStorage`/`sessionStorage` per dati che devono sincronizzarsi tra dispositivi. Vanno bene solo per preferenze locali: `ab_sort`, `ab_day_start_hour`, `ab_last_reminder`, `ab_pending_sync`.
- Non creare un nuovo progetto Supabase: il piano gratuito ne consente solo 2 per account. Abitudini resta nello stesso progetto di CaliMat e PesoDigiuno.
- Non usare il magic link per il login.
- Non sostituire `SUPABASE_URL` e `SUPABASE_ANON_KEY` con segnaposto.

## Possibili sviluppi futuri (non ancora richiesti)

- Avviso discreto quando ci sono troppe abitudini attive contemporaneamente (rischio di abbandono dopo la prima settimana)
- Gestione di `archiviata` e `colore` dalla UI
- Eventuale accesso condiviso multi-utente (come valutato per CaliMat/MammApp)
