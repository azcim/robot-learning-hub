---
tags:
  - robot-learning
  - decisione
---

# Griglie decisionali

## Confronto diretto dei modelli

| | **[[ACT]]** | **[[Diffusion Policy]]** | **[[PPO e SAC\|PPO / SAC]]** | **[[VLA]]** | **[[WAM e World Models\|WAM]]** |
| --- | --- | --- | --- | --- | --- |
| **Paradigma** | IL | IL (+ RL con DPPO) | RL | IL pretrain + IL finetune + RL post | self-supervised + IL |
| **Dati necessari** | 50-200 demo per task | 50-200 demo per task | nessuna demo, ma milioni di passi in sim | pretrain: enorme (già fatto); finetune: 50-500 demo | video web + poche demo di allineamento |
| **Deploy su task nuovo** | riaddestramento completo | riaddestramento completo | nuova reward + nuovo training | **fine-tuning breve** | fine-tuning breve |
| **Generalizzazione** | nulla fuori dal task | nulla fuori dal task | nulla fuori dal task | **buona**, zero/few-shot | **ottima**, anche su dinamiche nuove |
| **Velocità di inferenza** | **molto alta** | media (passi di denoising) | **molto alta** | media-alta con action expert; bassa se autoregressivo | **bassa**, richiede distillazione |
| **Linguaggio** | no | no | no | **sì** | sì |
| **Hardware inferenza** | qualsiasi | qualsiasi | qualsiasi | 8-16 GB | Thor / Spark |
| **Maturità** | consolidato | consolidato | consolidato | **in produzione** | sperimentale |

---

## Dallo scenario alla scelta

| Scenario | Scelta | Perché |
| --- | --- | --- |
| Compito di manipolazione ripetitivo, scena controllata, serve una demo funzionante in fretta | **[[ACT]]** | 50-200 demo, poche ore su una GPU, inferenza leggerissima |
| Stesso caso, ma gli operatori eseguono il compito in modi diversi | **[[Diffusion Policy]]** | gestisce nativamente la multimodalità; **DP3** se l'illuminazione cambia |
| Locomozione, equilibrio, controllo corporeo di un umanoide | **[[PPO e SAC\|RL (PPO) in Isaac Lab]]** + domain randomization | reward chiare, fisica rigida simulata bene, milioni di passi gratis |
| Molti compiti, oggetti variabili, istruzioni in linguaggio naturale | **[[VLA]] + fine-tuning** | un task nuovo costa un fine-tuning, non un modello nuovo |
| Policy IL già discreta ma non affidabile, esiste un simulatore e un criterio binario | **[[DPPO]]** | l'RL rifinisce l'imitazione restando sul manifold delle traiettorie valide |
| Il robot deve lavorare con tessuti, cavi, oggetti deformabili | **[[WAM e World Models\|WAM]]**, o almeno pre-training su video | serve causalità fisica, non solo semantica |
| Un singolo skill molto generale (es. grasping) su oggetti sempre diversi | **VLA specialistico su dati sintetici** (stile GraspVLA) | per uno skill ben definito il sintetico scala meglio della teleoperazione |
| Raccogliere dati costa troppo tempo-operatore | **teleoperazione in simulazione** (LeIsaac) + Isaac Lab Mimic | si moltiplicano le demo senza occupare il robot |
| Serve ragionamento su compiti lunghi e scomponibili | **[[Embodied Reasoning\|ECoT in training, disattivato a runtime]]** | +30% di successo senza pagare la latenza |
| **Navigazione in ambiente strutturato** | **Planner classico** (A\*, RRT, DWA, MPC), non RL | problema prevalentemente geometrico: il planner è verificabile e si debugga |
| **Navigazione su terreno irregolare, folla, vincoli dinamici** | **RL — qui sì** | la dinamica conta più della geometria: è dove il planner non arriva |

---

## Albero decisionale rapido

1. **Il compito è facile da *mostrare* o facile da *specificare*?**
	- facile da mostrare → **[[Imitation Learning|IL]]**
	- facile da specificare con una reward binaria → **[[Reinforcement Learning|RL]]**
	- entrambi → IL per la base, RL per rifinire (**[[DPPO]]**)
2. **Quanti task diversi, oggi e fra sei mesi?**
	- uno o due, stabili → **ACT / Diffusion Policy**
	- molti, o variabili → **VLA**
3. **Quanto varia la scena?**
	- nulla → modello task-specific
	- oggetti e posizioni nuove → **VLA**
	- materiali deformabili e dinamiche nuove → **WAM** (o pre-training su video)
4. **Dove deve girare?**
	- su Jetson Orin a bordo → ACT / Diffusion Policy, o **SmolVLA** quantizzato
	- su Thor / workstation cablata → VLA pieno, o WAM
5. **C'è un simulatore fedele del compito?**
	- no → niente RL, niente DPPO: resta l'IL
	- sì → si aprono RL, DPPO, valutazione scalabile, dati sintetici

---

## Budget indicativi

|                            | Dati                           | Tempo di training                            | Rischio principale                            |
| -------------------------- | ------------------------------ | -------------------------------------------- | --------------------------------------------- |
| **ACT / Diffusion Policy** | 1-2 giornate di teleoperazione | ore, 1 GPU da 24 GB                          | la policy replica le imperfezioni del dataset |
| **RL locomozione**         | nessuna demo                   | giorni, GPU da 48 GB                         | reward engineering e sim-to-real              |
| **Fine-tuning VLA**        | 1-3 giornate di teleoperazione | ore-giorni, 48 GB con LoRA                   | catastrophic forgetting, latenza in inferenza |
| **DPPO**                   | 50 demo + simulatore           | giorni                                       | serve un simulatore fedele del compito        |
| **WAM**                    | video + poche demo             | fuori portata da zero: partire da checkpoint | ingegneria di runtime                         |

---

## Collegato

[[Percorsi end-to-end]] · [[I quattro assi]] · [[Punti aperti]]
