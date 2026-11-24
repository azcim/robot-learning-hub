---
tags:
---



TODO








# WAM e World Models

> Non sono un'idea nuova: sono il **model-based RL** proiettato in un mondo fatto di pixel e parole.

---

## Da dove vengono: due punti di partenza in crisi

- **Model-based RL classico** — l'agente conosce (o impara) dinamica e reward e **pianifica simulando il futuro nella propria mente**.
	- *Limite:* presuppone uno stato descritto da pochi numeri. Con telecamere e linguaggio non regge.
- **[[VLA]] classico** — salta del tutto la modellazione del mondo, mappa percezione → azione.
	- *Limite:* **cieco alla causalità fisica**. Il VLM ha visto miliardi di foto statiche, mai il movimento causato dalle azioni: deve imparare la fisica da zero con pochissimi dati robotici.

**I world model fondono i due:** l'idea di simulare il futuro, resa capace di digerire la complessità multimodale.

> [!note] Nota storica
> Il model-free ha prevalso soprattutto perché è **più semplice** e perché in simulazione l'esperienza è quasi gratis. In robotica reale l'esperienza costa tantissimo: l'efficienza campionaria del model-based è tornata interessante.

---

## L'evoluzione in quattro passi

### 1. Nello spazio dei pixel

Se lo stato è un'immagine, il modello deve generare l'immagine successiva. Generare pixel fotorealistici da zero è insostenibile e incline ad allucinazioni.

Intuizione (Finn, Goodfellow, Levine, 2016): predire un **campo di flusso visivo** che *deforma* l'immagine attuale. Modellare il moto è molto più facile che generare texture coerenti.

*Limite:* calcolare i pixel resta costoso, e gran parte dell'immagine (sfondo, luce) è ridondante per il controllo.

### 2. Nello spazio latente

Si comprime l'osservazione in un **vettore latente compatto** e tutta la simulazione avviene lì. Tre moduli:

**encoder** → **modello di dinamica** (stato nascosto ricorrente; predice il latente successivo e la **reward immaginata**) → **policy**

> [!important] Il vantaggio enorme
> Addestrati encoder e dinamica su qualche dato reale, **la policy si addestra interamente in immaginazione**, senza altre interazioni con l'ambiente.

*World Models* (Ha e Schmidhuber, 2018): VAE + RNN/MDN + un controller volutamente minimale (un solo layer fully connected). La filosofia: se il world model fa bene il lavoro pesante, al controllore non serve profondità.

### 3. RSSM e la famiglia Dreamer

**Il problema — *latent drift*.** In immaginazione la RNN riceve latenti *campionati* (quindi rumorosi) e li usa per aggiornare la memoria: l'errore si avvita e in pochi passi l'immaginazione esce dalla regione di spazio che il modello capisce.

**RSSM** (PlaNet, 2019) inverte l'ordine:

- *Tempo 1:* la memoria si aggiorna in modo **deterministico**, senza campionare nulla → resta pulita
- *Tempo 2:* solo dopo, il prior campiona il nuovo latente da quella memoria sicura
- Una **loss KL** tiene il prior vicino al posteriore reale, così l'allucinazione resta regolata e confinata
- Solo il **primissimo latente** viene dalla telecamera vera

**Dreamer V1** (2020) — **pianificazione ammortizzata**: invece di rilanciare una ricerca a ogni passo come PlaNet, allena actor-critic nei sogni sfruttando la differenziabilità dell'RSSM. A runtime il robot non simula più nulla: interroga una policy che risponde istantaneamente.

**DayDreamer** (2022) — lo porta sul robot vero a parità di iperparametri: **un'ora di interazione fisica** basta a un quadrupede per imparare a camminare da zero. Architettura rigorosamente **asincrona**: thread di controllo ad alta frequenza + thread di ottimizzazione in background.

**Dreamer V4** (2025) — due pilastri: **pre-training disaccoppiato su milioni di video web** senza etichette d'azione (flow matching), e la RNN sostituita dalla **KV cache di un Transformer** per una memoria attenzionale a lunghissimo termine.

### 4. Video generativi come world model

→ [[Self-supervised su video#Il problema della tokenizzazione]]

---

## Il vocabolario

| Sigla | Che cos'è | Dove stanno le azioni | Esempi |
| --- | --- | --- | --- |
| **WFM** (World Foundation Model) | modello generativo su **tokenizer spazio-temporali** (3D VAE, video tokenizer causali), addestrato a **predire i token video futuri**; architettura tipica il **DiT** | nessuna, o come condizionamento | NVIDIA Cosmos, Google Genie |
| **AC-WM** (Action-Conditioned WM) | il world model classico | **in ingresso**: dai osservazioni e azioni, ottieni gli stati futuri | PlaNet, Dreamer |
| **WAM** (World Action Model) | un modello unico che **genera congiuntamente** video futuro e azioni che lo producono | **in uscita** | NVIDIA Isaac GR00T 2, DreamZero |
| **VAM** (Video Action Model) | **congela** un backbone video pre-addestrato e ci addestra sopra un **IDM leggero** | in uscita, solo dalla testa | mimic-video |

---

## Perché un WFM non è "un VLM con una testa che genera immagini"

| | **VLM** | **WFM** |
| --- | --- | --- |
| Costruito attorno a | un LLM autoregressivo | **tokenizer visivi spazio-temporali** |
| Cosa fa l'encoder | comprime l'immagine in pochi token semantici, **scartando la geometria fine** | comprime blocchi interi di frame **preservando continuità temporale, profondità e dinamica** |
| Loss | predire il **prossimo token di testo** | ricostruire e predire i **token video futuri** |
| Ruolo del testo | è l'output | solo condizionamento opzionale |

**Il DiT (Diffusion Transformer).** Invece delle vecchie U-Net convoluzionali, suddivide i dati spazio-temporali in una sequenza di token ed elabora con blocchi Transformer (self- e cross-attention), imparando a rimuovere progressivamente il rumore da sequenze video future. Permette di **scalare su grandi dataset** e di catturare dipendenze fisiche a lungo raggio.

**Il vantaggio chiave:** obbligando il modello a predire l'osservazione futura, il WFM **non può barare** con scorciatoie semantiche. Deve apprendere geometria 3D, attrito, permanenza degli oggetti e rapporti causa-effetto.

---

## VLA vs WAM: reattivo contro predittivo

- **VLA con action chunking e flow matching** (π0, GR00T N1) — genera un blocco di N azioni future, **ma basandosi esclusivamente sull'osservazione a t₀**.
	- Se un oggetto scivola, si deforma o urta un ostacolo al passo t₅, la traiettoria resta **cieca** fino al ciclo di osservazione successivo.
	- E i pesi del modello non sono mai stati vincolati a ricostruire la fisica: solo a imitare traiettorie.
- **WAM** (GR00T 2, DreamZero) — a runtime genera **congiuntamente**, spesso in un unico processo di denoising, i token dei **fotogrammi futuri** e i token delle **azioni**.
	- **Coerenza fisica forzata** — le azioni proposte devono essere compatibili con l'evoluzione visiva dell'oggetto. Il modello "immagina" l'oggetto che si piega mentre calcola la forza delle dita.
	- **Generalizzazione zero-shot da video umani** — un VLA impara solo da dati con etichette d'azione; un WAM assorbe milioni di video senza etichette motorie per imparare la dinamica, poi usa poche demo robotiche per **allineare** le azioni a quella fisica.

> [!important] La differenza in una riga
> Nel VLA il decoder video viene **buttato via** a fine training — a runtime il modello guarda un frame e sputa azioni. Nel WAM **la generazione video resta attiva a runtime**.

---

## Il nuovo ruolo della simulazione

I WFM spostano la simulazione **dentro i pesi della rete**.

1. **Simulazione data-driven invece che hand-crafted** — il WFM impara le leggi fisiche osservando milioni di ore di video. Non calcola equazioni di attrito su una mesh: "sa" come cade un oggetto o come si piega un tessuto perché ha appreso la distribuzione statistica delle dinamiche reali.
2. **Simulatore come generatore di mondi (offline)** — Cosmos e Genie sostituiscono o affiancano Isaac Lab generando infinite varianti fotorealistiche da un prompt o da un singolo video reale (**video-to-sim**, come anche Google Gemini 4 Argon). → [[Simulazione#Video-to-sim e sim-to-real]]
3. **Immaginazione interna in tempo reale (WAM)** — la simulazione diventa il **processo cognitivo interno del robot**: prima di eseguire il blocco di azioni, il robot fa una micro-simulazione mentale per verificare che la traiettoria sia fisicamente valida.
	- *Nota tecnica:* per non rallentare il robot, i WAM moderni usano **distillazione a 1-4 step** di flow matching, oppure predicono **nello spazio latente compresso** del 3D VAE senza decodificare i pixel reali.

---

## Limiti e vantaggi

**Vantaggi** — pre-training su dati video virtualmente illimitati; efficienza campionaria del model-based recuperata; comprensione della **dinamica e della causalità fisica**; coerenza fisica delle azioni generate; simulazione data-driven senza modellare mesh e attriti a mano.

**Limiti** — i WAM restano **pesanti da eseguire** e richiedono molta ingegneria per girare in tempo reale; il **compounding error** resta un rischio, contenuto sostituendo i frame generati con l'osservazione reale a ogni passo (*closed loop*); rischio di allucinazione; il costo si sposta sul **pre-training**, che però si fa una volta sola.

## Hardware

- **Training da zero di un WFM: fuori portata** per chiunque non sia un grande laboratorio (fino a 14B su milioni di video). La via realistica è partire da checkpoint esistenti (Cosmos) e fare fine-tuning, o costruirci sopra un **VAM**.
- **Generazione dati offline con Cosmos:** carico pesante ma *batch* — tollera di girare la notte su workstation da 48 GB.
- **Inferenza WAM a bordo:** è il caso d'uso che giustifica un **Jetson Thor** invece di un Orin. Il collo di bottiglia reale è la **banda di memoria** (273 GB/s su Thor e Spark). → [[Hardware]]

---

## Collegato

[[Self-supervised su video]] · [[VLA]] · [[Simulazione]] · [[Dati]] · [[Reinforcement Learning]]
