# Project 05: Improvement Options Analysis
## Decision Log — Consolidamento Risultati Pre-Paper

Data: Giugno 2026  
Contesto: I risultati analitici del P05 sono completi (NB01-06 eseguiti), ma prima di scrivere il paper si valutano miglioramenti mirati per rafforzare le metriche downstream (IC, SCR) e la stabilità del modello.

---

## Problema centrale

L'RMSE one-step-ahead (6.17) è buono, ma i risultati downstream sono inflazionati:
- IC ±6.4 anni (vs ±0.9 in P04)
- SCR CHE +3.76 anni M / +2.92 F (vs +1.15 in P04 Total)
- Multi-seed CV 8.90% (borderline, soglia <10%)
- Vincoli AINN neutrali sull'RMSE (-0.009%)

La catena causale principale è: **process noise sex-specific sovrastimato → IC larghi → tutto si gonfia**.

---

## Le 5 opzioni

### Opzione 1: Tornare a Total (abbandonare split M/F)
- **Pro**: Restringerebbe gli IC immediatamente (σ mediato come P04).
- **Contro**: Elimina la *raison d'être* del P05 — lo split M/F è il differenziale più forte rispetto a P04. Senza di esso il progetto diventa "P04 con loss leggermente diversa e Optuna", troppo sottile.
- **Verdetto**: ❌ SCARTATA. Non toccare lo split M/F.

### Opzione 2: Ricalibrazione del process noise
Il σ di Li-Lee è calibrato sull'intera serie 1956-2020, includendo regimi non più rappresentativi (catch-up giapponese anni '60, volatilità europea anni '70). Inoltre, il σ storico include variabilità che l'LSTM ha già imparato a predire — si rischia il double counting.

Due sotto-opzioni (combinabili):

**2a) Calibrare σ su finestra recente (es. 1990-2020 o 2000-2020)**
- Pro: Semplice, difendibile (regime change), facile da implementare.
- Contro: Ancora basato sulla variabilità storica totale, non isolata dal modello.
- Sforzo: basso (poche righe nel NB04).

**2b) Calibrare σ sui residui del modello**
- Pro: Concettualmente superiore — misura l'incertezza residua che il modello *non* cattura, evitando il double-counting.
- Contro: Se calcolato solo sul validation set (9 anni), il σ è instabile. Servono residui one-step-ahead su un set più ampio (walk-forward o cross-validation).
- Sforzo: medio (richiede calcolo residui one-step-ahead walk-forward sul training set).

**Approccio raccomandato**: Combinare 2a + 2b. Calcolare residui one-step-ahead walk-forward sull'intero training set (o su 1990-2020), prendere il σ di quei residui come process noise. Questo dà un σ pulito (senza double counting) e robusto (molti punti).

### Opzione 3: Non proiettare ricorsivamente i specific factors
- Stato: **già implementato** (specifici fissi al 2020).
- Variante: ancorare il common factor alla drift Li-Lee dopo N step per limitare la divergenza ricorsiva a lungo termine.
- Verdetto: da valutare se dopo le opzioni 2+4 i risultati sono ancora troppo larghi.

### Opzione 4: Miglioramenti architetturali
Due sotto-opzioni (complementari):

**4a) Teacher forcing parziale (scheduled sampling / multi-step loss)**
- Pro: Risolve il mismatch train/inference alla radice. Il modello è allenato one-step-ahead ma usato ricorsivamente per 30 step — teacher forcing riduce il bias ricorsivo.
- Contro: Intervento profondo — richiede modificare il training loop, reintrodurre la ricerca Optuna (i λ ottimali cambieranno), e rivalidare tutto.
- Sforzo: alto (potenzialmente 1-2 settimane).
- Impatto atteso: riduzione del drift sistematico nelle proiezioni a 30 anni → IC più stretti strutturalmente.

**4b) Ensemble di seed**
- Pro: "Free lunch" — i 5 modelli dal test multi-seed esistono già. Media delle predizioni → CV cala, stabilità aumenta.
- Contro: Non risolve il bias, solo la varianza. È un cerotto, non una cura.
- Sforzo: basso (un pomeriggio).
- Impatto atteso: CV sotto il 5%, predizioni mediane più stabili.

**Relazione tra le due**: L'ensemble abbassa la varianza, il teacher forcing abbassa il bias ricorsivo. Sono complementari, non alternativi.

### Opzione 5: Ricontestualizzare i risultati (narrativa, non tecnica)
- Presentare P05 come complementare a P04, non sostitutivo.
- Mostrare che SCR sex-specific è necessariamente più alto perché decompone incertezza che P04 nasconde.
- Calcolare SCR "Total sintetico" da M/F per confronto diretto.
- **Verdetto**: ⚠️ ULTIMA SPIAGGIA. Non vogliamo usarla come scappatoia — deve essere l'ultimo resort se i miglioramenti tecnici non bastano.

---

## Piano d'azione raccomandato (Rivisto — Giugno 2026)

Ordine rivisto per massimizzare il rapporto **impatto × spiegabilità / sforzo**. L'idea è: fare prima gli interventi che non toccano il modello (post-processing), valutare i risultati, e solo se necessario intervenire sull'architettura.

### Razionale dell'ordine

- Step 1 e 2 sono **diagnostici oltre che correttivi**: se i CI si restringono dopo la ricalibrazione, abbiamo dimostrato che il problema era il double-counting del σ, non un difetto del modello. Questo è un risultato pubblicabile.
- Step 1 e 2 **non toccano il modello**. Il champion LSTM resta identico. I notebook 01-03 non si rieseguono.
- Il teacher forcing richiederebbe di rieseguire tutto da NB03 in poi. Meglio verificare prima se serve.

### Sequenza operativa

| Step | Opzione | Cosa | Sforzo | Impatto atteso | Spiegabilità |
|:--|:--|:--|:--|:--|:--|
| 1 | **2a+2b** | Ricalibrazione σ su residui walk-forward del modello | Medio | IC realistici (no double-counting) | Alta: "non contiamo due volte l'incertezza" |
| 2 | **4b** | Ensemble di 3-5 seed (model averaging) | Basso | CV < 5%, predizioni stabili | Alta: "credibility pooling tra modelli" |
| 3 | — | Riesecuzione NB04-06 con nuovi parametri | Medio | Nuovi SCR, IC, stress test | — |
| 4 | — | **Valutazione**: confronto IC/SCR con P04 | Basso | Decisione go/no-go su Step 5 | — |
| 5 | **4a** | *(Solo se necessario)* Teacher forcing parziale | Alto | Riduzione bias ricorsivo | Media: richiede spiegazione tecnica |
| 6 | *5* | *(Ultima spiaggia)* Narrativa complementare | Basso | Framing per revisori | — |

**Criterio di stop**: se dopo Step 1+2+3 gli IC sono ±2-3 anni e SCR nell'ordine di +1.5-2.5 anni, ci fermiamo. Se ancora inflazionati, procediamo con Step 5 (teacher forcing).

---

## Decisioni prese
- [x] Non abbandonare lo split M/F (Opzione 1 scartata)
- [x] Opzione 5 solo come ultimissima spiaggia
- [x] Procedere con Step 1 (ricalibrazione σ sui residui) — avviato

---

## Execution Log

### Step 1: Ricalibrazione σ su residui walk-forward (IN CORSO)

**Data inizio**: Giugno 2026  
**Branch**: `feat/p05-improvement-plan`  
**Notebook**: `07_process_noise_recalibration.ipynb`

**Approccio implementato (Opzione 2b — residui walk-forward)**:

1. Per ogni time step $t \geq$ lookback (15), si usa il champion model in modalità deterministica (`training=False`, dropout disattivato) per predire $\Delta K_t$ one-step-ahead.
2. Si calcola il residuo: $r_t = \Delta K_t^{osservato} - \Delta K_t^{predetto}$.
3. Il nuovo $\sigma_{residual} = \text{std}(r_t)$ sostituisce il vecchio $\sigma_{storico}$ come process noise.

**Perché `training=False`**: i residui devono riflettere l'errore sistematico del modello nella sua modalità "best estimate". La variabilità indotta dal dropout è già catturata dal MC Dropout nel forecasting. Usare `training=True` anche qui contaminerebbe i residui con varianza epistemica.

**Perché walk-forward su tutto il dataset**: calcolando i residui su tutti i 49 step eligibili (anni 1972-2020 per lb=15), il σ è statisticamente robusto. Limitarsi al solo validation set (9 anni) darebbe un σ instabile.

**Nota tecnica importante**: analizzando il codice del NB04, il σ storico era calcolato come `np.std(np.diff(feature_matrices[sex], axis=0))`. Siccome `feature_matrices` contiene già le first differences di $K_t$, `np.diff()` computa le *second differences*. Questo va tenuto in considerazione nell'interpretazione dei risultati.

**Contenuto del Notebook 07**:
- Sezione 7.1: Setup e caricamento asset
- Sezione 7.2: Calcolo residui walk-forward one-step-ahead
- Sezione 7.3: Confronto σ vecchio vs nuovo con tabella e reduction percentages
- Sezione 7.4: Diagnostica residui (bar chart, summary stats)
- Sezione 7.5: Re-run completo del forecast stocastico (MC Dropout 1000 sim + σ corretto)
- Sezione 7.6: Tabella comparativa IC vecchi vs nuovi per tutti i paesi × sessi
- Sezione 7.7: SCR aggiornati (VaR 99.5% e ES 99.0%)
- Sezione 7.8: Persistenza asset corretti

**Stato**: ✅ COMPLETATO.

**Risultati (Giugno 2026)**:

| Metrica | NB04 (vecchio σ) | NB07 (σ corretto) | Riduzione |
|:--|:--|:--|:--|
| σ Kt Male | 5.99 | 2.80 | **53%** |
| σ Kt Female | 9.08 | 3.64 | **60%** |
| 95% CI CHE Male | 7.73 yr | 3.77 yr | **51%** |
| 95% CI CHE Female | 7.88 yr | 3.15 yr | **60%** |
| SCR CHE Male (ES 99%) | +3.760 yr | +2.167 yr | **42%** |
| SCR CHE Female (ES 99%) | +2.919 yr | +1.606 yr | **45%** |

**Osservazioni chiave**:
- Le mediane non si sono mosse (CHE Male: 82.01→82.04, Female: 86.04→85.99). Il problema era interamente nel noise, non nel modello.
- Gli IC sono ora ±1.9-2.2 anni (M) e ±1.3-1.9 anni (F) — nel target di ±2-3 anni.
- La riduzione è uniforme su tutti i 6 paesi, confermando che il double-counting era sistematico.
- Risultato pubblicabile: "la calibrazione del process noise domina l'incertezza delle proiezioni a 30 anni".

**Decisione**: procedere con Step 2 (ensemble) per stabilizzare ulteriormente.

---

### Step 2: Ensemble di 5 seed (IN CORSO)

**Data inizio**: Giugno 2026
**Notebook**: `08_seed_ensemble.ipynb`

**Approccio implementato (Opzione 4b — model averaging)**:

1. Riallenare il champion (LSTM 48-32, lb=15, lr=0.001, λ=0.001) con 5 seed diversi: [42, 123, 256, 512, 1024].
2. Per ogni modello, eseguire 1,000 simulazioni MC Dropout (30 anni × 2 sessi).
3. Calcolare la media delle predizioni scalate dei 5 modelli (averaging in scaled space).
4. Applicare il σ corretto (da NB07) come process noise.
5. Ricostruire e₀, CI, SCR e confrontare con single-seed.

**Perché media in spazio scalato**: tutti i modelli condividono lo stesso scaler (fittato sugli stessi dati di training). Mediare le predizioni scalate cancella il rumore seed-specific preservando il segnale di mortalità condiviso.

**Spiegabilità**: "Model averaging — equivalente a credibility pooling tra modelli. Ogni modello vede gli stessi dati ma impara rappresentazioni leggermente diverse; la media è più robusta di qualsiasi singolo modello."

**Contenuto del Notebook 08**:
- Sezione 8.1: Setup e caricamento config + σ corretto da NB07
- Sezione 8.2: Data preparation e loss function (identiche a NB03)
- Sezione 8.3: Training di 5 modelli con timing e salvataggio
- Sezione 8.4: MC Dropout forecast per tutti i 5 modelli
- Sezione 8.5: Ensemble averaging + noise corretto
- Sezione 8.6: Ricostruzione e₀
- Sezione 8.7: Tabella comparativa a 3 stadi (NB04 → NB07 → NB08)
- Sezione 8.8: SCR aggiornati (ensemble)
- Sezione 8.9: Persistenza asset ensemble

**Budget computazionale**: ~50 min training + ~150 min forecast = ~200 min (~3.5 ore su M1 Pro).

**Stato**: Notebook creato, da eseguire.
