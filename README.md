# Medly — Arabic RAG Medical Chatbot

An Arabic-language biomedical question-answering system. Retrieval-augmented generation over a 216,102-sample medical corpus, with MedCPT encoders, FAISS search, cross-encoder re-ranking, and Gemini for generation.

**Result:** faithfulness score of **83%** on RAGAS.

---

## The problem

Reliable medical information is hard to find and harder to read. For Arabic speakers it is harder still: the large, high-quality biomedical corpora are in English, and the Arabic datasets that exist are fragmented across dialects and need heavy preprocessing.

Medly answers Arabic medical questions in Arabic, while retrieving from an English corpus through end-to-end translation. A plain LLM would answer too, but it would hallucinate; grounding every answer in retrieved documents is the point.

## How it works

```
Arabic query → translate → MedCPT Query Encoder → FAISS (cosine similarity)
                                                        ↓
                                              top 20 documents
                                                        ↓
                                        MedCPT Cross-Encoder re-ranking
                                                        ↓
                            top 4 → answer prompt  ┐
                            next 4 → recommendations ┘ → Gemini → translate → Arabic answer
```

**Retrieval.** The corpus is embedded with the MedCPT document encoder and indexed in FAISS. Queries go through the matching query encoder, so questions and documents share one semantic space. MedCPT is built for zero-shot biomedical retrieval and trained on 255M query-article pairs from PubMed search logs.

**Re-ranking.** FAISS is fast but approximate. The top 20 candidates are re-scored by the MedCPT cross-encoder, which processes query and document together and is far more accurate — and far too expensive to run over the whole corpus. This is the classic retrieve-then-rerank trade-off.

**Generation.** The top 4 re-ranked documents build a zero-shot instructional prompt with context injection, passed to Gemini. The next 4 documents — deliberately *not* the ones used for the answer — build a second prompt for lifestyle recommendations and related reading. Using lower-ranked documents here widens the material available to the model and makes it more likely to surface prevention and behavior content the direct answer missed.

The recommendation prompt carries strict instructions against clinical or medication advice.

## Corpus

| Source | Samples | Share |
|---|---|---|
| MedMCQA | 192,331 | 89% |
| MedQuAD | 17,288 | 8% |
| BioASQ | 4,322 | 2% |
| Kaggle | 2,161 | 1% |
| **Total** | **216,102** | |

## Evaluation

Evaluated with RAGAS on a 100-question test set, built from the most-asked medical questions of the past five years. A separate 100-question validation set drove prompt iteration and encoder selection.

**Faithfulness: 0.83.** How factually consistent the response is with the retrieved context — the metric that matters most here, since the system's purpose is to not invent medical claims.

**Answer relevancy: 0.52,** and unstable. It was near-perfect on many questions and zero on others. Answer relevancy works by reconstructing the original question from the response, which penalizes a system designed to explain rather than mirror the query back. The metric is informative but misaligned with what Medly is for — worth reporting honestly rather than quietly dropping.

## Design decisions worth noting

**English corpus, Arabic interface.** The original intent was a fully Arabic corpus. The available Arabic medical datasets were smaller, dialectally inconsistent, and would have needed preprocessing beyond the compute available. Translation at both ends was the pragmatic trade-off, and it is a real limitation: translation errors propagate into retrieval.

**Two-stage retrieval.** Bi-encoder for recall, cross-encoder for precision. Running the cross-encoder alone over 216K documents is not feasible.

**Separate document pools for answers and recommendations.** Reusing the same top documents for both would produce recommendations that restate the answer.

## Stack

Python · MedCPT (PubMedBERT-based) · FAISS · Gemini · Streamlit · RAGAS

## Future work

User profiles (age, medical history) for personalized recommendations; specialist and hospital suggestions by location; feedback loops for iterative improvement.

Full paper: [`NLP_Project_Paper.pdf`](./NLP_Project_Paper%20(1).pdf)

## Authors

Sarah Hassouneh, Sondos Qasarwa, Rana Mosalam — Department of Electrical & Computer Engineering, Birzeit University.
