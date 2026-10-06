# CRISPR-Cas9 Engineering Platform 2.0
 
**Disease name in. Ranked guide RNAs out**
 
Type sickle cell anemia. Get back the top therapeutic gene target (from 20,000+ studies), ranked cut sites scored against 5310 real CRISPR experiments, and LLM generated biological rationale grounded in PubMed literature. No gene lookup. No manual sequence analysis
 
**Phase 1** solved guide RNA design given a target gene **Phase 2** removes that prerequisite entirely
 
**Live Demo (Phase 2):** https://crispr-cas9-platform-20-deployed-vzx6txzovzmz2rf7s69jvv.streamlit.app/
 
---
 
## What This Solves
 
Designing CRISPR guide RNAs is slow and manual. For any target gene, scientists must evaluate hundreds of candidate sequences against multiple biological criteria, cross-reference published literature and decide which sequence to order. This process typically takes hours of work before any wet lab work begins
 
Phase 1 automated guide RNA design given a target gene. Phase 2 removes that prerequisite entirely: given only a disease name, the pipeline selects the top therapeutic gene target using evidence from 20,000+ studies (OpenTargets), scores all cut sites using XGBoost trained on 5310 real CRISPR experiments (Doench 2016), and generates biological rationale grounded in retrieved PubMed literature
 
**Phase 1 (prior):** guide RNA design given a target gene
**Phase 2 (this project):** disease name in, ranked guide RNAs out
 
---
 
## Results
 
| Metric | Baseline LLM | RAG + LLM |
|---|---|---|
| Top-5 guide coverage | 0.0 | **0.52** |
| Score grounding | 0.0 | **1.0** |
| Mean cited rank | 0.0 | **2.29** |
| Hallucination rate | 0.38 | **0.06** |
 
Baseline score grounding is 0.0 across all 8 diseases because XGBoost scores(e.g 0.8752) were computed by this pipeline and could not exist in any LLM's training data. RAG scores 1.0 on all 8. This is the cleanest possible demonstration that retrieval is adding information the model cannot hallucinate
 
Hallucination dropped from **38% to 6%**. The 6% residual is a single-base substitution in DMD
 
---
 
## Why RAG and Not Just LLM
 
Ask a 7B LLM to design CRISPR guides for sickle cell disease with no context. It produces generic textbook output correct gene name, plausible sounding sequences, reversed mutation description ("valine replacing glutamic acid" instead of the correct direction) 38% of the sequences it cites are fabricated.
 
The sequences this pipeline computes (e.g. AGTCTGCCGTTACTGCCCTG, score 0.8752) did not exist before this pipeline ran them. No LLM can produce them from memory. RAG injects them as guaranteed context. That is why score grounding goes from 0.0 to 1.0 and hallucination drops from 0.38 to 0.06
 
---
 
## Architecture
 
```
"sickle cell anemia"
        |
        v
  OpenTargets API
  EFO lookup + associatedTargets
  -> HBB (score: 0.8287), BCL11A, RRM2B ...
        |
        v
  NCBI Entrez
  NM_ RefSeq accession + mRNA fetch
  -> HBB: NM_000518.5, 628 bp
        |
        v
  PAM Scanner
  All NGG sites -> 30mer context
  -> 52 candidates for HBB
        |
        v
  XGBoost Regressor
  Trained on Doench 2016 (5310 guides, 93 features)
  r2 = 0.27, within range of Azimuth sequence-only performance
  -> rank 1: AGTCTGCCGTTACTGCCCTG, score 0.8561
        |
        v
  3-section prompt (guaranteed):
  [OPENTARGETS]  gene association scores
  [ML PIPELINE]  XGBoost-ranked guide sequences
  [LITERATURE]   MMR-retrieved PubMed abstracts
        |
      +---+---+
      |       |
  Baseline  RAG+LLM
  Qwen2.5   Qwen2.5
  no ctx    3-section ctx
      |       |
      +---+---+
          |
    4-metric eval
    across 8 diseases
```
 
---
 
## What Changed from Phase 1
 
| Component | Phase 1 | Phase 2 |
|---|---|---|
| Gene selection | Fixed disease to gene map | OpenTargets GraphQL API (live association scores) |
| Scoring | Heuristic rules, 100 point scale | XGBoost regression, trained on Doench 2016 (5310 rows) |
| Features | 8 hand coded features | 93 features: 13 sequence+80 positional one-hot |
| Input | Disease or gene name | Disease name only |
| RAG context | PubMed + guide docs in ChromaDB | OT + guides hardcoded in prompt, MMR for PubMed only |
| Evaluation | 5 metrics, 8 genes | 4 metrics redesigned for grounding quality, 8 diseases |
 
Phase 1 heuristic and Phase 2 XGBoost produce **zero overlapping top guides** for HBB despite both using Doench 2016 as the source. The heuristic interprets the paper's described rules. XGBoost fits directly to the raw experimental data. XGBoost is ground truth.
 
---
 
## 8 Diseases Covered
 
| Disease | OT Top Gene | Expected | Note |
|---|---|---|---|
| Sickle cell anemia | HBB | HBB | Association score 0.8287 |
| HIV infection | CCR5 | CCR5 | Match |
| Huntington's disease | HTT | HTT | Match |
| Duchenne muscular dystrophy | DMD | DMD | Match |
| Cystic fibrosis | CFTR | CFTR | Match |
| Breast cancer | BRCA2 | BRCA1 | OT correct -- BRCA2 has higher evidence score |
| Leber congenital amaurosis | RPE65 | CEP290 | OT correct RPE65 is established clinical target |
| Chronic myeloid leukemia | ABL1 | BCR | OT correct ABL1 is the kinase BCR-ABL targets |
 
Three gene mismatches occurred because OpenTargets ranks by aggregate genetic evidence across 20,000+ studies, not by which gene is most commonly cited in CRISPR papers. These are treated as correct OT outputs rather than errors. Only the BRCA1/BRCA2 case is a real biological debate
 
---
 
## XGBoost Scoring Model
 
Trained on Doench et al. 2016 5310 experimental guide RNA efficiency measurements, the same dataset used to build Azimuth (the field's most cited guide RNA scoring tool). r2 = 0.27 on held-out test set (80/20 split, random_state=42). The gap between this and Azimuth's reported sequence-only performance (~0.3-0.4) reflects the same fundamental limit: chromatin accessibility, local DNA structure, and cellular context all affect efficiency but are not encodable from sequence alone
 
**93 features per candidate:**
 
13 sequence features
 
| Feature | What it captures |
|---|---|
| gc_30mer | GC content of full 30mer context |
| gc_20mer | GC content of guide (optimal 40-70%) |
| gc_seed | GC content of seed region (last 12nt before PAM) |
| poly_t | Binary flag: four or more consecutive T's terminate transcription |
| a_freq, t_freq, g_freq, c_freq | Per-base frequency of guide |
| gc_clamp | Binary: G or C at final guide position improves Cas9 binding |
| homopolymer | Longest single-base run (longer = higher off target risk) |
| dinuc_repeat | Count of repeated dinucleotides |
| unique_dinucs | Number of unique dinucleotide pairs (low = repetitive sequence) |
| seed_unique_bases | Unique bases in seed region (low diversity = off-target risk) |
 
80 positional features: one-hot encoding of each of the 4 bases at each of the 20 guide positions (20*4=80 binary features). This encodes position specific nucleotide preferences learned from the 5310 experimental measurements.
 
---
 
## Tech Stack
 
| Component | Tool |
|---|---|
| Gene target selection | OpenTargets GraphQL API v4 |
| Sequence retrieval | Biopython+NCBI Entrez API |
| Guide RNA scoring | XGBoost 2.x trained on Doench 2016 |
| Literature retrieval | PubMed via Entrez esearch + efetch |
| Embeddings | sentence-transformers/all-MiniLM-L6-v2 |
| Vector store | ChromaDB with MMR retrieval |
| RAG orchestration | LangChain |
| LLM | Qwen/Qwen2.5-7B-Instruct (float16, RTX 3060) |
| Evaluation | Custom regex metrics + Matplotlib |
| Hardware | NVIDIA RTX 3060 12GB VRAM |
 
---
 
## Dataset
 
| Source | Content | How used |
|---|---|---|
| OpenTargets API | Disease gene association scores (0-1) | Gene target selection and ranking |
| NCBI Nucleotide | Reference mRNA sequences via RefSeq | PAM scanning and 30mer context extraction |
| Doench et al. 2016 | 5310 experimental guide efficiency scores | XGBoost training data |
| PubMed via Entrez | Disease + CRISPR targeted paper abstracts | RAG document corpus (ChromaDB) |
 
All data frozen to disk on first fetch. Subsequent runs load from disk with no API calls. Cell 6 (8-disease loop) took 153 minutes on first run; all future reruns are instant
 
---
 
## Cell Structure
 
| Cell | Purpose |
|---|---|
| Cell 1 | Setup, OpenTargets query, NCBI sequence fetch, PubMed paper fetch, data freezing |
| Cell 2 | Gene ranking from OpenTargets association scores, top 5 selection |
| Cell 3 | PAM scan, 30mer context extraction, XGBoost training and scoring, guide ranking |
| Cell 4 | LLM standalone baseline disease name only, no context |
| Cell 5 | RAG+LLM 3 section grounded prompt, Qwen generation, frozen output |
| Cell 6 | 8-disease inference loop full pipeline per disease, load model once |
| Cell 7 | 4-metric evaluation, per disease table, visualization across all 8 diseases |
 
---
 
## Evaluation
 
Baseline (Cell 4) and RAG+LLM (Cell 5/6) are compared across 8 diseases using 4 metrics computed from response text via regex
 
| Metric | What it measures | Baseline avg | RAG avg |
|---|---|---|---|
| top5_coverage | Fraction of top-5 XGBoost-ranked guides cited in response | 0.0 | 0.52 |
| score_grounding | Binary: did response cite a real XGBoost score (4 decimal match) | 0.0 | 1.0 |
| mean_cited_rank | Mean rank position of sequences cited (lower = better) | 0.0 | 2.29 |
| hallucination | Fraction of cited 20-mers not in ranked guide list | 0.38 | 0.06 |
 
top5_coverage and score_grounding are both 0.0 for all baseline responses. The baseline LLM cannot produce XGBoost scores or cite sequences that were computed by this pipeline and never appeared in any training data. This is the most concrete possible demonstration that RAG is adding information the model could not have.
 
mean_cited_rank of 2.29 shows RAG is surfacing guides near the top of the ranked list, not random candidates from the bottom.
 
### Per-Disease Results
 
| Gene | b_cov | r_cov | b_scr | r_scr | b_rank | r_rank | b_hal | r_hal |
|---|---|---|---|---|---|---|---|---|
| HBB | 0.0 | 0.6 | 0 | 1 | 0.0 | 2.0 | 1.0 | 0.0 |
| CCR5 | 0.0 | 0.6 | 0 | 1 | 0.0 | 3.0 | 0.0 | 0.0 |
| HTT | 0.0 | 0.6 | 0 | 1 | 0.0 | 4.8 | 0.0 | 0.0 |
| DMD | 0.0 | 0.2 | 0 | 1 | 0.0 | 1.0 | 0.0 | 0.5 |
| CFTR | 0.0 | 0.8 | 0 | 1 | 0.0 | 2.5 | 0.0 | 0.0 |
| BRCA2 | 0.0 | 0.4 | 0 | 1 | 0.0 | 1.5 | 1.0 | 0.0 |
| RPE65 | 0.0 | 0.4 | 0 | 1 | 0.0 | 1.5 | 0.0 | 0.0 |
| ABL1 | 0.0 | 0.6 | 0 | 1 | 0.0 | 2.0 | 1.0 | 0.0 |
 
---
 
## Challenges
 
### Challenge 1 OpenTargets v4 tractability schema mismatch
The tractability subfields (smallmolecule { topBucket value }, antibody, otherModalities) returned 400 errors v4 restructured this field. Dropped tractability and genetic sub scores, kept overall association_score which is sufficient for gene ranking. Full tractability integration requires schema introspection against current v4 API
 
### Challenge 2 XGBoost r2 plateau at 0.08 with basic features
Initial 8-feature model (GC content, poly-T, seed GC, position, base frequencies) achieved r2=0.08. Solved by adding positional nucleotide encoding: one-hot encoding of each base at each of the 20 guide positions (80 binary features). r2 jumped from 0.08 to 0.27. This encodes position-specific nucleotide preferences that the sequence-level features cannot capture Cas9 cuts more efficiently when certain bases appear at specific positions regardless of overall GC content
 
### Challenge 3 Off-target scoring without genome alignment
True off target analysis requires Bowtie or BLAST against the 3GB human genome reference not feasible in a notebook. Solved by decomposing sequence complexity into 4 independent features (homopolymer, dinuc_repeat, unique_dinucs, seed_unique_bases) that capture different shapes of repetitiveness. XGBoost learns the weight of each from 5310 Doench experiments instead of a hand-tuned point deduction
 
### Challenge 4 XGBoost score compression, top guide stuck at 0.74
Initial 8 feature model had poor score separation rank 1 guide scored ~0.74 and most candidates bunched near 0.4. Resolved iteratively: positional encoding pushed top score from 0.74 to 0.879, complexity features improved distribution shape. Final top guide for HBB scores 0.8561 with 42 candidates passing threshold and reasonable spread across the distribution
 
### Challenge 5 Heuristic vs XGBoost guide disagreement
Phase 1 heuristic and Phase 2 XGBoost recommend completely different top guides for HBB with zero overlap. Both reference Doench 2016. The heuristic approximates the paper's described rules with hand assigned weights. XGBoost fits directly to the raw experimental data. Different guides get picked because approximated rules diverge from what 5310 real cutting experiments actually showed. This is expected and desirable XGBoost is what the data says, the heuristic is what the paper says the data says
 
### Challenge 6 ChromaDB MMR retrieval ignored ML pipeline guide RNA docs
MMR retrieval consistently returned 5-6 PubMed abstracts and excluded the guide RNA document and OpenTargets scores document, causing the LLM to hallucinate sequences instead of citing XGBoost ranked ones. Root cause: structured tabular data (ranked sequences+decimal scores) has low semantic similarity to the CRISPR design query compared to full paper abstracts. MMR's diversity penalty then bumped the guide doc out of the retrieved set. Fix: removed both from ChromaDB entirely, hardcoded them as guaranteed sections in the prompt. MMR now only handles PubMed literature retrieval where diversity actually helps
 
### Challenge 7 Metric design for RAG evaluation
Initial metrics included target gene accuracy which scored 1.0 for both baseline and RAG useless since baseline LLMs already know gene names from training data. Guide citation was binary (cited any real sequence or not) which gave a perfect 1.0 RAG score with no per-disease variation. Fixed by replacing target gene accuracy with mean rank of cited guides (lower=better) measures whether RAG surfaces high-ranked guides specifically. Changed guide citation from binary to top-5 coverage rate exposes real variation like DMD at 0.20 vs CFTR at 0.80. Final 4 metrics each capture a distinct failure mode: coverage (breadth), score grounding (factual accuracy), mean cited rank (ranking quality), hallucination (reliability)
 
---
 
## Limitations
 
- Off-target scoring uses sequence complexity features, not genome-wide alignment
- RAG quality depends on coverage and recency of fetched PubMed papers
- XGBoost trained on Doench 2016 only more recent large-scale datasets (Leenay 2019, Kim 2019) not included
- Qwen2.5-7B reasoning depth limited on consumer hardware summarizes tradeoffs rather than deeply comparing candidates
- Hallucination detection uses 20-mer regex match a single-base substitution is not caught
---
 
## Reproducibility
 
All data frozen to disk on first run. Subsequent runs load from disk with zero API calls.
 
```bash
pip install biopython requests xgboost scikit-learn pandas numpy matplotlib \
            transformers torch langchain langchain-community langchain-core \
            chromadb sentence-transformers
```
 
Run cells in order: 1, 2, 3, 4, 5, 6, 7. Cell 6 will take ~2.5 hours on first run (8 diseases, loads Qwen once for all). All data fetched from public sources no restricted access required
 
---
 
## Future Work
 
**Phase 2 extensions:**
- Bowtie-based genome-wide off-target scoring against GRCh38
- Additional training datasets (Leenay 2019, DeepCRISPR) for improved r2
- Reranker layer on top of MMR for better retrieval quality
**Phase 3:**
- Problem 1: mutation detection in patient sequences
- End-to-end pipeline connecting all three problems
- Multi-disease batch processing with structured output
---
 
## Impact
 
Phase 1 reduced guide RNA selection from 8-10 hours to minutes. Phase 2 removes the prerequisite: the user no longer needs to know which gene to target. Disease name in, ranked guide RNAs out with association scores from 20,000+ studies (OpenTargets), efficiency predictions from 5310 real experiments (Doench 2016), and biological rationale grounded in PubMed literature
 
The pipeline combines:
- Evidence-based gene selection (OpenTargets)
- Experimentally-grounded guide scoring (XGBoost on Doench 2016)
- Literature-grounded LLM reasoning (RAG + Qwen2.5-7B)
The output is not just ranked sequences it is ranked sequences with predicted efficiency scores, genomic position context and LLM generated biological rationale backed by retrieved literature
 
---
 
## Project Context
 
Built as part of a self-directed AI/ML specialization alongside a Biotechnology undergrad degree at NSUT Delhi. The Bio + AI positioning is intentional this project requires both ML engineering (XGBoost feature engineering, RAG architecture, VRAM management) and biomedical domain reasoning (PAM biology, seed region specificity, OpenTargets evidence schema)
 
Phase 1: https://github.com/Ishan-2-0/CRISPR-Cas9-Engineering-Platform
 
Phase 1 Live Demo: https://huggingface.co/spaces/i-chan/CRISPR-Cas9-Engineering-Platform-RAG
 
