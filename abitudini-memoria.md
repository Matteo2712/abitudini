# Abitudini: note di progetto

- Ultimo aggiornamento della memoria: 19/08/2026
- File generato il: 12/10/2026

## 1. Descrizione

App "Abitudini" per tracciare abitudini ricorrenti. Esempi: stretching, arnica, studio del fagotto, guardare i grafici, esami del sangue.

## 2. Stack e filosofia

1. PWA single-file HTML, come CaliMat e PesoDigiuno.
2. Backend Supabase, stesso progetto condiviso con CaliMat e PesoDigiuno.
3. Deploy su Cloudflare Pages.
4. Gestione Git con GitKraken.
5. TWA Android tramite PWABuilder.
6. URL e anon key Supabase già inseriti in `index.html`.

## 3. Database

1. Tabelle già create e popolate: `abit_abitudini`, `abit_abitudini_log`.
2. Prefisso `abit_` per le tabelle dell'app.

## 4. Autenticazione

1. Login con email e password, come nelle altre app.
2. Non si usa il magic link.

## 5. Funzioni richieste

1. Inserimento abitudini con frequenza (X volte al giorno, settimana, mese o anno).
2. Abitudini quantitative con unità di misura personalizzata (testo libero).
3. Grafico della frequenza per ogni abitudine.
4. Promemoria quotidiano delle abitudini da fare.
5. Tempo trascorso dall'ultima esecuzione.
6. Modifica ed eliminazione di abitudini e storico.
7. Inserimento manuale retroattivo di abitudini dimenticate.
8. Popup di annullamento (undo) su ogni cancellazione.
9. Tre tab: Oggi, Abitudini, Storico.
10. Ora di fine giornata configurabile dall'utente.
11. Ordinamento: alfabetico, per frequenza, personale (drag-and-drop).
12. Esporta e importa in CSV (PapaParse).

## 6. Decisioni

1. Nessuna frase motivazionale e nessun badge streak nell'app: tolti del tutto.
2. Restano solo dati oggettivi: progresso, ultima esecuzione, grafico.

## 7. Documentazione correlata

1. `CLAUDE.md`: documentazione tecnica del progetto (schema SQL, logica di calcolo, offline, esporta/importa).
2. `index.html`: l'app completa.
3. Questo file riassume solo le voci presenti in memoria.
