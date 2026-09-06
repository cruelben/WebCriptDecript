# WebCriptDecript

![HTML5](https://img.shields.io/badge/HTML5-E34F26?style=for-the-badge&logo=html5&logoColor=white)
![JavaScript](https://img.shields.io/badge/JavaScript-F7DF1E?style=for-the-badge&logo=javascript&logoColor=black)
![Client-Side](https://img.shields.io/badge/Privacy-Client_Side-success?style=for-the-badge)
![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg?style=for-the-badge)

**WebCriptDecript** è un'applicazione web avanzata e ultra-leggera, racchiusa interamente in un **singolo file `index.html`**, progettata per offrire un sistema di cifratura multi-livello e frammentazione ricorsiva dei file basato su standard di sicurezza geometrici. 

Nato come evoluzione moderna e portabile di un sistema di macro Excel/VBA, questo strumento permette di proteggere qualsiasi tipo di file trasformandolo in frammenti binari distribuiti su più container indipendenti.

---

## 🚀 Caratteristiche Principali

- **Zero Installazione & Portabilità Totale:** Non richiede l'installazione di software sul PC, né interpreti backend, database o strumenti esterni come 7-Zip. Funziona ovunque aprendo semplicemente il file `index.html` in un browser moderno.
- **Privacy al 100% (Client-Side Processing):** Tutti i calcoli di cifratura, hash, frammentazione e compressione avvengono interamente ed esclusivamente in locale all'interno della memoria RAM del browser tramite JavaScript nativo. Nessun dato, file o password viene mai trasmesso a server esterni.
- **Frammentazione Ricorsiva 4x4x4 (64 Blocchi):** Il file viene protetto da un header binario (`M22X`), sottoposto a passaggi di cifratura iterativi con chiavi derivate da salt specifici, e suddiviso geometricamente in 64 frammenti unici.
- **Distribuzione Multi-Container (`.m22c`):** I 64 frammenti e il file manifesto vengono mischiati casualmente e suddivisi in **3 archivi container distinti (`.m22c`)**. Per ricomporre il file originale è necessario disporre di **tutti e tre i container** e delle **3 password corrette**.
- **Mascheramento dei Contenitori:** Possibilità di scegliere lo stile di denominazione dei file container generati (nomi casuali mascherati come file di sistema/giochi, es. `nvidia_driver.m22c`, `steam_lib.m22c`, `fifa2010.m22c`, oppure nomi ordinati `gs-01.m22c`, `gs-02.m22c`, `gs-03.m22c`).

---

## 🛠️ Come Funziona l'Architettura di Sicurezza

La pipeline di protezione si articola in passaggi sequenziali rigorosi:

1. **Iniezione Header e 1° Livello:** Viene aggiunta una firma binaria di controllo (`M22X`) in testa al file originale, seguita da una trasformazione crittografica basata sulla **Password 1**.
2. **Frammentazione Primaria (4 parti) e 2° Livello:** Il flusso cifrato viene diviso in 4 sezioni distinte, ciascuna elaborata tramite la **Password 2**.
3. **Frammentazione Ricorsiva a 64 Blocchi e 3° Livello:** Ciascuna delle 4 parti viene ulteriormente suddivisa ricorsivamente in 4 blocchi (totale 16) e poi ancora in 4 blocchi (totale **64 frammenti binari finali**), cifrati singolarmente con la **Password 3**.
4. **Manifesto e Pacchettizzazione:** Viene generato un albero logico (`manifest_tree.dat`) che mappa l'ordine geometrico dei 64 frammenti. Il manifesto viene cifrato con una Master Key di sistema ed inserito nei container insieme ai frammenti, distribuiti in modo randomico su 3 archivi ZIP (`.m22c`).

---

## 📂 Struttura del Repository

Il repository è strutturato in modo minimale per garantire la massima semplicità di deployment:

    WebCriptDecript/
    │
    ├── index.html         # Applicazione web completa (UI + Logica di Cifratura/Decifratura)
    └── README.md          # Documentazione del progetto

---

## 💻 Guida all'Utilizzo

### 1. Avvio dell'Applicazione
Puoi utilizzare l'applicazione in due modi:
* **In Locale:** Scarica o clona il repository sul tuo computer e fai doppio clic sul file `index.html` per aprirlo direttamente nel tuo browser (Chrome, Edge, Firefox, Safari).
* **Online (Web Space / GitHub Pages):** Carica il file `index.html` su un tuo spazio web personale o attivalo direttamente tramite *GitHub Pages* nelle impostazioni del repository.

### 2. Cifratura di un File
1. Apri l'interfaccia e seleziona la scheda **"Cifratura & Protezione"**.
2. Clicca su **"Seleziona il file da proteggere"** e carica il file desiderato (qualsiasi formato: PDF, ZIP, Immagini, Documenti, ecc.).
3. Inserisci le **3 password di sicurezza** (puoi usare i valori di default o impostare password personalizzate e complesse).
4. Scegli lo **Stile Nomi Contenitori** (Casuali o Ordinati).
5. Clicca su **"Avvia Pipeline di Cifratura"**: l'applicazione elaborerà i dati in pochi istanti e scaricherà automaticamente i **3 file container `.m22c`** sul tuo PC.

### 3. Decifratura e Ripristino
1. Spostati sulla scheda **"Decifratura & Ripristino"**.
2. Seleziona contemporaneamente i **3 file container `.m22c`** generati in precedenza.
3. Inserisci le **3 password** utilizzate in fase di cifratura.
4. Clicca su **"Estrai Contenitori & Ripristina File Originale"**: il sistema verificherà l'integrità dell'header, ricomporrà geometricamente i 64 frammenti e scaricherà automaticamente il file originale ripristinato (anteponendo il prefisso `RIP_`).

---

## 🛡️ Requisiti di Sistema
- Un qualsiasi browser web moderno con supporto a JavaScript ES6+ e API Web Crypto / TextEncoder.
- Connessione a Internet **solo al primo avvio** per caricare la libreria `JSZip` tramite CDN (opzionalmente scaricabile e integrabile in locale per l'uso 100% offline).

---

## 📄 Licenza
Questo progetto è distribuito sotto licenza [MIT](LICENSE). Sentiti libero di usarlo, modificarlo e adattarlo alle tue esigenze di sicurezza personali.