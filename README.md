# M22C Secure Suite v2

**M22C Secure Suite** è una web application client-side per cifrare, frammentare, distribuire e successivamente ricostruire file locali senza utilizzare un backend applicativo.

Il progetto combina **AES-256-GCM**, **PBKDF2-SHA-256** e una struttura di frammentazione ricorsiva **4 × 4 × 4 = 64 frammenti logici**, seguita da un **packing dinamico in un numero variabile di container `.m22c`**.

> **Stato del progetto:** sviluppo/test. La logica è stata verificata staticamente e sono stati risolti diversi bug della gestione dinamica dei container; è comunque consigliato completare i test end-to-end nel browser prima di considerare il software pronto per uso critico.

---

## Indice

- [Caratteristiche principali](#caratteristiche-principali)
- [Architettura](#architettura)
- [Pipeline di cifratura](#pipeline-di-cifratura)
- [Container dinamici](#container-dinamici)
- [Pipeline di decifratura](#pipeline-di-decifratura)
- [Sicurezza e integrità](#sicurezza-e-integrità)
- [Compatibilità M22C v1 / v2](#compatibilità-m22c-v1--v2)
- [Interfaccia](#interfaccia)
- [Installazione ed esecuzione](#installazione-ed-esecuzione)
- [Utilizzo](#utilizzo)
- [Struttura del codice](#struttura-del-codice)
- [Formati e nomi](#formati-e-nomi)
- [Gestione degli errori](#gestione-degli-errori)
- [Limiti e considerazioni](#limiti-e-considerazioni)
- [Test consigliati](#test-consigliati)
- [Sviluppi futuri](#sviluppi-futuri)
- [Licenza](#licenza)

---

## Caratteristiche principali

- Cifratura locale nel browser.
- Tre livelli indipendenti di AES-256-GCM.
- Tre password distinte.
- PBKDF2-SHA-256 per la derivazione delle chiavi.
- 600.000 iterazioni per i nuovi container M22C v2.
- Compatibilità di lettura con i container legacy M22C v1 usando 210.000 iterazioni.
- Header applicativo `M22X` inserito prima della cifratura.
- Frammentazione logica ricorsiva 4 × 4 × 4, con 64 frammenti logici finali.
- Manifesto cifrato contenente i metadati necessari alla ricostruzione.
- Shuffling casuale dei dati prima del packing.
- Numero di container **dinamico**.
- Minimo sempre **3 container**.
- Dimensione massima del container configurabile dall'utente.
- Valore predefinito: **10 MB**.
- Un singolo frammento logico può essere ulteriormente suddiviso in parti fisiche.
- I container possono avere nomi casuali, progressivi o con prefisso personalizzato.
- Il decoder non dipende dal nome né dall'ordine dei container.
- Verifica finale della dimensione originale e dello SHA-256 in M22C v2.
- Supporto alla File System Access API quando disponibile.
- Fallback tramite selezione file e download Blob nei browser che non supportano la File System Access API.

---

## Architettura

Il progetto corrente è volutamente contenuto in un unico file:

```text
index.html
```

Il file contiene sia l'interfaccia HTML sia tutta la logica JavaScript.

### Tecnologie utilizzate

- **HTML5**
- **JavaScript ES6+**
- **Web Crypto API**
- **Tailwind CSS** tramite CDN
- **JSZip 3.10.1** tramite CDN
- **File System Access API** quando supportata dal browser

### Browser target

Il progetto è pensato principalmente per browser Chromium moderni, in particolare:

- Google Chrome
- Microsoft Edge
- Brave
- Opera
- altri browser basati su Chromium

---

## Pipeline di cifratura

La pipeline crittografica attuale è:

```text
FILE ORIGINALE
     │
     ▼
Header M22X
     │
     ▼
AES-256-GCM con Password 1
     │
     ▼
Divisione ×4
     │
     ▼
4 rami
     │
     ├── AES-256-GCM con Password 2
     │        │
     │        ▼
     │      Divisione ×4
     │        │
     │        ▼
     │      16 rami
     │        │
     │        ├── AES-256-GCM con Password 3
     │        │        │
     │        │        ▼
     │        │      Divisione ×4
     │        │        │
     │        │        ▼
     │        │      64 frammenti logici
     │        │
     ▼        ▼
Manifesto cifrato
     │
     ▼
Shuffling casuale
     │
     ▼
Packing dinamico
     │
     ├── Container 1
     ├── Container 2
     ├── Container 3
     ├── ...
     └── Container N
```

I **64 frammenti logici** sono parte della struttura crittografica e rimangono 64 anche quando il numero dei container fisici cambia.

Questa distinzione è importante:

```text
64 = frammenti logici della pipeline crittografica
N  = container fisici di archiviazione
```

N non è quindi vincolato a 64 e non deve essere necessariamente uguale a 3.

---

## Container dinamici

La versione attuale non utilizza più un numero fisso di tre container.

L'utente imposta nella sezione di cifratura:

```text
Dimensione massima container (MB)
```

Valore predefinito:

```text
10 MB
```

Il valore ammesso dall'interfaccia è compreso tra **1 MB e 4096 MB**.

La regola generale è:

```text
numero container >= 3
```

e il numero effettivo viene determinato dal packing necessario a rispettare il limite scelto.

### File piccoli

Anche un file molto piccolo produce almeno:

```text
3 container
```

### File grandi

Per file più grandi vengono creati tanti container quanti sono necessari.

Esempio concettuale:

```text
File piccolo     → 3 container
File medio       → 5 container
File grande      → 20 container
File molto grande → N container
```

Il numero esatto dipende dai dati cifrati e dal limite impostato.

### Limite sulla dimensione reale del container

Il codice cerca di rispettare il limite rispetto alla dimensione del **file ZIP `.m22c` effettivamente generato**, non solamente rispetto alla somma teorica dei payload.

Durante il packing viene usato un margine conservativo per l'overhead ZIP e, dopo la generazione, la dimensione effettiva viene controllata nuovamente.

Se il container risultante supera il limite, la cifratura viene interrotta con errore invece di produrre un container oltre il valore impostato.

---

## Parti fisiche dei frammenti

Un singolo frammento logico può essere troppo grande rispetto al limite configurato.

In questo caso non viene eliminato né modificato il frammento logico: viene diviso in più parti fisiche.

Esempio:

```text
sec_ab12_01.bin
```

può essere memorizzato fisicamente come:

```text
sec_ab12_01__part_001.bin
sec_ab12_01__part_002.bin
sec_ab12_01__part_003.bin
```

Il decoder riconosce questo schema, ordina le parti e ricompone nuovamente:

```text
sec_ab12_01.bin
```

prima di eseguire la ricostruzione crittografica 64 → 16 → 4.

---

## Pipeline di decifratura

Il decoder non richiede più esattamente tre container.

L'utente seleziona **almeno 3 container** e può selezionarne un numero maggiore.

La sequenza è:

```text
N container
     │
     ▼
Estrazione ZIP
     │
     ▼
Ricerca manifest_tree.dat
     │
     ▼
Decifratura manifesto
     │
     ▼
Validazione set di container
     │
     ▼
Ricostruzione delle eventuali parti fisiche
     │
     ▼
64 frammenti logici
     │
     ▼
Unione a gruppi di 4
     │
     ▼
16 blocchi
     │
     ▼
Decifratura Password 3
     │
     ▼
Unione a gruppi di 4
     │
     ▼
4 blocchi
     │
     ▼
Decifratura Password 2
     │
     ▼
Unione finale
     │
     ▼
Decifratura Password 1
     │
     ▼
Verifica M22X
     │
     ▼
Verifica dimensione
     │
     ▼
Verifica SHA-256
     │
     ▼
FILE RIPRISTINATO
```

### Ordine dei container

L'ordine con cui i container vengono selezionati non è significativo.

Il decoder utilizza:

```text
manifest_tree.dat
```

e i nomi dei frammenti contenuti nel manifesto per determinare la sequenza di ricostruzione.

Pertanto è possibile selezionare i container in ordine casuale.

### Nomi dei container

Il nome del container non è parte della logica di ricostruzione.

Un container può essere rinominato senza cambiare il contenuto interno e senza modificare l'algoritmo di ricostruzione.

---

## Manifesto

Ogni set di container contiene un file speciale:

```text
manifest_tree.dat
```

Il manifesto è cifrato tramite la combinazione delle tre password.

Per M22C v2 include almeno informazioni equivalenti a:

```json
{
  "version": 2,
  "algorithm": "AES-256-GCM",
  "kdf": {
    "name": "PBKDF2",
    "hash": "SHA-256",
    "iterations": 600000
  },
  "originalName": "file.ext",
  "originalSize": 12345678,
  "sha256": "...",
  "cryptoFragments": 64,
  "containerMaxBytes": 10485760,
  "fragments": [
    "sec_xxxx_01.bin",
    "sec_xxxx_02.bin",
    "..."
  ]
}
```

Il manifesto contiene quindi sia i dati necessari alla ricostruzione dei 64 frammenti logici sia i metadati necessari alla verifica finale.

---

## Sicurezza e integrità

### AES-256-GCM

La cifratura dei dati utilizza AES-256-GCM tramite Web Crypto API.

AES-GCM fornisce sia cifratura sia autenticazione del ciphertext. Una password errata o dati alterati normalmente impediscono la corretta decifratura del blocco.

### PBKDF2-SHA-256

Le chiavi AES vengono derivate dalle password tramite PBKDF2-SHA-256.

Parametri M22C v2:

```text
Hash:        SHA-256
Iterazioni:  600000
Salt:        16 byte
IV AES-GCM:  12 byte
Chiave:      256 bit
```

Per la compatibilità con i container legacy è mantenuta una modalità di lettura con:

```text
PBKDF2-SHA-256
210000 iterazioni
```

### Salt e IV casuali

Salt e IV vengono generati tramite:

```javascript
crypto.getRandomValues(...)
```

### Shuffling casuale

Lo shuffling dei frammenti utilizza Fisher-Yates con casualità proveniente da `crypto.getRandomValues()` e rejection sampling per evitare modulo bias.

### Integrità end-to-end

M22C v2 calcola lo SHA-256 del file originale prima della cifratura.

Durante la decifratura vengono verificati:

1. header `M22X`;
2. dimensione originale;
3. SHA-256 finale.

Se una di queste verifiche fallisce, il file non viene considerato validamente ricostruito.

---

## Password predefinite

Per comodità di test, le textbox sono preimpostate con:

```text
Password 1 = 123
Password 2 = 456
Password 3 = 789
```

Questi valori possono essere modificati dall'utente.

> **IMPORTANTE:** `123 / 456 / 789` sono password volutamente semplici e devono essere considerate valori di test. **Non devono essere usate per proteggere dati reali o sensibili.** Per un utilizzo reale devono essere sostituite con password robuste e non riutilizzate altrove.

---

## Compatibilità M22C v1 / v2

Il progetto mantiene la compatibilità di lettura con i container prodotti dalla precedente generazione M22C v1.

La procedura di lettura prova prima il formato v2 e, quando necessario, utilizza il percorso legacy con PBKDF2 a 210.000 iterazioni.

### Obiettivo di compatibilità

```text
Vecchio container M22C v1
        ↓
Decoder attuale
        ↓
Ricostruzione
```

I nuovi container vengono invece prodotti come:

```text
M22C v2 + container dinamici
```

La modifica del numero fisico dei container non cambia la pipeline crittografica 4 × 4 × 4.

---

## Interfaccia

L'interfaccia è divisa in due sezioni principali.

### Cifratura & Protezione

Comprende:

- selezione del file;
- Password 1;
- Password 2;
- Password 3;
- dimensione massima container;
- stile dei nomi container;
- cartella di destinazione;
- avvio pipeline.

### Decifratura & Ripristino

Comprende:

- selezione di almeno 3 container;
- Password 1;
- Password 2;
- Password 3;
- cartella di destinazione;
- opzione di eliminazione dei container originali dopo il completamento;
- ripristino e verifica del file.

---

## Stili di denominazione dei container

Sono disponibili tre modalità.

### Casuali

Vengono generati nomi casuali, ad esempio:

```text
qf2j8k91.m22c
91azp0xy.m22c
...
```

### Ordinati

Viene generata una sequenza progressiva, ad esempio:

```text
gs-01.m22c
gs-02.m22c
gs-03.m22c
gs-04.m22c
...
```

Il numero di cifre viene adattato al numero totale di container.

### Prefisso personalizzato

L'utente può inserire un prefisso, ad esempio:

```text
MIOFILE
```

e ottenere:

```text
MIOFILE-01.m22c
MIOFILE-02.m22c
MIOFILE-03.m22c
...
```

---

## Installazione ed esecuzione

Non è necessario installare un server applicativo.

È sufficiente avere:

```text
index.html
```

### Metodo rapido

Aprire `index.html` con un browser compatibile.

### GitHub Pages

Il progetto è adatto anche alla pubblicazione come sito statico tramite **GitHub Pages**.

Non è richiesto un backend per la pipeline di cifratura e decifratura.

### Nota sui CDN

L'applicazione carica Tailwind CSS e JSZip da CDN. Di conseguenza l'interfaccia e le librerie esterne richiedono normalmente una connessione Internet, salvo che tali dipendenze vengano successivamente portate localmente nel repository.

I file dell'utente vengono invece elaborati lato client dal browser e la logica applicativa non richiede un server M22C per la cifratura.

---

## Utilizzo

### Cifrare un file

1. Aprire la scheda **Cifratura & Protezione**.
2. Selezionare il file.
3. Verificare o modificare le tre password.
4. Impostare la dimensione massima dei container. Il default è **10 MB**.
5. Selezionare lo stile dei nomi.
6. Se supportato, scegliere la cartella di destinazione.
7. Avviare **Step 3 - Avvia pipeline di cifratura**.
8. Il programma genera almeno 3 container `.m22c`.

### Decifrare un set di container

1. Aprire la scheda **Decifratura & Ripristino**.
2. Selezionare almeno 3 container appartenenti allo stesso set.
3. Inserire le password corrette.
4. Se supportato, scegliere la cartella di destinazione.
5. Avviare **Step 3 - Estrai & Ripristina File**.
6. Il decoder estrae il manifesto, ricostruisce i 64 frammenti logici e verifica il risultato finale.

---

## Struttura del codice

### Utility e UI

```text
log()
setProgress()
setButtonState()
bytesToHex()
sha256Hex()
constantTimeStringEqual()
randomUint32()
randomInt()
generaStringaCasuale()
cryptoShuffle()
sanitizeFileName()
```

### Password e KDF

```text
passwordValida()
getEncryptionPasswords()
getDecryptionPasswords()
buildManifestPassword()
derivaChiaveAES()
```

### Crittografia

```text
cifraDati()
decifraDati()
```

### Frammentazione logica

```text
dividiInQuattro()
unisciQuattro()
```

### Container dinamici

```text
parseContainerMaxMB()
buildContainerNames()
splitBytesIntoParts()
buildZipForEntries()
packDynamicContainers()
```

### Manifesto e compatibilità

```text
parseLegacyManifest()
decryptManifest()
validateManifestV2()
```

### Decifratura dinamica

```text
estraiContainers()
reconstructDynamicFragments()
```

### Validazione

```text
validateContainerCount()
validateFragmentsPresence()
validateManifestAgainstContainers()
```

### File system

```text
meScegliCartellaOutput()
scegliCartellaOutput()
meScegliCartellaOutputDecrypt()
scegliCartellaOutputDecrypt()
getContainerSelection()
saveBytesToDirectory()
downloadBytes()
```

### Pipeline principali

```text
eseguiCifratura()
eseguiDecifratura()
```

---

## Formati e nomi

### Frammenti logici

I frammenti logici hanno una struttura equivalente a:

```text
sec_<prefisso4>_<indice2>.bin
```

Esempio:

```text
sec_8ahd_01.bin
sec_8ahd_02.bin
...
sec_8ahd_64.bin
```

### Parti fisiche

Quando un frammento viene suddiviso:

```text
sec_8ahd_01__part_001.bin
sec_8ahd_01__part_002.bin
...
```

Il nome logico rimane quello presente nel manifesto; le parti sono un dettaglio di storage.

### Container

Estensione prevista:

```text
.m22c
```

Tecnicamente il contenuto è un archivio ZIP creato tramite JSZip con modalità `STORE`.

---

## Gestione degli errori

Il software interrompe la procedura quando rileva condizioni non coerenti, tra cui:

- meno di 3 container in decifratura;
- container non validi o non leggibili come ZIP;
- `manifest_tree.dat` assente;
- password errate;
- manifesto v2 non valido;
- numero di frammenti logici diverso da 64;
- frammenti duplicati nel manifesto;
- frammenti mancanti;
- parti fisiche mancanti;
- frammenti estranei al manifesto;
- file duplicati tra container;
- header finale `M22X` non valido;
- dimensione finale non corrispondente al manifesto;
- SHA-256 finale non corrispondente;
- impossibilità di rispettare il limite del container impostato.

L'obiettivo è evitare ricostruzioni ambigue o risultati apparentemente validi ma corrotti.

---

## Bug critico già risolto

Durante la prima implementazione dei container dinamici è stato rilevato un errore di questo tipo:

```text
ERRORE: Frammento mancante: sec_8ahd_01.bin
```

Il problema era che il decoder cercava direttamente il frammento logico mentre il nuovo packing lo aveva memorizzato come una o più parti fisiche, per esempio:

```text
sec_8ahd_01__part_001.bin
```

La soluzione introdotta è la funzione:

```text
reconstructDynamicFragments()
```

che ricompone le parti fisiche prima della ricostruzione crittografica.

### Regola da non violare

Non reintrodurre una validazione che richieda la presenza diretta di tutti i 64 filename logici quando il set utilizza il formato con `__part_NNN.bin`.

---

## Duplicati e frammenti estranei

Gli oggetti estratti dai container vengono raccolti in una `Map`.

Se due container contengono lo stesso filename, il decoder interrompe la procedura invece di sovrascrivere silenziosamente il primo oggetto.

Nel formato dinamico sono consentiti i filename delle parti fisiche purché il frammento logico corrispondente sia dichiarato nel manifesto.

Questo serve anche a ridurre il rischio di combinare accidentalmente container appartenenti a file diversi.

---

## Eliminazione dei container dopo il ripristino

L'interfaccia offre un'opzione per eliminare i container originali al termine della procedura.

L'eliminazione avviene solo dopo:

1. ricostruzione riuscita;
2. verifica di integrità riuscita;
3. salvataggio del file riuscito;
4. conferma esplicita dell'utente.

La funzione richiede la File System Access API. Nel fallback basato sul download del browser l'eliminazione automatica non è disponibile.

---

## Limiti e considerazioni

### Non è un sistema server-side

La cifratura e la decifratura sono eseguite localmente nel browser.

### Non è anonimizzazione

Lo shuffling e la suddivisione in container rendono più difficile associare immediatamente i contenuti, ma **non devono essere interpretati come una tecnica crittografica aggiuntiva indipendente dalle cifrature AES-GCM**.

### Frammentazione non significa sicurezza aggiuntiva di per sé

La sicurezza principale deriva dalla corretta implementazione della crittografia e delle password. La frammentazione e il packing hanno principalmente finalità di distribuzione, separazione fisica e gestione dei dati.

### Password predefinite

I valori:

```text
123
456
789
```

sono destinati ai test e sono volutamente deboli.

### Audit crittografico

Il progetto non deve essere considerato sottoposto a un audit di sicurezza formale solo perché utilizza API crittografiche standard.

---

## Test consigliati

Prima di considerare stabile la versione dinamica, è consigliato eseguire almeno questi test:

### 1. File molto piccolo

```text
Input piccolo
→ 3 container
→ decifratura
→ SHA-256 coincidente
```

### 2. File che richiede più di 3 container

Verificare che il numero generato aumenti e che il decoder ricostruisca correttamente il file.

### 3. Superamento del limite da parte di un singolo frammento

Impostare un limite basso in modo da forzare:

```text
sec_xxxx_YY.bin
```

in più:

```text
__part_001.bin
__part_002.bin
...
```

### 4. Ordine casuale

Selezionare i container in ordine casuale durante la decifratura.

### 5. Rinomina

Rinominare i file `.m22c` prima della decifratura.

### 6. Password errate

Verificare che la ricostruzione non venga completata con password errate.

### 7. Container mancante

Rimuovere un container e verificare che il sistema segnali l'incompletezza del set.

### 8. Parte fisica mancante

Rimuovere intenzionalmente una parte `__part_NNN.bin` e verificare l'errore corretto.

### 9. Integrità

Modificare un contenuto/corrompere un container e verificare che AES-GCM o SHA-256 rilevino il problema.

### 10. File System Access API

Provare il browser sia con supporto sia senza supporto alla File System Access API.

### 11. Container legacy

Provare almeno un set M22C v1 già esistente e verificare che sia ancora leggibile.

---

## Sviluppi futuri

Possibili evoluzioni del progetto:

- suddivisione del JavaScript in moduli separati;
- test automatici di round-trip;
- test automatici di compatibilità v1/v2;
- test di integrità e corruzione intenzionale;
- migliore visualizzazione della distribuzione dei container;
- eventuale rafforzamento della protezione dei metadati del packing;
- possibilità di usare dipendenze locali invece dei CDN per avere una build realmente offline;
- documentazione formale del formato M22C per facilitare implementazioni future di decoder indipendenti.

Una possibile struttura futura potrebbe essere:

```text
js/
├── app.js
├── crypto.js
├── fragmentation.js
├── manifest.js
├── containers.js
└── filesystem.js

tests/
├── crypto-tests.js
├── fragmentation-tests.js
├── manifest-tests.js
└── roundtrip-tests.js
```

---

## Versioning

Il formato applicativo corrente è:

```text
M22C v2
```

La compatibilità con M22C v1 è mantenuta in lettura.

Una modifica futura che alteri in modo incompatibile la struttura crittografica, il manifesto o il formato dei dati dovrà essere trattata come una nuova versione del formato e non come una modifica invisibile a M22C v2.

---

## Licenza

**Nessuna licenza open-source è specificata in questo README.**

Prima di pubblicare il repository con una licenza precisa (MIT, GPL, Apache-2.0, ecc.), scegliere esplicitamente il modello di licenza desiderato e aggiungere il relativo file `LICENSE`.

---

## Autore

**Benevolo B.**

M22C Secure Suite

---

## Repository

Questo progetto è pensato per poter essere pubblicato come applicazione web statica, ad esempio tramite GitHub Pages.

