---
tags:
  - robot-learning
  - infrastruttura
---

# Hardware

## Quadro di sintesi

| Attività | VRAM richiesta | Dove gira |
| --- | --- | --- |
| Simulazione Isaac Sim con GUI | **48 GB** | workstation dedicata, o SSH + streaming (WebRTC Omniverse, Sunshine/Moonlight) |
| Training RL in Isaac Lab | **48 GB** | workstation via SSH |
| Training IL (ACT, Diffusion Policy) | **24 GB** | workstation via SSH |
| Training modelli di visione (es. YOLO) | **24 GB** | workstation via SSH |
| **Fine-tuning VLA** | **48 GB con LoRA** | workstation; cloud per le run più grandi |
| Inferenza ACT / Diffusion Policy | pochi GB | computer di bordo, laptop, nodo cablato — indifferente |
| **Inferenza VLA** | **8-16 GB**, latenza bassa e **costante** | Thor / DGX Spark esterno, laptop con GPU, workstation cablata |

---

## Le opzioni per l'inferenza sul robot

### 1. Computer di bordo (Jetson AGX Orin)

È ciò che montano X-Humanoid Tian Yi 2.0, PaXini Tora Double One e Galbot S1.

- **Pro:** migliore per la **mobilità**, non vincolata a cavi; basta per ACT e Diffusion Policy.
- **Contro:** **lento per i VLA**; la potenza è **condivisa col software del produttore** (controllo, sensori, sicurezza), quindi non tutta disponibile per le policy; mettere il proprio codice sull'hardware proprietario del robot pone un tema di isolamento.

### 2. Jetson AGX Thor esterno, montato sul robot

*5-6 k$*

- 128 GB di memoria unificata, **~3× più veloce di un Orin, 2-3× più lento di una GPU desktop**
- **consumo 40-130 W** (alimentatore 140 W, alimentabile a batteria con la tensione giusta)
- JetPack 7 con Ubuntu 24.04, Isaac ROS, ricette ottimizzate per GR00T
- 4×25 GbE + USB per camere aggiuntive (es. camera da polso per la raccolta dati)
- È l'opzione pensata per stare a bordo **e** mantenere il codice isolato dallo stack del produttore

### 3. DGX Spark

Stessa fascia di Thor in inferenza VLA.

- 128 GB (esiste anche in 64 GB) e **stessa banda di memoria, 273 GB/s** — che per i VLA è **il vincolo reale**
- Thor dichiara più potenza FP4 (2.070 TFLOPS contro ~1.000); Spark ha CPU più potente e stack DGX OS per sviluppo e fine-tuning di modelli grandi
- **Pensato per la scrivania:** ~15×15×5 cm, 1,2 kg, silenzioso, Wi-Fi 7 — non progettato per stare su un robot

### 4. Laptop con GPU

| Uso | GPU | VRAM | Riferimento |
| --- | --- | --- | --- |
| Teleoperazione + ACT / Diffusion Policy | RTX 5070 mobile | 12 GB | Lenovo Legion Pro 5i (~3.000 $) |
| + VLA da 2-3B, Isaac Sim al limite | RTX 5080 mobile | 16 GB | Lenovo Legion Pro 7i (~4.000 $) |
| + VLA da 7B, Isaac Sim | RTX 5090 mobile | 24 GB | Lenovo Legion Pro 7i (~5.800 $) |

### 5. Nodo robot cablato (workstation + switch)

L'opzione **più veloce**, perché usa una GPU desktop, e tipicamente già pagata.

- La workstation non deve stare vicino al robot: basta uno switch in ogni area di lavoro collegato allo switch principale; si lanciano le policy via SSH.
- **Non funziona se il robot deve muoversi liberamente** — lì serve per forza un computer a bordo.

---

## Il vincolo che si sottovaluta sempre: ARM

Orin, Thor e Spark sono **tutti ARM**. Questo significa:

- immagini Docker e librerie per ARM, diverse da quelle sviluppate sulle workstation x86
- conversione del modello con **TensorRT** e quantizzazione per ottenere velocità accettabili
- **è lavoro da rifare per ogni nuovo modello**, non una tantum
- da verificare caso per caso quanto ACT, Diffusion Policy e VLA siano effettivamente compatibili con Jetson, TensorRT ed eventualmente DeepStream

---

## Criterio pratico di allocazione

> [!tip] Centralizzare il training, distribuire l'inferenza
> - workstation condivise via SSH — le GPU personali restano ferme gran parte del tempo
> - una workstation **cablata dedicata ai robot**, per il test delle policy prima del deploy
> - un computer esterno a bordo (Thor o simile) per ogni robot nella configurazione finale

---

## Collegato

[[Performance in esecuzione]] · [[Framework]] · [[Simulazione]] · [[VLA]] · [[WAM e World Models]]
