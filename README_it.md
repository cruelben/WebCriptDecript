# M22C Secure Suite v2

**M22C Secure Suite** è un'applicazione web client-side per la
protezione locale dei file tramite cifratura AES-256-GCM, derivazione
delle chiavi con PBKDF2-SHA-256 e frammentazione ricorsiva del payload
in 64 frammenti logici.

Il progetto è pensato per funzionare **senza un server applicativo**: il
file viene elaborato nel browser e i container M22C vengono generati
localmente.

> **Nota importante:** M22C è un progetto sperimentale/di protezione
> locale. La presenza della cifratura non deve essere interpretata come
> garanzia assoluta contro ogni forma di analisi del container, perdita
> delle password o compromissione del dispositivo/browser.

------------------------------------------------------------------------

## Funzionalità principali

-   Cifratura multilivello con **AES-256-GCM**.
-   Derivazione delle chiavi con **PBKDF2-SHA-256**.
-   600.000 iterazioni PBKDF2 per il formato v2.
-   Compatibilità di lettura con il precedente formato v1 a 210.000
    iterazioni.
-   Frammentazione ricorsiva **4 × 4 × 4 = 64 frammenti logici**.
-   Distribuzione casuale dei frammenti nei container.
-   Numero di container **dinamico**, determinato dal limite massimo
    impostato.
-   Minimo fisico di **3 container**.
-   Container ZIP con estensione `.m22c`.
-   Manifest cifrato contenente le informazioni necessarie alla
    ricostruzione.
-   SHA-256 del payload effettivamente cifrato e verifica dopo il
    ripristino.
-   Nomi container casuali, ordinati o con prefisso personalizzato.
-   Modalità opzionale di **camouflage JPG**.
-   Supporto al salvataggio tramite File System Access API quando
    disponibile.
-   Fallback al normale download del browser.
-   Possibilità, quando supportata dal browser, di eliminare i container
    originali dopo una decifratura verificata.
-   Nuova opzione, **disattivata di default**, per tentare la rimozione
    dei metadati prima della cifratura.

------------------------------------------------------------------------

# 1. Obiettivo del progetto

M22C nasce con l'obiettivo di trasformare un file originale in un
insieme di container che non contengono direttamente il file originale.

La pipeline concettuale è:

``` text
FILE ORIGINALE
      │
      ├── opzionale: tentativo rimozione metadati
      │
      ▼
M22X + dati
      │
      ▼
AES-256-GCM / Password 1
      │
      ▼
4 sezioni
      │
      ├── AES-256-GCM / Password 2
      │
      ▼
16 sezioni
      │
      ├── AES-256-GCM / Password 3
      │
      ▼
64 frammenti logici
      │
      ▼
shuffle casuale
      │
      ▼
suddivisione fisica
      │
      ▼
3 o più container .m22c
```

La decifratura esegue il percorso inverso utilizzando il manifesto
cifrato.

------------------------------------------------------------------------

# 2. Architettura

Il progetto è costituito da un singolo file:

``` text
index.html
```

Le principali tecnologie utilizzate sono:

-   HTML5
-   JavaScript ES6+
-   Tailwind CSS via CDN
-   Web Crypto API
-   JSZip 3.10.1
-   File System Access API, quando disponibile

Non è richiesto un backend applicativo per eseguire la pipeline di
cifratura.

------------------------------------------------------------------------

# 3. Cifratura multilivello

## Livello L1

Il file viene preceduto dall'header binario:

``` text
M22X
```

Il payload viene quindi cifrato con:

``` text
AES-256-GCM
Password 1
PBKDF2-SHA-256
600.000 iterazioni
```

La struttura prodotta contiene salt e IV casuali.

## Livello L2

Il risultato L1 viene diviso in 4 parti.

Ogni parte viene cifrata separatamente con:

``` text
Password 2
```

Si ottengono 4 sezioni L2.

## Livello L3

Ognuna delle 4 sezioni L2 viene nuovamente divisa in 4 parti e ciascuna
parte viene cifrata con:

``` text
Password 3
```

Le 16 sezioni risultanti vengono ulteriormente divise in 4.

Il risultato finale è:

``` text
16 × 4 = 64 frammenti logici
```

------------------------------------------------------------------------

# 4. Tre password

M22C utilizza tre password distinte:

``` text
Password 1
Password 2
Password 3
```

Le password vengono utilizzate nei tre livelli della pipeline.

Il manifesto viene cifrato utilizzando una password derivata dalla
concatenazione:

``` text
Password1_Password2_Password3
```

Le password di test presenti nell'interfaccia sono:

``` text
123
456
789
```

**Per un utilizzo reale devono essere sostituite con password robuste e
non riutilizzate.**

------------------------------------------------------------------------

# 5. PBKDF2 e AES-GCM

Il formato v2 utilizza:

``` text
PBKDF2
Hash: SHA-256
Iterazioni: 600000
Salt: 16 byte
IV AES-GCM: 12 byte
Chiave AES: 256 bit
```

Per ogni operazione di cifratura viene generato un salt casuale e un IV
casuale tramite `crypto.getRandomValues()`.

La generazione casuale applicativa utilizza inoltre Fisher-Yates con
rejection sampling per evitare il modulo bias nella generazione degli
indici casuali.

------------------------------------------------------------------------

# 6. Manifesto cifrato

Ogni set di container contiene un:

``` text
manifest_tree.dat
```

Il manifesto è cifrato.

Nel formato v2 contiene, tra le altre informazioni:

-   versione del formato;
-   algoritmo;
-   parametri KDF;
-   nome originale;
-   dimensione del payload cifrato;
-   SHA-256;
-   numero dei frammenti logici;
-   elenco ordinato dei 64 frammenti;
-   informazioni relative alla sanitizzazione dei metadati, quando
    richiesta;
-   limite massimo configurato per i container.

Il manifesto è fondamentale perché permette alla decifratura di
conoscere l'ordine esatto con cui ricomporre i frammenti.

------------------------------------------------------------------------

# 7. Frammentazione e distribuzione

I 64 frammenti logici non vengono semplicemente salvati in ordine.

Prima del packaging viene applicato uno shuffle casuale
crittograficamente appropriato:

``` text
Fisher-Yates
+
crypto.getRandomValues()
+
rejection sampling
```

I frammenti vengono quindi distribuiti fisicamente nei container.

Questo significa che un singolo container normalmente contiene soltanto
una parte del materiale necessario alla ricostruzione.

------------------------------------------------------------------------

# 8. Container dinamici

La versione attuale non utilizza più un numero fisso di tre container.

L'utente può impostare:

``` text
Dimensione massima container (MB)
```

Valore predefinito:

``` text
10 MB
```

Il programma determina automaticamente quanti container sono necessari.

Regole:

-   almeno 3 container;
-   il limite configurato viene applicato al container `.m22c`;
-   per file più grandi possono essere generati più di 3 container;
-   la decifratura accetta un numero variabile di container;
-   in fase di estrazione vengono controllati duplicati, frammenti
    mancanti e frammenti estranei al manifesto.

È previsto un limite massimo configurabile di:

``` text
4096 MB
```

------------------------------------------------------------------------

# 9. Struttura fisica dei frammenti

Quando un frammento logico non entra interamente nel target fisico di un
container, può essere ulteriormente suddiviso.

Esempio:

``` text
sec_ab12_01.bin
```

può diventare fisicamente:

``` text
sec_ab12_01__part_001.bin
sec_ab12_01__part_002.bin
sec_ab12_01__part_003.bin
...
```

Il programma ricostruisce queste parti durante la decifratura prima di
procedere alla ricomposizione logica dei 64 frammenti.

------------------------------------------------------------------------

# 10. Verifica SHA-256

Nel formato v2 viene calcolato SHA-256 sul **payload che viene
effettivamente cifrato**.

Questo dettaglio è importante quando è attiva la funzione di tentativo
di rimozione dei metadati.

### Senza sanitizzazione

``` text
file originale
      ↓
SHA-256
      ↓
cifratura
```

### Con sanitizzazione riuscita

``` text
file originale
      ↓
tentativo rimozione metadati
      ↓
payload sanitizzato
      ↓
SHA-256
      ↓
cifratura
```

Durante la decifratura vengono verificati:

1.  header `M22X`;
2.  dimensione del payload;
3.  SHA-256 del payload ricostruito.

Se il controllo SHA-256 fallisce, il file non viene considerato
correttamente ripristinato.

------------------------------------------------------------------------

# 11. Nuova funzione: tentativo di rimozione metadati

Nell'interfaccia di cifratura è presente:

``` text
☐ Prova a rimuovere metadati prima della cifratura
```

L'opzione è **disattivata per impostazione predefinita**.

La dicitura è volutamente prudente.

M22C **non promette di eliminare tutti i metadati**.

La funzione esegue un tentativo best-effort solamente per i formati
attualmente gestiti.

## JPEG

Per i JPEG vengono individuati e rimossi i segmenti utilizzati
comunemente per:

-   EXIF;
-   XMP;
-   IPTC / Photoshop IRB;
-   commenti JPEG.

La parte compressa dell'immagine viene lasciata invariata.

Questo approccio evita di ricodificare l'immagine semplicemente per
rimuovere i metadati.

## PNG

Per i PNG vengono rimossi specifici chunk associati a informazioni
metadata, tra cui:

``` text
eXIf
tEXt
zTXt
iTXt
tIME
pHYs
cHRM
gAMA
sRGB
iCCP
```

I chunk non inclusi nell'elenco vengono conservati.

## Formati non supportati

Per gli altri formati il programma non tenta modifiche generiche.

Il file viene cifrato senza sanitizzazione e il log comunica che il
formato non è supportato per la rimozione automatica dei metadati.

Questo comportamento evita di fare modifiche arbitrarie o potenzialmente
distruttive a formati non conosciuti.

------------------------------------------------------------------------

# 12. Il file originale non viene modificato

La funzione di sanitizzazione lavora sui dati caricati **in memoria**.

Il `File` originale selezionato dall'utente non viene sovrascritto e non
viene modificato sul filesystem.

Se la sanitizzazione riesce, viene cifrata la versione sanitizzata
presente in memoria.

Di conseguenza, quando l'opzione è attiva, il file ripristinato può
essere **diverso byte-per-byte dall'originale** se alcuni metadati sono
stati effettivamente rimossi.

Il nome originale viene comunque mantenuto nel manifesto.

------------------------------------------------------------------------

# 13. Importante: sanitizzazione ≠ anonimizzazione garantita

La funzione deve essere considerata una misura di riduzione dei
metadati, non una garanzia di anonimizzazione completa.

Per esempio:

-   un formato può contenere metadati non riconosciuti;
-   alcune informazioni possono essere incorporate nella struttura del
    file;
-   alcuni formati non sono attualmente supportati;
-   informazioni possono esistere anche al di fuori del file, ad esempio
    nel filesystem, nel nome del file, nei timestamp del filesystem o
    nei container prodotti.

La dicitura:

``` text
Prova a rimuovere metadati
```

è quindi intenzionale.

------------------------------------------------------------------------

# 14. Camouflage JPG

M22C può opzionalmente camuffare ogni container all'interno di un JPEG.

Il payload M22C viene aggiunto come trailer dopo il contenuto JPEG.

Il programma utilizza un identificatore specifico:

``` text
M22C-JPEG-PAYLOAD-V1
```

La decifratura può riconoscere il trailer e recuperare il container
M22C.

Sono disponibili:

-   generazione automatica delle immagini;
-   una singola immagine riutilizzata;
-   un'immagine differente per ogni container;
-   nomi casuali;
-   nomi ordinati.

Il camouflage è **disattivato per impostazione predefinita**.

Il limite MB configurato continua a riferirsi al container `.m22c`; il
JPEG finale può quindi avere una dimensione maggiore.

------------------------------------------------------------------------

# 15. Nomi dei container

Sono disponibili tre modalità:

### Casuali

Esempio:

``` text
a8k3m1xz.m22c
q9f2p7ab.m22c
```

### Ordinati

Esempio:

``` text
M22C-01.m22c
M22C-02.m22c
M22C-03.m22c
```

### Prefisso personalizzato

Esempio:

``` text
PROGETTO-01.m22c
PROGETTO-02.m22c
PROGETTO-03.m22c
```

La modalità casuale è utile quando si desidera evitare nomi
immediatamente descrittivi.

------------------------------------------------------------------------

# 16. Metadati dei container

La cifratura del contenuto non significa che il contenitore ZIP sia
privo di informazioni strutturali.

Un osservatore che possiede un `.m22c` può, a seconda degli strumenti
utilizzati, vedere informazioni come:

-   il fatto che il file sia un archivio ZIP;
-   i nomi degli elementi interni;
-   il numero di elementi presenti;
-   le dimensioni dei dati;
-   informazioni strutturali ZIP;
-   timestamp ZIP, se presenti.

Il contenuto dei frammenti e il manifesto sono invece cifrati.

Questa distinzione è importante:

``` text
Cifratura del contenuto
≠
eliminazione automatica di tutti i metadati del contenitore
```

------------------------------------------------------------------------

# 17. Decifratura

La procedura di decifratura è:

``` text
container M22C / camouflage JPG
              ↓
estrazione
              ↓
ricerca manifest_tree.dat
              ↓
decifratura manifesto
              ↓
validazione manifesto
              ↓
verifica presenza dei 64 frammenti
              ↓
ricostruzione dei frammenti fisici
              ↓
ricomposizione L3
              ↓
AES-GCM / Password 3
              ↓
ricomposizione L2
              ↓
AES-GCM / Password 2
              ↓
ricomposizione L1
              ↓
AES-GCM / Password 1
              ↓
verifica M22X
              ↓
verifica dimensione
              ↓
verifica SHA-256
              ↓
RIP_nomeoriginale
```

------------------------------------------------------------------------

# 18. Controlli di integrità

La decifratura esegue controlli per individuare:

-   numero insufficiente di container;
-   manifesto mancante;
-   manifesto non valido;
-   versione non supportata;
-   algoritmo non riconosciuto;
-   KDF non coerente;
-   numero di frammenti diverso da 64;
-   frammenti duplicati;
-   frammenti mancanti;
-   frammenti estranei;
-   password errate;
-   container corrotti;
-   header `M22X` non valido;
-   dimensione finale errata;
-   SHA-256 non corrispondente.

------------------------------------------------------------------------

# 19. Compatibilità browser

Il programma utilizza:

-   Web Crypto API;
-   File API;
-   File System Access API quando disponibile;
-   API browser standard.

La selezione e il salvataggio tramite cartella dipendono dal supporto
del browser alla File System Access API.

Quando l'API non è disponibile, M22C utilizza il normale download del
browser.

L'eliminazione automatica dei container originali è disponibile
solamente quando il browser consente l'accesso filesystem necessario.

------------------------------------------------------------------------

# 20. Privacy e modello operativo

L'applicazione è progettata per elaborare i dati localmente nel browser.

Non è previsto un server applicativo M22C che riceva il file da cifrare.

Tuttavia, il modello di sicurezza reale dipende anche dall'ambiente in
cui viene eseguito il file HTML:

-   browser utilizzato;
-   estensioni installate;
-   sistema operativo;
-   sicurezza del dispositivo;
-   eventuale malware;
-   gestione delle password;
-   CDN da cui vengono caricate le librerie;
-   modalità di distribuzione dell'HTML.

Per un ambiente con requisiti di sicurezza elevati è consigliabile
valutare anche l'utilizzo di dipendenze locali e verificabili invece di
CDN esterni.

------------------------------------------------------------------------

# 21. Struttura logica del progetto

Il file `index.html` comprende:

``` text
UI
├── Tab Cifratura
├── Tab Decifratura
├── Password
├── Limite container
├── Nomi container
├── Sanitizzazione metadati
├── Camouflage JPG
├── Cartelle output
├── Progress bar
└── Log

Crypto
├── PBKDF2
├── AES-256-GCM
├── SHA-256
├── Randomness
└── Fisher-Yates

Fragmentation
├── L1
├── L2
├── L3
└── 64 frammenti

Packaging
├── ZIP / JSZip
├── Dynamic containers
├── Physical parts
└── Camouflage JPG

Decryption
├── Extraction
├── Manifest
├── Validation
├── Reconstruction
├── Integrity check
└── Restore
```

------------------------------------------------------------------------

# 22. Limiti attuali

M22C non deve essere considerato un sistema universale di sanitizzazione
dei file.

In particolare:

-   la rimozione automatica dei metadati è attualmente best-effort;
-   JPEG e PNG sono i formati attualmente trattati dalla nuova funzione;
-   altri formati vengono lasciati invariati quando la sanitizzazione è
    richiesta ma non supportata;
-   i container rimangono archivi ZIP e possono quindi esporre
    informazioni strutturali;
-   la perdita delle tre password impedisce la decifratura;
-   il progetto dipende dalle API disponibili nel browser;
-   file molto grandi possono richiedere una quantità significativa di
    RAM perché la pipeline lavora in memoria.

------------------------------------------------------------------------

# 23. Test consigliati

Prima di utilizzare M22C con dati importanti è consigliato eseguire
almeno questi test:

### Test 1 --- File piccolo

``` text
test.txt
```

Verificare:

-   vengono creati almeno 3 container;
-   tutti i container contengono dati;
-   la decifratura ricostruisce correttamente il file.

### Test 2 --- JPEG senza sanitizzazione

``` text
foto.jpg
```

Con:

``` text
☐ Prova a rimuovere metadati
```

disattivato.

### Test 3 --- JPEG con sanitizzazione

Ripetere il test con:

``` text
☑ Prova a rimuovere metadati
```

Verificare nel log che il tentativo sia stato eseguito.

### Test 4 --- PNG

Ripetere il test con un PNG contenente metadata.

### Test 5 --- Formato non supportato

Utilizzare un formato diverso da JPEG/PNG e verificare che M22C segnali
correttamente:

``` text
Formato non supportato per la sanitizzazione automatica
```

e proceda comunque con la cifratura.

### Test 6 --- Container dinamici

Provare diversi valori:

``` text
1 MB
10 MB
50 MB
100 MB
```

e verificare che il numero di container cambi in funzione della
dimensione.

### Test 7 --- Camouflage

Generare container camouflage e verificare che:

-   il JPEG rimanga apribile come immagine;
-   la decifratura riconosca il payload;
-   il file venga ripristinato correttamente.

------------------------------------------------------------------------

# 24. Password di test

L'interfaccia attuale contiene valori precompilati per facilitare i
test:

``` text
Password 1 = 123
Password 2 = 456
Password 3 = 789
```

**Questi valori non devono essere utilizzati per proteggere dati
reali.**

------------------------------------------------------------------------

# 25. Licenza

Se il repository non contiene ancora una licenza specifica, questo
README non ne stabilisce una.

Aggiungere una licenza esplicita al repository se il progetto deve
essere distribuito, modificato o riutilizzato da terzi.

------------------------------------------------------------------------

# 26. Autore

**Powered by Benevolo B.**

Sito:

https://cruelben.github.io/

------------------------------------------------------------------------

## Stato del progetto

**M22C Secure Suite v2**

Caratteristiche attualmente presenti:

-   AES-256-GCM
-   PBKDF2-SHA-256 --- 600.000 iterazioni
-   64 frammenti logici
-   container dinamici
-   minimo 3 container
-   manifest cifrato
-   SHA-256 end-to-end
-   camouflage JPG opzionale
-   compatibilità legacy v1
-   rimozione metadati best-effort opzionale
-   elaborazione client-side
-   File System Access API + fallback download
