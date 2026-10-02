# CRISPR-Cas9 Engineering Platform 2.0
 
Phase 2 of the biomedical AI platform disease name in, ranked guide RNAs out. Extends Phase 1 upstream with OpenTargets-driven gene selection and replaces the hand-coded heuristic scorer with XGBoost trained on 5310 Doench 2016 experimental measurements
 
---
 
## Summary
 
A disease-to-guide-RNA pipeline that:
 
- Queries OpenTargets (EFO ontology) to select the top therapeutic gene target for any disease
- Fetches reference mRNA sequences dynamically from NCBI via RefSeq accession
- Scans all NGG PAM sites and scores candidates using XGBoost trained on real CRISPR efficiency data
- Retrieves PubMed literature via MMR and builds a 3-section grounded prompt (OpenTargets + ML pipeline + literature)
- Generates baseline (no context) and RAG+LLM responses with Qwen2.5-7B-Instruct
- Benchmarks both across 8 diseases using 4 custom evaluation metrics
---
 
## What Phase 2 Added Over Phase 1
 
| Component | Phase 1 | Phase 2 |
|---|---|---|
| Gene selection | Fixed disease-to-gene map | OpenTargets GraphQL API (live association scores) |
| Scoring | Heuristic rules, 100-point scale | XGBoost regression, trained on Doench 2016 (5310 rows) |
| Features | 8 hand-coded features | 93 features: 13 sequence + 80 positional one-hot |
| Input | Disease or gene name | Disease name only |
| RAG context | PubMed + guide docs in ChromaDB | OT + guides hardcoded in prompt, MMR for PubMed only |
| Evaluation | 5 metrics, 8 genes | 4 metrics redesigned for grounding quality, 8 diseases |
 
---
 
## Problem Statement
 
Phase 1 solved Problem 3 (guide RNA design given a target gene). Phase 2 pulls the starting point upstream: given only a disease name, which gene should you target and which cut sites should you use?
 
OpenTargets aggregates genetic, clinical, and functional evidence across thousands of studies to produce an association score (0-1) for every known disease-gene pair. This project uses that score as the gene selection signal, replacing the static lookup table from Phase 1 with a live ranked query.
 
Scoring candidates with hand-coded rules (Phase 1) is a reasonable proxy when no training data exists. Phase 2 replaces the proxy with an XGBoost model fitted to actual guide efficiency measurements from Doench et al. 2016 the same experimental dataset the field's own Azimuth tool was built on.
 
---
 
## System Architecture
 
```
User inputs disease name (e.g. "sickle cell anemia")
            |
            v
OpenTargets GraphQL API
EFO ontology lookup -> disease EFO ID
associatedTargets query -> ranked gene list with association scores
            |
            v
NCBI Entrez API
esearch NM_ RefSeq accession for top gene
efetch reference mRNA sequence
(e.g. HBB -> NM_000518.5 -> 628 bp)
            |
            v
PAM Scanner
Finds all NGG sites in sequence
Builds 30mer context: 4nt upstream + 20nt guide + 3nt PAM + 3nt downstream
(e.g. HBB yields 52 NGG candidate sites)
            |
            v
XGBoost Regressor
Trained on Doench 2016 (5310 guides, score_drug_gene_rank target)
93 features: 13 sequence + 80 positional one-hot
Predicts efficiency score for each candidate
Filter>=0.4, keep top 10 ranked guides
(e.g. HBB rank 1: AGTCTGCCGTTACTGCCCTG, score 0.8752)
            |
            v
PubMed fetch via Entrez
Disease + gene + CRISPR targeted queries
MMR retrieval from ChromaDB (pubmed only)
            |
            v
3-section context block:
[OPENTARGETS] hardcoded gene scores
[ML PIPELINE] hardcoded ranked guides
[LITERATURE]  MMR-retrieved pubmed chunks
            |
      +-----+-----+
      |           |
      v           v
  Baseline       RAG+LLM
  Qwen2.5-7B    Qwen2.5-7B
  no context    3-section context
      |           |
      +-----+-----+
            |
            v
Cell 7: 4-metric evaluation across 8 diseases
top-5 coverage | score grounding | mean cited rank | hallucination rate
```
 
---
 
## Diseases and Target Genes
 
| Disease | OT Top Gene | Expected | Note |
|---|---|---|---|
| Sickle cell anemia | HBB | HBB | Association score 0.8287 |
| HIV infection | CCR5 | CCR5 | Match |
| Huntington's disease | HTT | HTT | Match |
| Duchenne muscular dystrophy | DMD | DMD | Match |
| Cystic fibrosis | CFTR | CFTR | Match |
| Breast cancer | BRCA2 | BRCA1 | OT correct BRCA2 has higher evidence score |
| Leber congenital amaurosis | RPE65 | CEP290 | OT correct RPE65 is established clinical target |
| Chronic myeloid leukemia | ABL1 | BCR | OT correct ABL1 is the kinase BCR-ABL targets |
 
3 gene mismatches occurred because OpenTargets ranks by aggregate genetic evidence, not by which gene is most commonly cited in CRISPR papers. These are treated as correct OT outputs rather than errors. Only the BRCA1/BRCA2 case is a real biological debate.
 
---
 
## XGBoost Scoring Model
 
### Training Data
 
Doench et al. 2016 published 5310 experimental guide RNA efficiency measurements across human and mouse genes. Each row contains a 30mer sequence context and a normalized efficiency score (`score_drug_gene_rank`). This is the same dataset used to build Azimuth, the field's most cited guide RNA scoring tool
 
### Feature Engineering
 
Each candidate is scored from its 30mer context: 4 nucleotides upstream + 20 nucleotide guide + 3 nucleotide PAM + 3 nucleotides downstream.
 
**13 sequence features:**
 
| Feature | What it captures |
|---|---|
| gc_30mer | GC content of full 30mer context |
| gc_20mer | GC content of guide (optimal 40-70%) |
| gc_seed | GC content of seed region (last 12nt before PAM) |
| poly_t | Binary flag: 4 or more consecutive T's terminate transcription |
| a_freq, t_freq, g_freq, c_freq | Per-base frequency of guide |
| gc_clamp | Binary: G or C at final guide position improves Cas9 binding |
| homopolymer | Longest single-base run (longer=higher off-target risk) |
| dinuc_repeat | Count of repeated dinucleotides |
| unique_dinucs | Number of unique dinucleotide pairs (low=repetitive sequence) |
| seed_unique_bases | Unique bases in seed region (low diversity=off-target risk) |
 
**80 positional features:** one-hot encoding of each of the 4 bases at each of the 20 guide positions (20*4=80 binary features). This encodes position-specific nucleotide preferences learned from the 5310 experimental measurements.
 
### Performance
 
r2=0.27 on held-out test set (80/20 split, random_state=42). This is comparable to Azimuth's reported performance on sequence-only features (~0.3-0.4). The gap between sequence-based r2 and real cutting efficiency reflects genuine biology: chromatin accessibility, local DNA structure, and cellular context all affect efficiency but are not encodable from sequence alone
 
---
 
## Tech Stack
 
| Component | Tool |
|---|---|
| Gene target selection | OpenTargets GraphQL API v4 |
| Sequence retrieval | Biopython + NCBI Entrez API |
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
| OpenTargets API | Disease-gene association scores (0-1) | Gene target selection and ranking |
| NCBI Nucleotide | Reference mRNA sequences via RefSeq | PAM scanning and 30mer context extraction |
| Doench et al. 2016 | 5310 experimental guide efficiency scores | XGBoost training data |
| PubMed via Entrez | Disease + CRISPR targeted paper abstracts | RAG document corpus (ChromaDB) |
 
All data frozen to disk on first fetch. Subsequent runs load from disk with no API calls. Cell 6 (8-disease loop) took 153 minutes on first run; all future reruns are instant.
 
---
 
## Cell Structure
 
| Cell | Purpose |
|---|---|
| Cell 1 | Setup, OpenTargets query, NCBI sequence fetch, PubMed paper fetch, data freezing |
| Cell 2 | Gene ranking from OpenTargets association scores, top 5 selection |
| Cell 3 | PAM scan, 30mer context extraction, XGBoost training and scoring, guide ranking |
| Cell 4 | LLM standalone baseline disease name only, no context |
| Cell 5 | RAG+LLM 3-section grounded prompt, Qwen generation, frozen output |
| Cell 6 | 8-disease inference loop full pipeline per disease, load model once |
| Cell 7 | 4-metric evaluation, per-disease table, visualization across all 8 diseases |
 
---
 
## Evaluation
 
Baseline (Cell 4) and RAG+LLM (Cell 5/6) are compared across 8 diseases using 4 metrics computed from response text via regex.
 
| Metric | What it measures | Baseline avg | RAG avg |
|---|---|---|---|
| top5_coverage | Fraction of top-5 XGBoost-ranked guides cited in response | 0.0 | 0.52 |
| score_grounding | Binary: did response cite a real XGBoost score (4 decimal match) | 0.0 | 1.0 |
| mean_cited_rank | Mean rank position of sequences cited (lower=better) | 0.0 | 2.29 |
| hallucination | Fraction of cited 20-mers not in ranked guide list | 0.38 | 0.06 |
 
`top5_coverage` and `score_grounding` are both 0.0 for all baseline responses. The baseline LLM cannot produce XGBoost scores or cite sequences that were computed by this pipeline and never appeared in any training data. This is the most concrete possible demonstration that RAG is adding information the model could not have
 
`mean_cited_rank` of 2.29 shows RAG is surfacing guides near the top of the ranked list, not random candidates from the bottom
 
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
 
## Key Findings
 
**RAG eliminates hallucinated guide sequences.** Baseline hallucination rate of 0.38 means 38% of sequences the LLM cited were fabricated plausible looking 20-mers that do not appear anywhere in the pipeline output. RAG drops this to 0.06. The 6% residual occurs in DMD where the model paraphrased 1 sequence with a single-base substitution
 
**Baseline cannot produce pipeline output under any prompt.** Score grounding is 0.0 for all 8 baseline responses because XGBoost scores with 4 decimal precision (e.g. 0.8752) could not exist in Qwen's training data this pipeline computed them. This is a cleaner demonstration than the Phase 1 guide citation metric because there is no ambiguity about whether the model memorized the sequences.
 
**XGBoost and hand coded rules recommend completely different guides.** The Phase 1 heuristic and Phase 2 XGBoost produce zero overlapping top guides for HBB. Both use the Doench 2016 paper as their source. The heuristic interprets rules from the paper text; XGBoost learns directly from the 5310 experimental measurements. When the paper's described rules diverge from what 5310 experiments actually showed, XGBoost reflects the data and the heuristic reflects the interpretation. Phase 2 scores are ground truth.
 
**OpenTargets gene selection is biologically defensible.** 5 of 8 diseases matched expected genes exactly. The 3 mismatches (BRCA2/RPE65/ABL1) reflect higher aggregate genetic evidence for those genes in the OT database, not API errors. Using OT top gene as ground truth is the correct approach for a system positioned as evidence-driven
 
---
 
## Challenges
 
### Challenge 1 OpenTargets v4 tractability schema mismatch
The tractability subfields (`smallmolecule { topBucket value }`, `antibody`, `otherModalities`) returned 400 errors v4 restructured this field. Dropped tractability and genetic sub-scores, kept overall `association_score` which is sufficient for gene ranking. Full tractability integration requires schema introspection against current v4 API.
 
### Challenge 2 XGBoost r2 plateau at 0.08 with basic features
Initial 8-feature model (GC content, poly-T, seed GC, position, base frequencies) achieved r2=0.08. Solved by adding positional nucleotide encoding: one-hot encoding of each base at each of the 20 guide positions (80 binary features). r2 jumped from 0.08 to 0.27. This encodes position-specific nucleotide preferences that the sequence-level features cannot capture Cas9 cuts more efficiently when certain bases appear at specific positions regardless of overall GC content.
 
### Challenge 3 Off-target scoring without genome alignment
True off-target analysis requires Bowtie or BLAST against the 3GB human genome reference not feasible in a notebook. Solved by decomposing sequence complexity into 4 independent features (`homopolymer`, `dinuc_repeat`, `unique_dinucs`, `seed_unique_bases`) that capture different shapes of repetitiveness. XGBoost learns the weight of each from 5310 Doench experiments instead of a hand-tuned point deduction.
 
### Challenge 4 XGBoost score compression, top guide stuck at 0.74
Initial 8-feature model had poor score separation rank 1 guide scored ~0.74 and most candidates bunched near 0.4. Resolved iteratively: positional encoding pushed top score from 0.74 to 0.879, complexity features improved distribution shape. Final top guide for HBB: 0.8752 with 42 candidates passing threshold and reasonable spread across the distribution.
 
### Challenge 5 Heuristic vs XGBoost guide disagreement
Phase 1 heuristic and Phase 2 XGBoost recommend completely different top guides for HBB with zero overlap. Both reference Doench 2016. The heuristic approximates the paper's described rules with hand-assigned weights. XGBoost fits directly to the raw experimental data. Different guides get picked because approximated rules diverge from what 5310 real cutting experiments actually showed. This is expected and desirable XGBoost is what the data says, the heuristic is what the paper says the data says
 
### Challenge 6 ChromaDB MMR retrieval ignored ML pipeline guide RNA docs
MMR retrieval consistently returned 5-6 PubMed abstracts and excluded the guide RNA document and OpenTargets scores document, causing the LLM to hallucinate sequences instead of citing XGBoost-ranked ones. Root cause: structured tabular data (ranked sequences + decimal scores) has low semantic similarity to the CRISPR design query compared to full paper abstracts. MMR's diversity penalty then bumped the guide doc out of the retrieved set. Fix: removed both from ChromaDB entirely, hardcoded them as guaranteed sections in the prompt. MMR now only handles PubMed literature retrieval where diversity actually helps
 
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
 
```
pip install biopython requests xgboost scikit-learn pandas numpy matplotlib \
            transformers torch langchain langchain-community langchain-core \
            chromadb sentence-transformers
```
 
Run cells in order: 1, 2, 3, 4, 5, 6, 7. Cell 6 will take ~2.5 hours on first run (8 diseases, loads Qwen once for all). All data fetched from public sources -- no restricted access required.
 
---
 
## Future Work
 
**Phase 2 extensions:**
- Bowtie-based genome wide off-target scoring against GRCh38
- Additional training datasets (Leenay 2019, DeepCRISPR) for improved r2
- Reranker layer on top of MMR for better retrieval quality
- Streamlit app with disease-name input and live guide output
**Phase 3:**
- Problem 1: mutation detection in patient sequences
- Problem 2: literature-based target gene selection beyond OT scores
- End-to-end pipeline connecting all 3 problems
- Multi-disease batch processing with structured output
---
 
## Impact
 
Phase 1 reduced guide RNA selection from 8-10 hours to minutes. Phase 2 removes the prerequisite: the user no longer needs to know which gene to target. Disease name in, ranked guide RNAs out with association scores from 20,000+ studies (OpenTargets), efficiency predictions from 5310 real experiments (Doench 2016), and biological rationale grounded in PubMed literature
 
The pipeline combines:
- Evidence-based gene selection (OpenTargets)
- Experimentally-grounded guide scoring (XGBoost on Doench 2016)
- Literature-grounded LLM reasoning (RAG + Qwen2.5-7B)
The output is not just ranked sequences it is ranked sequences with predicted efficiency scores, genomic position context, and LLM-generated biological rationale backed by retrieved literature.
 
---
 
## Project Context
 
Built as part of a self-directed AI/ML specialization alongside a Biotechnology undergrad degree at NSUT Delhi. The Bio + AI positioning is intentional this project requires both ML engineering (XGBoost feature engineering, RAG architecture, VRAM management) and biomedical domain reasoning (PAM biology, seed region specificity, OpenTargets evidence schema). Phase 1 is available at: https://github.com/Ishan-2-0/CRISPR-Cas9-Engineering-Platform
 
Live Demo (Phase 1): https://huggingface.co/spaces/i-chan/CRISPR-Cas9-Engineering-Platform-RAG
 