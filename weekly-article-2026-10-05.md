# `FailoverLlmClient`: la catena di generatori di riserva

## 1. Dov'è, a cosa serve, chi la usa

In ARA ogni volta che un agente "pensa" passa da **`LlmClient`**, l'interfaccia verso il modello (OpenAI, Claude, Ollama…). In produzione, però, un modello può cadere: un timeout, un 5xx, un limite di richieste. **`FailoverLlmClient`** (`ara-runtime`, package `io.ara.runtime.factory`) è il decoratore che mette più client **in fila per priorità**: prova il primo e, se non risponde, passa al successivo, senza che l'agente se ne accorga.

**Chi la usa?** Nessuno la chiama a mano. La costruisce `DefaultLlmRouter` da solo, quando l'agente dichiara `LlmSelectionPolicy.FAILOVER` e una lista di `fallbackLlms(...)`. Poi le strategie di esecuzione (`ReactStrategy` e sorelle) la interrogano come se fosse **un unico modello**, chiamando `complete()` o `stream()`.

**Perché serve?** Perché "il modello è giù" non deve mai essere un errore mostrato all'utente: è un caso previsto.

## 2. Una metafora per tenere il filo

Una **batteria di generatori di un ospedale**. Davanti c'è la rete elettrica (il modello principale) e, dietro, generatori numerati. Al primo blackout parte il generatore 1; se anche quello non regge, il 2. Il centralino annota **quale sorgente sta alimentando la sala**, così chi indaga sa chi ha tenuto. E una regola ferrea: se la luce salta **a operazione iniziata**, non si ricomincia l'intervento dall'inizio — si accetta l'interruzione, perché riavviare significherebbe rifare tutto ciò che è già stato fatto.

## 3. Il flusso: cosa entra, cosa succede, cosa esce

**Input.** I messaggi e le opzioni della chiamata (`LlmCallContext`), più la lista di client già **in ordine di priorità**.

**Processo.** `complete()` li prova uno dopo l'altro. Un errore **deterministico** (401, richiesta non valida) ferma subito la catena; un errore **ritentabile altrove** (rete, 5xx, rate limit) fa passare al candidato successivo. Dopo ogni fallimento controlla se il thread è stato **interrotto** (scadenza superata) e, in tal caso, si ferma invece di proseguire. Sul percorso in streaming (`stream()`) la regola si stringe: si può cambiare fornitore **solo prima del primo token**.

**Output.** Il `LlmCompletion` del primo che risponde, oppure l'ultima eccezione. In più, con la telemetria attiva, uno span `llm.failover` che dice com'è finita: `served_by_primary`, `served_by_fallback`, `aborted_non_failover`, `exhausted` o `interrupted`.

## 4. Le scelte, e i perché

**"Ritentare qui" ≠ "provare altrove".** Il pool non guarda `isRetryable()` ma un flag separato, `shouldFailover()`: un errore di connessione non va riprovato sullo *stesso* endpoint, ma la catena deve poter avanzare su un *altro*. Sono due domande diverse, e meritano due risposte diverse.

**Un 401 non si moltiplica.** Davanti a un errore che si ripeterebbe identico su ogni candidato, la catena si ferma: cambiare modello non cambierebbe l'esito, solo il rumore nei log.

**Niente doppioni in streaming.** Una volta emesso il primo token, passare al fornitore successivo riscriverebbe da capo ciò che l'utente ha già visto. Meglio un fallimento pulito che un testo ripetuto.

**La scadenza non si amplifica.** Ogni candidato ha il proprio timeout: senza il controllo sull'interruzione, un watchdog scaduto farebbe pagare l'attesa a *tutti* i candidati rimasti. Interrompere il filo ferma la catena al primo colpo.

**Si promette solo ciò che tutti mantengono.** `supportedMediaTypes()` e `supportsNativeTools()` valgono `true` solo se *ogni* candidato li supporta: altrimenti un ripiego solo-testo risponderebbe su un PDF che non ha mai ricevuto. Meglio un errore immediato, dal primo candidato, che una risposta sbagliata e silenziosa.

**Perché un decoratore, e non dentro le strategie.** Tenendo la resilienza separata dall'intelligenza, l'agente resta semplice e le capacità si **compongono**: aggiungere un `CircuitBreakerLlmClient`, un nuovo fornitore o una chain diversa è una questione di *configurazione*, non di codice dell'agente.

---

*Classe: `io.ara.runtime.factory.FailoverLlmClient` — `ara-runtime`. Attivazione: `LlmSelectionPolicy.FAILOVER` + `fallbackLlms(...)`.*
