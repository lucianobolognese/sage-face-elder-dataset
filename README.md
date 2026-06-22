# SAGE-Face: Senior Affective Graph of Emotions

> **Tesi di Laurea in Informatica — Sistemi ad Agenti**  
> Università degli Studi di Bari Aldo Moro, A.A. 2025–2026  
> **Laureando:** Luciano Domenico Bolognese  
> **Relatori:** Prof.ssa Berardina Nadja De Carolis · Dr. Giuseppe Palestra

---

## Panoramica

I modelli di **Facial Expression Recognition (FER)** mostrano un marcato degrado prestazionale sui volti di soggetti anziani. La causa è strutturale: la morfologia dell'invecchiamento — rughe statiche, ptosi dei tessuti molli, solchi nasolabiali — interferisce con i pattern di contrazione muscolare che i modelli hanno imparato a riconoscere, quasi esclusivamente su soggetti giovani-adulti.

**SAGE-Face** (Senior Affective Graph of Emotions) nasce per colmare questa lacuna. Non è un dataset raccolto da zero tramite campagna di acquisizione, ma il risultato di una raffinata operazione di **ingegneria dei dati**: aggregazione, filtraggio demografico e purificazione multimodale delle istanze senili presenti nei principali dataset FER dello stato dell'arte.

Il dataset si propone come standard di riferimento per la validazione della resilienza dei modelli FER in contesti di assistenza geriatrica e robotica sociale.

---

## Pipeline di costruzione

La trasformazione di sorgenti eterogenee in un dataset coeso è articolata in tre moduli sequenziali.

### 1. Integrazione statico-dinamica e Motore di Decisione Semantica

SAGE-Face aggrega sei sorgenti primarie con domini di acquisizione complementari:

| Sorgente | Tipo | Caratteristiche |
|----------|------|-----------------|
| **AffectNet** | Statico | In-the-wild, 96×96, annotazioni discrete + valenza/arousal |
| **FACES** | Statico | Laboratorio controllato, 2835×3453, altissima purezza semantica |
| **RAF-DB** | Statico | In-the-wild, crowdsourcing 40+ annotatori |
| **FERPlus** | Statico | Re-etichettatura di FER-2013 con soft-label, 10 annotatori |
| **DFEW** | Dinamico | Video da produzioni cinematografiche, 16 frame per clip |
| **ElderReact** | Dinamico | Video focalizzati su soggetti anziani, annotazioni multi-label |

Per i dataset dinamici (DFEW, ElderReact), ogni sequenza video standardizzata a 16 frame viene ridotta al **Frame 8** (apex dell'espressività facciale).

Nel caso di ElderReact, le annotazioni presentano frequenti etichette multiple e conflittuali. Un **Motore di Decisione Semantica** risolve l'ambiguità incrociando i flag delle emozioni discrete con il punteggio dimensionale di Valenza, secondo quattro regole:

- **Emozione pura** — un solo flag attivo → etichetta acquisita direttamente
- **Conflitto positivo** — flag *Happy* + altra emozione → classificato *Happy* solo se Valenza ≥ 5.0
- **Conflitto negativo** — emozione negativa + *Surprise* → preservata l'etichetta negativa solo se Valenza ≤ 3.5 (stato di distress confermato)
- **Scarto per entropia** — caos multi-negativo o incertezza insanabile → campione rimosso

### 2. Cancello Demografico di Primo Livello (Predictive Filtering)

Per estrarre i volti senili dai dataset generalisti, ogni immagine viene sottoposta a stima predittiva dell'età tramite **DeepFace**.

La soglia di ammissione è fissata a **38 anni** — un limite volutamente conservativo che opera in *over-fetching*. Data la tendenza degli algoritmi biometrici a sottostimare l'età su immagini a bassa risoluzione o con illuminazione forte, una soglia bassa evita di scartare soggetti anziani ben conservati (falsi negativi), demandando la pulizia fine allo stadio successivo.

### 3. Gatekeeper Multimodale (VLM Purification)

Il filtraggio algoritmico lascia inevitabilmente una quota di falsi positivi — soggetti giovani valutati erroneamente come anziani dall'analisi biometrica bidimensionale. Per garantire l'integrità del dataset viene introdotto un livello di validazione finale basato su un **Vision-Language Model**.

Ogni frame non certificato viene analizzato da **Qwen3-VL** (eseguito localmente per latenza e privacy) con un prompt di ispezione rigoroso. Al modello viene assegnato il mandato esplicito di agire da *Gatekeeper*, cercando segni inequivocabili di invecchiamento cellulare: rughe profonde, macchie senili, ptosi, perdita di elasticità epidermica. Viene imposta l'identificazione di un'età apparente superiore ai 60 anni.

Il modello è vincolato a una risposta binaria **(YES / NO)**. Tutte le immagini non classificate con certezza assoluta vengono deviate in una directory di contenimento ed escluse dal dataset finale.

**Whitelist metodologica:** i dataset clinicamente certificati come anziani — FACES ed ElderReact — bypassano il blocco VLM tramite pre-etichettatura nel nome del file, fluendo direttamente nel dataset finale. Questo ottimizza il carico computazionale ed evita un filtraggio ridondante su sorgenti già demograficamente pure.

---

## Statistiche finali

| Split | Campioni |
|-------|----------|
| Train | 7.650 |
| Test | 1.916 |
| **Totale** | **9.566** |

- **Risoluzione:** 224×224 px
- **Classi:** 7 emozioni di Ekman (Angry, Disgust, Fear, Happy, Neutral, Sad, Surprise)
- **Annotazione:** ibrida — singolo esperto (AffectNet, FACES), crowdsourcing (RAF-DB, FERPlus), automatica con validazione VLM

> ⚠️ **Il test set non va mai soggetto a data augmentation.** È costruito per rappresentare una condizione reale in-the-wild e serve da specchio fedele per la valutazione comparativa. Alterarlo introdurrebbe artefatti statistici che invaliderebbero le metriche di resilienza.

---

## Struttura della repository

```
├── pipeline/
│   ├── 01_preprocess.py        # Preprocessing sorgenti statiche: normalizzazione
│   │                           # etichette, resize, costruzione Master CSV
│   ├── 02_Sage_Builder_Dynamic.py  # Estrazione Frame 8 da DFEW ed ElderReact
│   │                               # + Motore di Decisione Semantica (Valenza)
│   ├── 03_pulizia_llm.py       # Gatekeeper VLM: validazione binaria con Qwen3-VL
│   │                           # su tutti i campioni non certificati anagraficamente
│   ├── 04_rescue_anziani.py    # Recupero falsi negativi: re-ispezione campioni
│   │                           # borderline scartati dal filtro DeepFace
│   ├── 05_resize_llm.py        # Resize finale a 224×224 px dei campioni validati
│   ├── 06_split_data.py        # Split stratificato train/test (80/20)
│   │                           # con random state fisso per riproducibilità
│   ├── 07_reforgesage.py       # Aggregazione finale: fusione di tutte le sorgenti
│   │                           # nel dataset SAGE-Face completo
│   └── 08_analyze.py           # Analisi distribuzione classi, statistiche
│                               # e validazione integrità del dataset
│
```

---

## Utilizzo del dataset

### Per benchmarking

Usare esclusivamente lo **split di test ufficiale** (1.916 campioni). Non applicare augmentation. Non mischiare con il train set.

### Per fine-tuning

Il train set (7.650 campioni) può essere arricchito con augmentation esclusivamente sulle classi minoritarie (Fear, Disgust, Surprise). Si raccomanda di affiancare campioni da altri dataset per evitare il catastrophic forgetting delle capacità di generalizzazione pre-acquisite.

---

## Requisiti

```bash
pip install torch torchvision
pip install deepface
pip install albumentations
pip install scikit-learn
pip install opencv-python
pip install transformers   # per Qwen3-VL
```

---

## Download Dataset
<a href="[https://example.com](https://drive.google.com/drive/folders/1SIr9uJv4uuyN2d9pdLvANSsOUdl3IbeF?usp=sharing)">
  <button>SAGE-Face</button>
</a>

## Citazione

```bibtex
@thesis{bolognese2026sageface,
  author  = {Bolognese, Luciano Domenico},
  title   = {Analisi comparativa dei metodi di riconoscimento delle emozioni
             primarie dal volto negli anziani},
  school  = {Università degli Studi di Bari Aldo Moro},
  year    = {2026},
  type    = {Tesi di Laurea Triennale},
  advisor = {De Carolis, Berardina Nadja and Palestra, Giuseppe}
}
```

---

## Licenza

Il codice è rilasciato sotto licenza **MIT**.  
SAGE-Face è distribuito esclusivamente per fini di ricerca accademica, nel rispetto delle licenze dei dataset sorgente: AffectNet, RAF-DB, FER-2013/FERPlus, FACES, DFEW, ElderReact.
