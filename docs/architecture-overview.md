# Kiron CDG: panoramica architetturale

**Stato:** Draft  
**Ambito:** architettura logica e responsabilità dei componenti  
**Principio guida:** [RFC-0001](https://github.com/ignazio-ingenito/agent-os/blob/main/rfcs/RFC-0001-principles.md)

## Scopo

Questo documento descrive i confini architetturali di Kiron CDG sulla base delle informazioni oggi disponibili.

È la fonte di riferimento per le responsabilità dei componenti e per le decisioni tecniche ancora da verificare. Non definisce infrastruttura, deployment, dimensionamento, API di dettaglio o schema fisico del database.

Per il funzionamento del prodotto si rimanda a [`product-overview.md`](product-overview.md).

## Decisioni confermate

- OutSystems è il frontend operativo.
- MariaDB `10.6.22-MariaDB-log` è l'unico database ed è self-managed.
- Configurazioni, mapping, parametri e schedulazioni vengono gestiti in OutSystems e salvati in MariaDB.
- Le elaborazioni possono essere avviate manualmente o tramite Timer OutSystems.
- La risoluzione minima dei Timer è 5 minuti.
- MariaDB è anche il punto di pubblicazione di Actual e Forecast.
- Non verrà sviluppato un linguaggio proprietario per descrivere le formule.

SQLMesh e GoRules ZEN sono candidati da validare, non componenti ancora confermati.

## Architettura logica

```mermaid
flowchart LR
    C[Campus] --> I[Ingestione dati]
    C2[Campus 2.0] --> I
    Z[Zucchetti Infinity] --> I

    O[OutSystems] -->|configura e richiede| M[(MariaDB)]
    O -->|Timer ogni 5 minuti| D[Dispatcher]

    D -->|registra richiesta| M
    D -->|avvia| E[Servizio di esecuzione]

    I --> M
    E --> X[Motore di elaborazione]
    X --> M

    M --> A[Actual gestionale]
    M --> AC[Actual consolidato]
    M --> F[Forecast]

    A --> O
    AC --> O
    F --> O
    M --> B[BI Oracle / dashboard]

    S[SQLMesh<br/>da validare] -. candidato .-> X
    G[GoRules ZEN<br/>validazione differita] -. formule e decisioni .-> X
```

## Responsabilità

### OutSystems

OutSystems gestisce:

- configurazioni, mapping e parametri;
- schedulazioni;
- avvio manuale delle elaborazioni;
- stato, anomalie ed esiti;
- consultazione di Actual e Forecast.

La logica di trasformazione e calcolo resta fuori da OutSystems.

### Timer e dispatcher

Un Timer OutSystems controlla ogni 5 minuti le schedulazioni attive in MariaDB. Quando trova un'elaborazione dovuta:

1. crea la richiesta, se non esiste già;
2. avvia il servizio di esecuzione;
3. termina senza attendere la fine del processo.

Le richieste manuali e schedulate seguono lo stesso percorso.

### MariaDB

MariaDB conserva:

- dati acquisiti e normalizzati;
- configurazioni, mapping, parametri e schedulazioni;
- richieste e stato delle elaborazioni;
- dati intermedi;
- Actual gestionale e consolidato;
- Forecast;
- anomalie, rettifiche, conguagli e audit;
- regole e relative versioni, quando saranno definite.

La separazione tra aree raw, normalizzate, di configurazione, output e audit è per ora logica. La struttura fisica verrà definita con il modello dati.

### Servizio di esecuzione

Il servizio separa le chiamate brevi di OutSystems dalle elaborazioni lunghe.

Deve:

- ricevere una richiesta;
- evitare prese in carico duplicate;
- avviare l'elaborazione con periodo e parametri richiesti;
- aggiornare stato ed esito in MariaDB;
- restituire subito l'accettazione della richiesta.

Non gestisce le schedulazioni e non contiene formule Kiron.

L'implementazione verrà scelta insieme al motore di elaborazione. Python è l'opzione più naturale se SQLMesh verrà confermato, ma non è ancora un vincolo architetturale.

### Motore di elaborazione

Il motore dovrà gestire:

- dipendenze tra elaborazioni;
- normalizzazione e trasformazioni;
- aggregazioni, stime e ribaltamenti;
- riconciliazioni e delta;
- produzione di Actual e Forecast;
- intervalli elaborati;
- audit e controlli sui dati;
- tracciabilità tecnica delle esecuzioni.

SQLMesh è il candidato preferito, ma il supporto ufficiale riguarda MySQL. La compatibilità con MariaDB deve essere dimostrata prima di confermarlo.

### GoRules ZEN

GoRules ZEN verrà valutato dopo l'ingestione dati, quando saranno disponibili formule e ribaltamenti reali.

La prova dovrà verificare che:

- rappresenti le regole senza introdurre un linguaggio proprietario;
- riduca sviluppo e manutenzione;
- mantenga precisione e tracciabilità adeguate;
- si integri senza workaround sproporzionati;
- sia adatto ai volumi da elaborare.

Se il vantaggio non sarà dimostrato, ZEN non verrà introdotto.

### BI Oracle e dashboard

Actual e Forecast vengono esposti tramite tabelle o viste stabili su MariaDB. La modalità tecnica di accesso della BI è ancora da definire.

## Flusso di esecuzione

```mermaid
sequenceDiagram
    participant U as Utente o Timer
    participant O as OutSystems
    participant M as MariaDB
    participant E as Servizio di esecuzione
    participant X as Motore di elaborazione

    U->>O: Richiede o individua l'elaborazione
    O->>M: Registra la richiesta se non esiste
    O->>E: Avvia la richiesta
    E-->>O: Richiesta accettata
    E->>M: Stato RUNNING
    E->>X: Esegue l'elaborazione
    X->>M: Scrive risultati e audit
    E->>M: Stato COMPLETED o FAILED
    O->>M: Legge stato ed esito
```

Gli stati iniziali sono `PENDING`, `RUNNING`, `COMPLETED` e `FAILED`. Altri stati verranno aggiunti solo in presenza di un requisito concreto.

## Vincolo sulla versione MariaDB

MariaDB `10.6.22` appartiene a una serie che non riceve più manutenzione Community. L'ultima patch della serie è la `10.6.27`.

Poiché l'istanza è self-managed, il progetto deve definire:

- la versione che intende supportare;
- il percorso di aggiornamento;
- la compatibilità con OutSystems e con le integrazioni esistenti;
- backup, monitoraggio e ripristino.

La versione target non viene scelta in questo documento.

Fonti:

- [Ciclo di vita delle versioni MariaDB](https://mariadb.org/about/)
- [Versioni MariaDB pubblicate](https://mariadb.org/mariadb/all-releases/)
- [Supporto MySQL di SQLMesh](https://sqlmesh.readthedocs.io/en/latest/integrations/engines/mysql/)

## Decisioni ancora aperte

- baseline MariaDB da supportare;
- compatibilità effettiva di SQLMesh con MariaDB;
- modalità di acquisizione da Campus, Campus 2.0 e Infinity;
- volumi e tempi attesi delle elaborazioni;
- modalità di accesso di BI Oracle;
- formule, riconciliazioni e ribaltamenti ancora da definire;
- utilità effettiva di GoRules ZEN.

Questi punti non bloccano la definizione dei confini logici, ma bloccano le scelte tecniche che ne dipendono.

## Prossimo gate

La prima verifica tecnica riguarda SQLMesh e MariaDB.

Un prototipo isolato deve rispondere a questa domanda:

> SQLMesh può collegarsi a MariaDB, eseguire un modello semplice e uno incrementale, applicare un audit e gestire correttamente gli intervalli senza workaround sproporzionati?

La prova deve includere solo:

- connessione tramite l'adapter MySQL;
- un modello semplice;
- un modello incrementale per periodo;
- un audit;
- una seconda esecuzione sugli stessi intervalli;
- raccolta delle incompatibilità riscontrate.

L'esito sarà `SÌ`, `NO` o `PARZIALE`. Nel terzo caso, ogni workaround dovrà essere valutato contro RFC-0001.

## Passaggio ad Active

Il documento potrà diventare `Active` quando saranno verificati almeno:

- baseline e percorso di aggiornamento di MariaDB;
- compatibilità del motore di elaborazione;
- una richiesta manuale end-to-end;
- una richiesta schedulata end-to-end;
- protezione dalle richieste duplicate;
- registrazione e consultazione dello stato;
- pubblicazione di un primo output su MariaDB;
- accesso della BI ai risultati.
