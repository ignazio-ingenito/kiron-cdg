# Kiron CDG: panoramica architetturale

**Stato:** Active  
**Ambito:** architettura logica e responsabilità dei componenti  
**Principio guida:** [RFC-0001](https://github.com/ignazio-ingenito/agent-os/blob/main/rfcs/RFC-0001-principles.md)

## Scopo

Questo documento definisce i confini architetturali di Kiron CDG e guida le prossime decisioni tecniche.

Lo stato `Active` indica che l'architettura logica è approvata e applicabile al lavoro corrente. I componenti ancora da verificare restano indicati come candidati e potranno essere confermati o scartati sulla base delle prove tecniche previste.

Il funzionamento del prodotto è descritto in [`product-overview.md`](product-overview.md).

## Decisioni confermate

- OutSystems è il frontend operativo.
- MariaDB `10.6.22-MariaDB-log` è l'unico database ed è self-managed dal cliente.
- La versione MariaDB presente dal cliente non è modificabile dal progetto. Deve essere verificato il corretto funzionamento dei componenti scelti su questa versione.
- Configurazioni, mapping, parametri e schedulazioni vengono gestiti in OutSystems e salvati in MariaDB.
- Le elaborazioni possono essere avviate manualmente o tramite Timer OutSystems.
- La risoluzione minima dei Timer è 5 minuti.
- MariaDB è il punto di pubblicazione di Actual e Forecast.
- BI Oracle accede direttamente ai dati pubblicati su MariaDB. Connessione, credenziali e rete verranno definite durante l'integrazione.
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

Non gestisce le schedulazioni e non contiene formule Kiron. L'implementazione verrà scelta insieme al motore di elaborazione.

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

SQLMesh è il candidato preferito. La compatibilità con MariaDB `10.6.22` deve essere dimostrata prima di confermarlo.

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

Actual e Forecast vengono esposti tramite tabelle o viste stabili su MariaDB. BI Oracle accede direttamente a questi dati.

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

## Decisioni e verifiche ancora aperte

- compatibilità effettiva di SQLMesh con MariaDB `10.6.22`;
- modalità di acquisizione da Campus, Campus 2.0 e Infinity;
- volumi e tempi attesi delle elaborazioni;
- dettagli tecnici di accesso di BI Oracle;
- formule, riconciliazioni e ribaltamenti ancora da definire;
- utilità effettiva di GoRules ZEN.

Questi punti non modificano i confini architetturali approvati. Ogni scelta tecnica che ne dipende verrà presa dopo la relativa verifica.

## Gate tecnici

### SQLMesh e MariaDB

La prima prova deve rispondere a questa domanda:

> SQLMesh può collegarsi a MariaDB `10.6.22`, eseguire un modello semplice e uno incrementale, applicare un audit e gestire correttamente gli intervalli senza workaround sproporzionati?

La prova include solo:

- connessione tramite l'adapter MySQL;
- un modello semplice;
- un modello incrementale per periodo;
- un audit;
- una seconda esecuzione sugli stessi intervalli;
- raccolta delle incompatibilità riscontrate.

L'esito sarà `SÌ`, `NO` o `PARZIALE`. Nel terzo caso, ogni workaround verrà valutato contro RFC-0001.

### GoRules ZEN

La validazione verrà aperta dopo l'ingestione dati e prima di sviluppare formule e ribaltamenti. Userà casi reali Kiron e gli stessi criteri di semplicità, beneficio verificabile e costo di integrazione.

## Aggiornamento del documento

Il documento verrà aggiornato quando una verifica modifica una scelta architetturale, conferma un candidato o ne determina l'esclusione. I dettagli di implementazione che non cambiano i confini restano fuori da questa panoramica.