---
tags:
---
TODO

- piattaforme
- teleoperazione in simulazione
- sim2real
- integrazione con wfm e video 2sim












# Simulazione

> Non è un paradigma né un modello: è **l'ambiente in cui girano paradigmi e modelli**.

Serve ogni volta che l'esperienza reale è troppo costosa, troppo lenta o troppo pericolosa: **quasi sempre nell'RL, spesso nella valutazione, raramente nell'IL puro**.

---

## Gli strumenti

**Isaac Sim** — rendering fotorealistico e fisica di base.

**Isaac Lab** (successore di Isaac Gym e Orbit) — il framework specifico per il robot learning:

- **ambienti paralleli accelerati su GPU** — è ciò che rende praticabile il PPO su locomozione
- pipeline native di RL **e** di IL
- **Isaac Lab Mimic** — data augmentation: moltiplica le dimostrazioni reali variando proceduralmente la scena
- motore fisico **Newton** in beta dalla 3.0, alternativa per task di precisione

**MuJoCo** (Google DeepMind) — ultraleggero, focalizzato sulla precisione dei contatti.

- prototipazione rapida
- validazione **sim-to-sim**: una policy addestrata in Isaac viene testata in MuJoCo per verificare che non abbia **sfruttato i bug (artefatti fisici) del primo simulatore**
- è un controllo che vale la pena fare sempre prima di toccare il robot vero

**Humanoid-Gym** (RobotEra) — RL di locomozione bipede. Include verifica sim-to-sim su MuJoCo, ma è **vincolato a Isaac Gym Preview 4, deprecato**: utile come riferimento logico (reward, domain randomization), non come base per nuovo codice.

---

## Creazione di scene vs utilizzo di WFM

È la frattura più interessante del momento.

| | **Simulazione esplicita (Isaac Sim, MuJoCo)** | **[[WAM e World Models\|World Foundation Model]] (Cosmos, Genie)** |
| --- | --- | --- |
| Come nasce la scena | modellata a mano: mesh, peso, attrito, collisioni, USD/URDF | generata da un prompt o da un singolo video reale (*video-to-sim*) |
| La fisica | equazioni di corpo rigido risolte da un motore | appresa dalla distribuzione statistica di milioni di video |
| Punto di forza | **precisione e ripetibilità** su corpi rigidi, controllo totale, reward calcolabili esattamente | **realismo visivo e varietà**, materiali difficili (tessuti, cavi, liquidi), nessun lavoro di modellazione |
| Punto debole | lunga modellazione manuale, visivamente finta, fragile su materiali deformabili | nessuna garanzia fisica esatta, allucinazioni, niente reward esatte, costo computazionale |
| Uso tipico | RL di locomozione, validazione, benchmark riproducibili | augmentation di dataset, generazione di varianti, riduzione del gap visivo |

> [!tip] In pratica non si sceglie: si combinano
> Isaac Lab per la fisica e le reward, un WFM per moltiplicare la varietà visiva e coprire scenari che sarebbe assurdo modellare a mano.

---

## Teleoperazione in simulazione

È lo snodo pratico che collega simulazione e [[Imitation Learning]].

Invece di teleoperare il robot vero (che occupa l'hardware, si usura e richiede presenza fisica), si teleopera il **gemello simulato** e si registrano episodi nello stesso formato. Lo strumento di riferimento è **LeIsaac**: teleoperazione dentro Isaac Lab, con i log salvati **nativamente in formato LeRobot**.

**Vantaggi concreti:**

- si raccolgono dati **in parallelo** su più istanze
- si può partire da **pose iniziali randomizzate** che sul robot vero sarebbero scomode da impostare
- si hanno **gratis le annotazioni perfette** (posa degli oggetti, successo/fallimento)

**Limite:** le dimostrazioni ereditano il gap visivo e di contatto del simulatore. Ottimo per il grosso del dataset, da integrare con demo reali per la rifinitura.

---

## Il ruolo in IL, RL, VLA e WAM

| | Ruolo della simulazione | Quanto è indispensabile |
| --- | --- | --- |
| **[[Imitation Learning]]** | teleoperazione in sim, Isaac Lab Mimic, valutazione riproducibile | **opzionale** — si può fare tutto sul robot vero |
| **[[Reinforcement Learning]]** | è l'ambiente stesso: milioni di rollout paralleli, reward calcolabili, nessun rischio hardware | **praticamente obbligatoria** |
| **[[VLA]]** | dati sintetici (SynGrasp-1B è interamente sintetico), augmentation del fine-tuning, valutazione scalabile | **molto utile**, non obbligatoria |
| **[[WAM e World Models\|WAM]]** | rapporto invertito: il world model **è** il simulatore | il simulatore esplicito diventa il **verificatore** |

---

## La valutazione scalabile: SIMPLER

Il problema: testare una policy generalista su decine di compiti sul robot vero è insostenibile.

- **deriva della calibrazione visiva** — basta urtare la telecamera di pochi millimetri
- **usura e guasti hardware** — sostituito un motore, le letture della propriocezione cambiano
- **lentezza** — un singolo tentativo di presa può richiedere minuti
- **mancanza di riproducibilità** — un esperimento di oggi è quasi impossibile da replicare domani, o altrove

**SIMPLER** (CoRL 2024) percorre il percorso **inverso** al sim-to-real: prende policy addestrate **esclusivamente su dati reali** e costruisce un ambiente specchio per testarle. Non punta al fotorealismo estremo, ma a chiudere due divari precisi:

1. **Divario di controllo** — *system identification* su pochissimi campioni reali tara i motori virtuali, così l'end-effector risponde agli input producendo traiettorie identiche a quelle fisiche.
2. **Divario visivo (visual matching)** — in due passaggi:
	- **green screening** — gli oggetti interattivi 3D vengono isolati e sovrapposti a **sfondi fotografici** presi da laboratori reali
	- **texture matching** — le immagini ad alta risoluzione degli oggetti veri vengono proiettate come texture sui modelli 3D

---

## Video-to-sim e sim-to-real

**Sim-to-real** — il problema classico. Tre strumenti:

- **domain randomization** — si variano proceduralmente massa, attrito, illuminazione, ritardi e rumore sensoriale, così la policy impara a essere **robusta** invece che tarata su un mondo preciso
- **system identification** — si misura il robot vero e si tarano i parametri del simulatore
- **validazione sim-to-sim** — su un secondo motore fisico

**Video-to-sim** — la direzione nuova: da un singolo video reale, un WFM genera un ambiente interattivo che ne riproduce aspetto e dinamica, **riducendo il gap all'origine** invece di coprirlo con la randomizzazione. Lo fanno Cosmos, Genie e Google Gemini 4 Argon.

> [!note] Osservazione collegata
> Un buon pre-training riduce la **necessità stessa di randomizzazione** durante il fine-tuning — cioè richiede meno dati. È uno dei vantaggi pratici di VLA e WAM.

---

## Vantaggi e limiti

**Vantaggi** — esperienza virtualmente infinita e a costo marginale nullo; nessun rischio per l'hardware; riproducibilità; annotazioni perfette gratis; possibilità di esplorare stati pericolosi.

**Limiti** — il gap sim-to-real non sparisce mai del tutto; fisica del contatto e materiali deformabili restano il punto debole; modellare le scene a mano è lento; rischio sottile che la policy **sfrutti i bug del simulatore** invece di imparare il compito; serve hardware serio.

## Hardware

- **Isaac Sim con GUI:** **48 GB di VRAM**, streaming fluido, sessioni lunghe. Workstation dedicata o SSH con streaming (WebRTC Omniverse, Sunshine/Moonlight).
- **Training RL in Isaac Lab:** **48 GB** — è il carico che satura davvero la GPU.
- **Vincolo:** **Isaac Sim su ARM gira solo su Linux**, quindi i portatili e mini-PC solo-Windows sono esclusi.

---

## Collegato

[[Reinforcement Learning]] · [[Framework]] · [[WAM e World Models]] · [[Hardware]] · [[Dati]]



