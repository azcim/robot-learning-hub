---
tags:
---

# VLA — Vision-Language-Action

> Un VLA è un'**architettura**, non un paradigma. Nasce dal limite strutturale di [[ACT]] e [[Diffusion Policy]]: sono specialisti di un singolo compito, e per ogni nuovo task o robot vanno riaddestrati.

---

## Da dove viene: la catena della tokenizzazione

Tutto ruota attorno a *qual è l'unità giusta da trasformare in token*.

- **Transformer** — elimina la ricorrenza, collega due posizioni qualsiasi con un percorso di lunghezza costante. Risolve i tre limiti delle RNN: dipendenze a lungo raggio, parallelizzazione, collo di bottiglia a dimensione fissa.
- **LLM** — testo tokenizzato con **BPE** (sottoparole: compromesso fra vocabolario enorme e sequenze lunghissime). Decoder-only, BPE a livello di byte, **RoPE** per le posizioni relative.
- **ViT** — le **patch 16×16** di un'immagine diventano token, appiattite e proiettate linearmente, con un embedding di posizione.
- **CLIP** — risolve l'**allineamento multimodale**: due encoder proiettati in uno spazio condiviso con loss contrastiva, così il vettore dell'immagine di una mela e quello della parola "mela" convergono nello stesso punto.
- **VLA** — la domanda successiva: *possiamo tokenizzare anche le azioni?* Se sì, un intero episodio robotico diventa **una sequenza di token eterogenei** che un unico Transformer processa nello stesso spazio.

**Il motivo di fondo:** un robot lavora quasi sempre in un **POMDP** — non vede lo stato completo, solo osservazioni parziali. Da un singolo fotogramma non si stima una velocità, e decisioni prese una alla volta producono traiettorie a scatti (*jitter*). Serve ragionare sulla **storia**: è un problema di sequence modeling.

---

## Due scuole per far parlare un VLM in azioni

### Scuola 1 — Azione come testo (binning)

*RT-2, OpenVLA.*

- **Normalizzazione per quantili** — si usa il 1° e il 99° percentile, non min/max assoluti: un singolo movimento brusco allargherebbe la scala a dismisura, schiacciando tutti i movimenti normali nei primi bin.
- **Divisione in 256 bin** sull'intervallo [−1, +1] → ogni valore continuo diventa un intero da 1 a 256, cioè un **ID di token**.
- **Un token per dimensione** — un braccio a 6 assi + pinza produce 7 ID per istante.
- **Sovrascrittura del vocabolario** — i 256 bin sostituiscono i **256 token meno frequenti** dell'LLM. Non si aggiungono token nuovi: allungare la matrice di embedding romperebbe la compatibilità coi pesi pre-addestrati e rallenterebbe la softmax. **L'architettura resta identica al 100%.**
- **Pro:** elegante, nessuna modifica architetturale. **Contro:** la discretizzazione riduce la precisione, e la generazione sequenziale rende l'inferenza troppo lenta per il tempo reale.

#### Perché il binning fallisce ad alta frequenza — e FAST

A 20-50 Hz il robot si muove pochissimo fra due passi → il binning arrotonda i micro-spostamenti **sempre sullo stesso intero** → i vettori d'azione consecutivi diventano **identici**.

> [!warning] La scorciatoia deleteria
> L'informazione marginale va a zero, e il modello **smette di guardare il contesto e fa copia-incolla dell'ultimo token**, ottenendo un errore bassissimo senza aver capito nulla del compito.

**FAST** (RSS 2025) prende in prestito la **DCT**, la trasformata usata per comprimere i JPEG:

1. **chunk** — si isola ~1 secondo di movimento (es. 50 micro-comandi) e si normalizza
2. **DCT** — dal dominio del tempo a quello delle frequenze. I movimenti fluidi sono "lisci", quindi concentrano l'energia nelle **basse frequenze**
3. **quantizzazione** — i coefficienti diventano interi
4. **appiattimento** — prima le basse frequenze, le più ricche di significato
5. **BPE finale** — fonde le lunghe scie di zeri in **un unico token compatto**

### Scuola 2 — Action Expert dedicato (flow matching). **È l'approccio dominante**

- Separazione netta dei compiti: il **VLM** fa percezione e ragionamento semantico, un **modulo separato e leggero** traduce le sue feature in azioni continue ad alta frequenza.
- **π0 / π0.5** (Physical Intelligence): VLM da ~3B + action expert da ~300M basato su **flow matching**.
- **Knowledge insulating** — in π0.5 l'expert attinge alle rappresentazioni del VLM tramite un **flusso unidirezionale**: questo **protegge il VLM durante il fine-tuning**, evitando il *catastrophic forgetting*.
- **Fused attention** — l'Action Expert non sta a valle ma è **integrato dentro ogni layer** del Transformer. Conseguenza: l'architettura diventa **agnostica rispetto alla configurazione delle telecamere** — una o cinque, i token visivi si aggiungono e l'expert impara a quali dare priorità.
- **Co-training con dati web** — continuare ad alimentare il modello con testi e immagini normali durante l'addestramento robotico evita che dimentichi la sua intelligenza linguistica e visiva.

---

## I dati: dal generalismo allo specifico

**Tre fasi, tre tipi di dati, tre paradigmi diversi.**

| Fase | Dati | Paradigma | Cosa si ottiene |
| --- | --- | --- | --- |
| **Pre-training** | video web + testo, video egocentrici, OXE / DROID / AgiBot World / RoboMIND, dati sintetici | [[Self-supervised su video]] + BC su scala | prior semantici e motori generici; baseline su task ed embodiment mai visti |
| **Fine-tuning** | 50-500 demo sul **tuo** robot, nel **tuo** ambiente, sul **tuo** task | [[Imitation Learning]] | competenza specifica, affidabilità sul caso d'uso |
| **Post-training** | rollout del robot con criterio di successo verificabile | [[Reinforcement Learning]] | superamento dell'esperto, recupero dagli errori (π\*0.6 / RECAP) |

I due accorgimenti che governano la transizione: **mescolare dati web** durante l'addestramento robotico, e **isolare il backbone** durante il fine-tuning.

> [!important] Il fine-tuning è Behavior Cloning
> Coppie (immagine + istruzione, azione teleoperata) e una loss supervisionata. Vale per intero il problema del covariate shift: le demo devono essere **pulite, fluide e consistenti**, perché il modello impara anche le esitazioni dell'operatore.

---

## Il panorama dei modelli

**Generalisti addestrati da zero su dati robotici:** → [[Octo e CrossFormer]]

**VLA con prior di internet:**

| Modello | Chi | Caratteristica |
| --- | --- | --- |
| **OpenVLA** / **OpenVLA-OFT** | Stanford et al. | riferimento open con azione tokenizzata; OFT ottimizza velocità e success rate |
| **π0 / π0.5** | Physical Intelligence | standard dell'approccio action-expert + flow matching; codice in **openpi** |
| **GR00T N1 / N1.7** | NVIDIA | foundation model per umanoidi; azioni latenti e IDM; N1.7 su ~20.000 h di egocentrico |
| **SmolVLA** | Hugging Face | VLA piccolo ed economico — l'opzione quando il vincolo è la VRAM di bordo |
| **UniVLA / WorldVLA / XR-1** | vari | rappresentazioni da video, unificazione visione-movimento |
| **π\*0.6 (RECAP)** | Physical Intelligence | un VLA che impara dall'esperienza: il post-training RL nel ciclo di vita |

**Frontiera e proprietari:**

- **GraspVLA** (Galbot + Univ. Pechino, CoRL 2025) — foundation model di grasping pre-addestrato su **un miliardo di frame sintetici**. Dimostra che per un *singolo skill ben definito* il sintetico scala meglio della teleoperazione.
- **OmniVTLA** (Paxini + SJTU) — integra il **tatto**: Vision-Tactile-Language-Action.
- **NeuroVLA** (2026) — ispirato al cervello, punta su controllo fluido e **riflessi rapidi agli imprevisti**: esattamente il punto debole dei VLA a chunk.
- **Stack verticali dei produttori** — *Huisi Kaiwu* (X-Humanoid), *AstraBrain* (Galbot).

---

## Quando usarli

- **Sì:** molti compiti, oggetti e scene variabili, istruzioni in linguaggio naturale. Un task nuovo costa un **fine-tuning**, non un modello nuovo.
- **Sì:** generalizzazione a oggetti mai visti, dove ACT fallisce.
- **No:** un singolo compito ripetitivo a ciclo fisso in ambiente controllato — lì ACT o Diffusion Policy vincono per semplicità, latenza e costo.

> [!warning] Il limite di fondo
> L'action chunk viene generato sull'osservazione a t₀ ed è **cieco** a ciò che succede nel frattempo. È il problema che i [[WAM e World Models|WAM]] vogliono risolvere.

---

## Limiti e vantaggi

**Vantaggi** — pre-training che permette di generalizzare meglio e di superare in qualità le traiettorie esperto disponibili; **minore necessità di randomizzazione e meno dati nel fine-tuning**; comprensione del linguaggio; con [[Embodied Reasoning|ECoT]], interpretabilità e correzione online.

**Limiti** — il fine-tuning resta task-specific; **ciechi alla causalità fisica** (il backbone ha visto coppie statiche immagine-testo, mai il movimento causato dalle azioni); rischio di catastrophic forgetting; latenza di inferenza; costo di training molto superiore all'IL classico.

## Hardware

- **Fine-tuning:** **48 GB di VRAM con LoRA**; cloud per le run più grandi.
- **Inferenza:** **8-16 GB**, con **latenza bassa e costante**. VLA da 2-3B → RTX 5080 mobile (16 GB); da 7B → RTX 5090 mobile (24 GB). Sul robot: Orin è **lento per i VLA**, Thor è l'opzione pensata per stare a bordo. → [[Hardware]]

---

## Collegato

[[Self-supervised su video]] · [[Imitation Learning]] · [[WAM e World Models]] · [[Embodied Reasoning]] · [[Octo e CrossFormer]] · [[Dati]]
