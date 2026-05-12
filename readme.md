# FantASTA — Web app per aste Fantacalcio live

# SOFTWARE:

**FantASTA** è una web application pensata per **aste in presenza**: un unico dispositivo funge da **console di regia** (ricerca giocatori, timer, chiusura asta, roster e budget), mentre un secondo schermo — tipicamente un proiettore o TV — può mostrare una **vista “solo spettacolo”** con offerte, timer e squadre, senza controlli operativi.

L’obiettivo del prodotto è offrire **un’asta sincronizzata con l’esperienza live della stanza**: chi gestisce vede tutti gli strumenti; la sala vede solo ciò che conta per seguire l’asta in tempo reale.

---

## Architettura del sistema

| Livello | Responsabilità |
|--------|----------------|
| **Backend (Flask)** | Pagine HTML (Jinja2), servizio dell’upload CSV (restituzione del testo al client), salvataggio/restituzione dello stato asta e delle impostazioni nella **sessione server-side** (cookie firmato), primo file CSV disponibile in `csv/` come default. |
| **Frontend (JavaScript “vanilla”)** | Tutta la **logica dell’asta in tempo reale**: stato squadre, roster, budget, timer, rilanci, chiusura asta, ripristino backup, export. Nessun WebSocket: la UI è aggiornata nel browser del gestore; la TV riceve aggiornamenti tramite **finestra secondaria** e `postMessage`. |

In sintesi: **Flask ospita e persiste (in sessione) alcuni dati**; **il motore dell’asta vive nel browser** (`auction.js` + `csv_handler.js`).

---

## Funzionalità principali

### Setup personalizzabile (squadre, budget, regole)

<img width="893" height="766" alt="Screenshot 2026-05-12 alle 13 30 04" src="https://github.com/user-attachments/assets/48e7ddaa-22b3-41ce-9f7e-d47618586743" />


Dalla schermata iniziale (`templates/index.html`) si configurano:

- **Timer principale** dell’asta (secondi) e **soglia minima al rilancio**: se, al momento di un nuovo rilancio, il tempo residuo è inferiore alla soglia, il timer viene riportato al valore configurato (fino al massimo del timer principale).
- **Modalità ruoli**: *Ruoli a blocchi* (progressione POR → DIF → CEN → ATT con filtri coerenti) oppure *Ruoli liberi*.
- **Modalità chiamata**: *Scelta del battitore*, *A giro* (offerta iniziale 1 credito al chiamante), *Random* (estrazione guidata dal codice con controlli dedicati).
- **Numero squadre**, **budget iniziale**, **nomi squadre** e **slot per ruolo** (POR, DIF, CEN, ATT).

Opzioni aggiuntive: **Modalità sviluppo** (shortcut per test rapidi) e **Carica stato asta** da file JSON (vedi sotto).

### Gestione database calciatori (import / export CSV)

- **Import**  
  - All’avvio viene tentato il caricamento automatico del **primo file `.csv`** nella cartella `csv/` tramite l’endpoint Flask `GET /default-csv`.  
  - È possibile **caricare un altro CSV** dal form: il file viene inviato a `POST /upload-csv` e il **parsing avviene nel browser** con **Papa Parse** (`static/js/csv_handler.js`). Il formato atteso è quello **tipo export Fantacalcio** (righe con molte colonne: id, nomi, codice ruolo, squadra, nazionalità, URL immagine, ecc.); i ruoli vengono normalizzati in **POR / DIF / CEN / ATT**.  
  - Nota: il testo di aiuto in pagina menziona colonne “Name, Role, Team, Value”; il codice oggi è ottimizzato sul **layout Fantacalcio** — il campo “valore base” in tabella usa principalmente i default lato parser (es. valore iniziale crediti) salvo estensioni future.

- **Export**  
  Dalla fase asta, il pulsante **Esporta CSV** genera un file con struttura dedicata (righe segnaposto `$,$,$` e triple `nomeSquadra,idGiocatore,costo`) ad uso di tool esterni / leghe che importano questo formato.

### Dual-view: Console regia vs TV Mode




- **Console**: una volta avviata l’asta, l’interfaccia principale riunisce ricerca e avvio asta su un giocatore, pannello asta attiva (timer, offerta corrente), e bacheche squadre con budget e roster.
  <img width="892" height="760" alt="Screenshot 2026-05-12 alle 13 31 33" src="https://github.com/user-attachments/assets/6af0faee-9264-4070-ba2f-ab88597e89c0" />

- **TV Mode (finestra dedicata)**: dalla console, **TV Mode (1080p / 4K)** apre `static/tv_auction.html` in una nuova finestra. Ogni ~**300 ms** la pagina madre invia tramite `postMessage` l’HTML clonato delle sezioni asta attiva e delle bacheche squadre; la pagina TV **estrae testi e struttura dal DOM** e ridisegna layout ottimizzato per la visione da lontano (scala automatica da tela 3840×2160).  
<img width="1705" height="957" alt="Screenshot 2026-05-12 alle 13 32 38" src="https://github.com/user-attachments/assets/10121ecb-450d-4f4a-a824-116e41d59c45" />

La “sincronizzazione” è quindi **polling lato client tra finestra madre e popup**, non un canale server push multi-dispositivo.

### Motore asta live

Implementato in `static/js/auction.js`:

- **Avvio asta** dal click su un giocatore in elenco (filtri ricerca / ruolo secondo le regole scelte).
- **Timer** a conteggio verso zero; allo scadere: se c’è un miglior offerente, **assegnazione automatica** a quel team; altrimenti chiusura senza acquisto (con breve ritardo se non ci sono offerte).
- **Rilanci**: Ogni rilancio incrementa l’offerta di **1 credito**, con controlli su **budget**, **massimo offribile** in funzione dei **posti rimanenti da riempire**, e **slot per ruolo**.
- **Estensione tempo al rilancio** quando il tempo residuo scende sotto la soglia configurata (`minBidResetTimer` in `auctionState`).
- **Chiusura manuale**: pulsante *Chiudi asta* — in modalità *A giro* senza offerte può assegnare al chiamante a **1 credito**; con offerte assegna al **miglior offerente**.
- **Tasto `+` (numpad)**: assegna al miglior offerente se l’asta è ancora attiva e il timer è positivo.
- **`Esc`**: termina l’asta corrente senza passare da `assignPlayerToTeam` (comportamento da usare consapevolmente).

### Auto-save e ripristino

- **Salvataggio periodico** (~ogni 30 secondi) dell’oggetto di stato completo verso `POST /save-auction-state` e **copia in `localStorage`** (`fantaLeague_auction_backup`).
- **All’uscita dalla pagina** (`beforeunload`): backup in `localStorage` e tentativo di invio allo stesso endpoint (es. `navigator.sendBeacon`).
- **Al caricamento**: `GET /load-auction-state`; se assente, uso del backup locale; dialogo per **ripristinare** oppure **scaricare JSON e cancellare** lo stato server/local.
- **Manuale**: *Salva stato asta* scarica un JSON; *Carica stato asta* ricarica asta, giocatori e storico assegnazioni.

Quando tutti i roster risultano completi secondo la formazione, il codice può **fermare l’auto-save** e **pulire** stato server e localStorage.

---

## Guida rapida all’uso

1. **Ambiente**  
   Python 3 consigliato, dipendenze da `requirements.txt`, virtualenv facoltativa ma consigliata.

2. **Avvio server**  
   Dalla root del progetto:
   ```bash
   python main.py
   ```  
   Di default l’app è in ascolto su **`http://127.0.0.1:5001`** (`host 0.0.0.0`, utile in LAN). Alternativa equivalente: eseguire `app` tramite Flask/wsgi se preferite, ma il punto d’ingresso documentato nel repo è **`main.py`**.

3. **Sicurezza sessione (produzione)**  
   Impostare la variabile d’ambiente `SESSION_SECRET` a una stringa lunga e casuale così i cookie di sessione non usano il fallback di sviluppo in `app.py`.

4. **Configurare l’asta**  
   Aprire `/` nel browser, verificare/caricare il CSV, impostare squadre, budget, regole e avviare con **Start Auction**.

5. **Collegare la TV**  
   - Sul PC di regia: dopo l’avvio asta, clic su **TV Mode (1080p o 4K)**.  
   - Spostare la finestra sulla finestra del proiettore/display esteso e, se serve, passare a tutto schermo nel browser (**consigliato: stessa macchina** — `postMessage` funziona tra finestre dello stesso origin).

6. **Gestione live**  
   Battitore/avvocato d’asta usa la console: avvio giocatore, squadre rilanciano, chiusura con il pulsante o allo scadere timer. La TV si aggiorna grazie al ciclo di messaggi descritto sopra.

7. **Fine asta**  
   Una volta conclusa l'asta può essere esportato il file dei roster da inserire in leghe. Inoltre è possibile esportare i roster di ogni giocatore in formato jpeg.
---

## Struttura della directory (cartella pulita)

```
FantaLeagueDraft_Clean/
├── main.py                 # Entry point: avvio Flask su porta 5001 (debug)
├── app.py                  # Applicazione Flask: route, sessione, upload CSV, stato asta, default CSV
├── requirements.txt        # Dipendenze Python (Flask, ecc.)
├── package.json            # Riferimenti NPM opzionali (es. html2canvas da CDN nel codice)
├── csv/                    # CSV di default serviti da /default-csv (primo .csv trovato)
├── templates/
│   ├── layout.html         # Shell pagina: Bootstrap, Font Awesome, Papa Parse, FileSaver, blocchi Jinja
│   └── index.html          # Setup + fase asta + stili/script TV inline e apertura finestra TV
├── static/
│   ├── css/
│   │   └── custom.css      # Stili supplementari (timer, board, ecc.)
│   ├── js/
│   │   ├── csv_handler.js  # Upload/fetch CSV, Papa Parse, export risultati, caricamento default
│   │   └── auction.js      # Modello stato asta, timer, rilanci, board, tastiera, auto-save, modalità di gioco
│   └── tv_auction.html     # Pagina TV standalone: ricezione postMessage e layout proiezione
└── README.md               # Questa documentazione
```

---

## Stack tecnologico

| Componente | Uso nel progetto |
|-------------|-------------------|
| **Python 3** | Runtime backend |
| **Flask** | Routing, template, JSON API minima, session |
| **Jinja2** | Template (`layout.html`, `index.html`) |
| **JavaScript (vanilla)** | Logica applicativa lato client |
| **Bootstrap 5** | UI (tema dark da CDN in `layout.html`; Bootstrap 5 anche su `tv_auction.html`) |
| **Font Awesome** | Icone |
| **Papa Parse** | Parsing CSV nel browser |
| **FileSaver.js** | Download file (export CSV/JSON) |
| **html2canvas** (CDN on-demand) | Generazione immagini roster da stampa / archivio |

---

*FantASTA — web app per aste live coordinate tra console di regia e vetrina TV.*

# HARDWARE: Pulsanti di rilancio fisici.

Per rendere l'asta un'esperienza davvero immersiva e "televisiva", è stato sviluppato un sistema di pulsantiere fisiche personalizzate. Questo permette ai partecipanti di rilanciare premendo un vero bottone.

<img width="3180" height="1492" alt="2" src="https://github.com/user-attachments/assets/a3e7b37e-e5de-456a-bce4-1c63ef49a419" />

## Caratteristiche principali:
- Componentistica: Il sistema utilizza comuni Arcade Button (pulsanti da sala giochi), scelti per la loro resistenza e il feedback tattile immediato.
- Connettività: Ogni pulsantiera è dotata di un connettore USB Type-C. Questo permette di collegarle e scollegarle facilmente utilizzando cavi standard, facilitando il setup e il trasporto.
- Cervello del sistema: Il cuore del dispositivo è un Arduino Nano (o compatibile). La particolarità di questa scelta è che l'Arduino viene programmato per essere riconosciuto dal PC come una tastiera standard (HID).

<img width="3728" height="1408" alt="1" src="https://github.com/user-attachments/assets/c52553d2-1f11-41ef-8017-eb442a130717" />

## Perché questa soluzione:
- Plug & Play: Non sono necessari driver, software aggiuntivi o configurazioni complicate. Una volta collegato, il computer "vede" una tastiera e i tasti inviano semplicemente delle combinazioni di tasti.
- Zero Latenza: Trattandosi di un'emulazione hardware diretta, il segnale arriva al software istantaneamente, eliminando i ritardi che si potrebbero avere con un mouse o un'interfaccia web.
