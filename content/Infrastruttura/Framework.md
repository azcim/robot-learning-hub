---
tags:
  - robot-learning
  - infrastruttura
---

# Framework e stack software

L'ecosistema è modulare, con **due pilastri** che coprono più strati dello stack — **LeRobot** e **Isaac Lab** — e strumenti verticali che si innestano quando servono modelli specifici o simulazione avanzata.

| Framework | A cosa serve | Quanto è aperto |
| --- | --- | --- |
| **LeRobot** (Hugging Face) | registrazione, dataset, training e deploy di policy in PyTorch | completamente open, `pip install lerobot` |
| **Isaac Sim / Isaac Lab** (NVIDIA) | simulazione, RL, IL, data augmentation | codice open, runtime NVIDIA proprietario, richiede GPU NVIDIA |
| **Isaac-GR00T** (NVIDIA) | fine-tuning, export TensorRT e valutazione open-loop di GR00T | open, modello con licenza propria |
| **openpi** (Physical Intelligence) | famiglia π0 / π0.5, in JAX con porting PyTorch | open |
| **MuJoCo** (Google DeepMind) | prototipazione rapida e validazione sim-to-sim | completamente open |
| **Humanoid-Gym** (RobotEra) | RL di locomozione bipede | open ma **legato a Isaac Gym Preview 4, deprecato** |
| **Huisi Kaiwu** (X-Humanoid), **AstraBrain** (Galbot) | stack verticali pre-installati sui robot | **proprietari** |

---

## 1. Come registrare i dati

- **`lerobot-record`** gestisce teleoperazione e raccolta dati dall'hardware.
- **Astrazione hardware:** la classe nativa `Robot` avvolge i driver fisici.
	- supporta nativamente dai bracci economici a umanoidi come l'**Unitree G1**
	- per **Tien Kung** o **TORA-ONE** serve sviluppare un **wrapper custom sopra l'SDK del produttore** → è il primo costo di integrazione da mettere a budget per ogni robot nuovo
- **In alternativa: LeIsaac** registra la teleoperazione dentro Isaac Lab salvando direttamente in formato LeRobot. Stessa pipeline a valle, dati sintetici invece che reali. → [[Simulazione#Teleoperazione in simulazione]]

## 2. Come gestire i dati

- I formati sono in [[Dati#Formati di archiviazione]]. La scelta pratica è una sola: **adottare LeRobot v3 come standard interno.**
- Perché: `lerobot-train` allena modelli diversi **sullo stesso formato**, quindi cambiare modello non costa rifare la pipeline dati.
- **Il costo nascosto:** Isaac-GR00T usa **LeRobot v2** + `modality.json` + *embodiment tag*. Ogni formato parallelo è una conversione da manutenere.

## 3. Librerie o implementazione locale?

> [!tip] Risposta pratica: librerie, salvo motivo esplicito
> `lerobot-train` allena **ACT, Diffusion Policy, SmolVLA, π0 e GR00T N1.7** modificando solo il file di configurazione della policy. È la leva che rende **confrontabili gli esperimenti** e riutilizzabile il lavoro sui dati.

**L'implementazione locale ha senso in tre casi:**

1. si modifica l'architettura (una testa nuova, un sensore in più come il tatto)
2. si integra un robot non supportato (il wrapper SDK)
3. si ottimizza l'inferenza per l'hardware di bordo

Per i modelli di punta esistono pipeline ufficiali da usare come punto di partenza: **Isaac-GR00T** (training, export TensorRT, valutazione open-loop) e **openpi** per π0.

## 4. Test in simulazione o deploy reale diretto?

La sequenza sana è a tre stadi; saltarne uno costa di solito più tempo di quanto ne faccia risparmiare.

1. **Valutazione open-loop** sul dataset di validazione — costa niente, intercetta subito i problemi grossolani (normalizzazione sbagliata, frequenze non allineate)
2. **Valutazione in simulazione** — `lerobot-eval`, benchmark tipo **LIBERO** o **RoboDojo**, o un ambiente specchio in stile [[Simulazione#La valutazione scalabile SIMPLER|SIMPLER]]. Per le policy RL, in più, la **verifica sim-to-sim in MuJoCo**
3. **Deploy reale** — prima su workstation cablata, poi su hardware di bordo

> [!note] L'eccezione ragionevole
> Una policy IL addestrata su demo raccolte **su quel robot, in quella scena**, può andare direttamente in test reale: non c'è nessun gap di dominio da verificare. **Il gap sim-to-real esiste solo se c'è stata simulazione.**

## 5. Architettura di inferenza e deploy

Sul robot reale si usa un modello **client-server disaccoppiato**, per proteggere l'hardware di bordo dal carico dei VLA.

```
┌──────────────────────────────┐              ┌──────────────────────────────┐
│  Server di Inferenza (GPU)   │   ROS 2      │       Robot Client           │
│  • Esegue il modello VLA     │ ──────────►  │  • Invia Video e Stato       │
│  • Genera l'Action Chunk     │ ◄──────────  │  • Esegue comandi motore     │
└──────────────────────────────┘              └──────────────────────────────┘
```

Il middleware standard è **ROS 2**. I server di inferenza (quello nativo di GR00T, il client asincrono di LeRobot) espongono API per ricevere le osservazioni e restituire chunk di azioni ad alta frequenza.

### Il dettaglio di rete che si paga caro se ignorato

Su Wi-Fi i dispositivi non si parlano direttamente: **tutto passa dal router**.

- PC via cavo + robot via Wi-Fi → **un solo tratto in aria**
- PC e robot entrambi via Wi-Fi → **due tratti, sullo stesso canale**
- Dove serve mobilità: router dedicato e vicino. Dove non serve: **il cavo vince sempre**
- Gli umanoidi (Paxini forse escluso) e il visore Meta Quest 3 supportano al massimo **Wi-Fi 6** → robot sulla banda 5 GHz, visore sulla 6 GHz

---

## Collegato

[[Simulazione]] · [[Hardware]] · [[Dati]] · [[Percorsi end-to-end]]
