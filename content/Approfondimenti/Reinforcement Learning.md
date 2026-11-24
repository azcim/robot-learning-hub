---
tags:
  - robot-learning
  - paradigma
---

# Reinforcement Learning

> Nell'[[Imitation Learning|IL]] qualcuno dice cosa fare, e la policy è vincolata al tetto delle dimostrazioni. **Nell'RL l'agente interagisce da solo con l'ambiente** e ottimizza per tentativi ed errori. Il tetto dell'esperto salta.

Il framework è l'**MDP**, definito da quattro elementi: spazio degli stati, spazio delle azioni, dinamica di transizione `P(s'|s,a)`, funzione di reward `r(s,a)`.

Tre concetti di ottimizzazione:

- **ritorno accumulato** — somma scontata delle reward lungo **una singola traiettoria**; è una variabile casuale
- **obiettivo J(π)** — il **ritorno atteso**, cioè la media su tutte le traiettorie indotte dalla policy
- **policy ottima** — quella che massimizza J(π)

---

## Perché è difficile

- **Reward sparse e discontinue** — una reward a gradino ha gradiente nullo ovunque e indefinito nel salto. Si risolve ottimizzando il **valore atteso** rispetto a una policy stocastica, che può essere continuo e differenziabile anche se la reward non lo è.
- **Credit assignment** — il premio arriva molti passi dopo l'azione che lo ha causato.
- **Exploitation vs exploration** — sfruttare ciò che si sa contro esplorare per scoprire.
- **Dati non stazionari** — cambiando policy cambia la distribuzione di ciò che si osserva.
- **Costo dell'esperienza** — provare davvero è lento, usura l'hardware e può essere pericoloso.

---

## I due meccanismi che rendono tutto praticabile

### Value function

Una mappa locale che dice quanto conviene trovarsi in uno stato, assumendo di seguire la policy da lì in poi. Risolve il problema che J(π) è uno scalare globale, non interrogabile a metà episodio.

Da qui l'**equazione di Bellman**: valore attuale = reward immediato + valore atteso e scontato dello stato successivo.

### Bootstrapping

Non aspettare la fine dell'episodio: ci si ferma dopo un passo e si usa la **propria stima corrente** del valore dello stato successivo.

- **Senza bootstrapping (Monte Carlo):** si gioca l'episodio intero e si usa il ritorno reale.
- **Con bootstrapping (Temporal Difference):** dopo un passo si sostituisce "tutto il resto del ritorno" con la stima corrente.
- L'operatore di Bellman è una **contrazione**: applicato ripetutamente converge al punto fisso, e l'informazione si propaga all'indietro un passo per volta, senza path planning esplicito.

---

## Due distinzioni che spiegano tutta la tassonomia

### Model-based vs model-free

L'agente conosce dinamica e reward, oppure no?

- **Model-based:** si può calcolare tutto a tavolino con la programmazione dinamica. È puro **planning**.
- **Model-free:** si campiona l'esperienza reale al posto della media teorica. È l'unica via praticabile in robotica — la dinamica del mondo reale non è nota, e calcolarla a ogni passo sarebbe troppo lento per un loop a millisecondi.

### On-policy vs off-policy

La strategia che genera i dati è la stessa che si sta ottimizzando?

- **On-policy** (behavior = target): più stabile, ma i dati sono **usa e getta**.
- **Off-policy** (le due sono diverse): i dati si **riutilizzano** da un *experience replay buffer*. Molto più efficiente in campioni, ma off-policy + reti neurali + bootstrapping può far divergere l'algoritmo (da cui la *target network*).

---

## Quando usarlo

> [!important] L'RL da zero rende solo quando valgono tre condizioni insieme
> 1. il successo è **verificabile in modo oggettivo e binario**
> 2. l'**esplorazione casuale produce abbastanza successi** da generare segnale
> 3. la **fisica che conta è simulata bene** — servono milioni di passi, quindi serve per forza un simulatore

- **Locomozione e controllo corporeo** → le soddisfa tutte e tre. Caso d'uso principe.
- **Navigazione** → ne soddisfa una e mezza.
- **Manipolazione e grasping** → non ne soddisfa nessuna.
- **Dopo un'imitazione (post-training)** → le condizioni cambiano completamente.

### Navigazione vs manipolazione, in dettaglio

| | **Locomozione** | **Navigazione** | **Manipolazione** |
| --- | --- | --- | --- |
| **1. Successo binario** | sì — "sei in piedi?" | sì ma insufficiente — "sei arrivato?" non dice nulla su *come* | ambiguo — "ha afferrato?" dipende da slittamento, forza, stabilità |
| **2. L'esplorazione casuale produce successi** | sì — muovere i giunti a caso dà subito segnale di equilibrio | sì — muoversi a caso ti porta da qualche parte | **no** — un braccio mosso a caso non afferra mai niente |
| **3. La fisica che conta è simulata bene** | sì, corpo rigido | sì, geometria e ostacoli | **no** — attrito, deformazione e slittamento sono il punto debole dei simulatori |
| **Credit assignment** | immediato | medio | lungo: il premio arriva dopo decine di micro-aggiustamenti |
| **Verdetto per l'RL da zero** | **standard di fatto** | **spesso non conviene** | **da evitare** |

**Perché la navigazione delude, anche se "sembra" adatta.**

- Il problema non è trovare il traguardo, è tutto il resto: fluidità, sicurezza, non bloccarsi, non spaventare le persone, consumo. Tutto questo finisce in una **reward composita** con pesi da sintonizzare a mano — e la policy massimizza la somma, non l'intento.
- *Reward hacking* tipico: oscillare vicino al goal, girare in tondo per accumulare un termine di progresso, sfruttare contatti che il simulatore non penalizza.
- **Esiste un'alternativa matura.** La navigazione è un problema in larga parte **geometrico**, e i planner classici (A\*, RRT, DWA, MPC) lo risolvono bene, sono verificabili e si debuggano. Un RL end-to-end dà in cambio soprattutto opacità.
- **L'RL serve dove il planner non arriva:** terreni irregolari, dinamica che conta più della geometria, folla e interazione sociale, vincoli non olonomi. Lì il confine con la locomozione si assottiglia — e non a caso è lì che funziona.

**Perché la manipolazione da zero è ancora peggio.**

- Fallisce sulla **condizione 2**, la più grave: senza successi casuali non c'è segnale, e si è costretti al **reward shaping denso**, cioè a descrivere a mano il movimento — che è esattamente ciò che non si sa fare. Se lo si sapesse, non servirebbe imparare.
- Fallisce sulla **condizione 3**: la fisica del contatto è ciò che i simulatori modellano peggio, quindi anche quando l'RL converge in simulazione il sim-to-real morde dove fa più male.
- **Eccezioni reali:** skill brevi, ripetitivi, con successo misurato da un sensore — inserzione *peg-in-hole* con force/torque, rotazione *in-hand*, lancio.

---

## RL dopo l'imitazione: perché sul post-training le regole cambiano

*Se la reward su manipolazione è così difficile, come fa a funzionare il post-training RL dei VLA?*

**Perché il punto di partenza non è più il caso vuoto.**

- **Da zero** la reward deve *guidare l'esplorazione* attraverso uno spazio enorme → shaping denso obbligatorio.
- **Dopo l'imitazione** la policy porta già a casa il task buona parte delle volte. L'esplorazione diventa **locale**, attorno a traiettorie che quasi funzionano → **una reward sparsa e binaria riceve abbastanza successi da produrre gradiente utile**.
- In altre parole: **la condizione 2 viene soddisfatta dall'imitazione, non dalla reward.**

**Il problema cambia natura: da *progettare una reward* a *rilevare il successo*.**

- **sensori** — forza/coppia, contatto, pinza chiusa su qualcosa, peso rilevato
- **visione** — un classificatore o un VLM che guarda il frame finale e risponde sì/no
- **un umano** che etichetta qualche centinaio di rollout a posteriori, invece di scrivere una funzione
- È il concetto di **verifiable rewards**: un verificatore oggettivo, deterministico e binario.

**I metodi del post-training evitano del tutto di scrivere una reward a mano:** [[DPPO]], RECAP (π\*0.6), [[IRL, GAIL e DPPO]].

**Cosa resta difficile:**

- servono comunque **molti rollout**: sul robot vero è lento e usura l'hardware, in simulazione torna il sim-to-real sui contatti
- il **credit assignment** su orizzonti lunghi obbliga a spezzare il task in sotto-task verificabili
- *reward hacking* possibile anche con reward binarie: si inganna il rilevatore, non la fisica
- rischio di **degradare il pre-training**, da contenere regolarizzando verso la policy di partenza
- **ordine pratico:** il post-training RL è l'ultimo passo, non il primo. Spesso conviene prima raccogliere più demo o fare HG-DAgger

---

## La scala pratica, in ordine di costo crescente

1. **Planner classico** — se il problema è prevalentemente geometrico
2. **[[Imitation Learning]]** — se il compito è facile da *mostrare* e difficile da *specificare*
3. **IL → RL** — se serve superare l'esperto e si ha un verificatore di successo ([[DPPO]], RECAP)
4. **RL da zero** — solo se valgono tutte e tre le condizioni

---

## Che dati servono

**Non dimostrazioni ma interazione**: ambiente + reward. In pratica un simulatore veloce e parallelizzabile ([[Simulazione|Isaac Lab]]), un modello URDF/USD corretto, una **reward progettata a mano** e una strategia di **domain randomization**.

Ordini di grandezza molto sopra l'IL — ma sono campioni *sintetici e gratuiti*, non ore-uomo.

---

## Limiti e vantaggi

**Vantaggi** — può superare l'esperto; non servono dimostrazioni complete; esplorazione nativa; domain randomization per il sim-to-real.

**Limiti** — **reward engineering** come vero collo di bottiglia; resta task-specific; instabilità numerica con off-policy + reti + bootstrapping; gap sim-to-real; tempi lunghi e difficili da stimare.

---

## Collegato

[[PPO e SAC]] · [[Q-learning e DQN]] · [[IRL, GAIL e DPPO]] · [[DPPO]] · [[Imitation Learning]] · [[Simulazione]]
