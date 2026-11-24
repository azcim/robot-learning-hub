---
tags:
  - robot-learning
  - decisione
---

# Percorsi end-to-end

## Percorso A — task specifico, risultato rapido

```
Teleoperazione sul robot reale (lerobot-record)
  → dataset LeRobot v3
  → training ACT o Diffusion Policy (lerobot-train, 24 GB)
  → valutazione open-loop
  → deploy sul computer di bordo
```

- **Nessun simulatore richiesto.**
- Se la policy è discreta ma non affidabile, si innesta **[[DPPO]]**: si costruisce l'ambiente simulato del compito e si rifinisce con RL.

Vedi [[ACT]] · [[Diffusion Policy]] · [[Framework]]

---

## Percorso B — locomozione e controllo corporeo

```
Modello URDF/USD del robot
  → ambiente Isaac Lab con reward progettate
  → training PPO + domain randomization (48 GB)
  → validazione sim-to-sim in MuJoCo
  → deploy a bordo
```

- **Nessuna dimostrazione umana coinvolta.**

Vedi [[PPO e SAC]] · [[Simulazione]]

---

## Percorso C — generalista multi-task

```
Checkpoint VLA pre-addestrato (π0, GR00T N1.7, SmolVLA)
  → teleoperazione mirata sul proprio robot e task
     (eventualmente aumentata in simulazione)
  → fine-tuning con LoRA (48 GB)          ← questo è Behavior Cloning
  → valutazione in ambiente specchio stile SIMPLER
  → deploy client-server su Thor o workstation cablata
  → opzionale: post-training RL sui rollout reali
```

Vedi [[VLA]] · [[Imitation Learning]] · [[Reinforcement Learning#RL dopo l'imitazione perché sul post-training le regole cambiano]]

---

## Le dipendenze da tenere a mente

- **I dati sono il collo di bottiglia, non gli algoritmi.** Azioni latenti, IDM, retargeting, spazi d'azione condivisi, video-to-sim, dati sintetici, teleoperazione in simulazione: tutti modi per spendere meno ore-operatore.
- **Il pre-training decide quanto costa il fine-tuning.** Un VLA costa meno demo di un ACT **per task**, anche se costa infinitamente di più in assoluto. Più task si prevedono, più il VLA conviene.
- **La scelta dell'hardware di inferenza retroagisce sul modello.** Se il deploy è su Orin, un VLA da 7B non è un'opzione: o SmolVLA, o quantizzazione, o si sposta l'inferenza su Thor o workstation cablata.
- **Il formato dei dati è una decisione architetturale.** LeRobot v3 come standard interno significa poter cambiare modello senza rifare la pipeline.
- **La simulazione serve a tre cose diverse** — generare dati, addestrare, valutare. Un progetto può averne bisogno per una sola delle tre.
- **La velocità del gesto si imposta in raccolta dati**, non solo in inferenza: è più economico che comprare GPU. → [[Performance in esecuzione]]

---

## Collegato

[[Griglie decisionali]] · [[Framework]] · [[Hardware]] · [[Punti aperti]]
