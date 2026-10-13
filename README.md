# SejongQA: An Evidence-Grounded Diagnostic Benchmark for Korean Historical Question Answering

Benchmark data, execution logs and scoring code for the SejongQA paper (submitted to ICEIC 2027).

SejongQA is a test set of 975 Korean questions on the early reign of King Sejong (1418–1427). The questions were built from the modern Korean translation of the *Veritable Records of King Sejong* (Sejong Sillok) by the Institute for the Translation of Korean Classics (ITKC). Every question has an answer, a list of accepted answer aliases, its gold articles (the source articles cited as evidence) and quotations from them, so answer accuracy and gold-article retrieval can be looked at together for each question type. There is no training or development split.

| Type | Questions | Design |
|---|---|---|
| Alias | 150 (75 pairs) | the same question about one person under two different names or titles; both answers are the same |
| Confusable | 150 (75 pairs) | the same question frame for two easily confused people; the answers differ |
| Temporal | 300 (150 pairs) | one anchor article with two different temporal expressions; the target articles and answers differ |
| Relation Chain | 375 | facts linked across 2, 3, 5 or 7 wiki pages (92, 98, 100 and 85 questions); only the starting entity is named |

Alias and Confusable together form the Identity dimension in the paper.

## Checking the numbers in the paper

The scripts need only Python 3 and its standard library (tested with 3.10 and 3.14). No API key or network access is needed.

```
python scripts/check_paper.py
python scripts/score.py runs/gold/answers.jsonl
```

`check_paper.py` recomputes the values in Tables I–III and the counts given in the text, prints each one next to the number in the paper, and exits with status 1 if any of them differ. `score.py` reports EM, Pair EM and abstention by question type for one answer file. It accepts any JSONL file with `id` and `answer` fields and scores only the questions the file contains.

## Files

```
data/
  questions.jsonl            975 questions (give a system only the question text)
  answers.jsonl              answers, aliases, gold article IDs, Temporal expression category
  evidence.jsonl             quotations, Temporal anchor/target articles, Relation Chain steps
  article_ids.tsv            the 4,273 corpus article IDs with text length and SHA-256 (no text)
  excluded_article_ids.tsv   the 7,030 article IDs removed before wiki construction (no text)
  identity_names.tsv         the two names or titles used in each Identity pair
prompts/
  reader_closed.txt          answer prompt for Parametric Knowledge
  reader_gold.txt            answer prompt for Gold-context (also used for gold_gpt41)
  reader_vanilla.txt         answer prompt for Vanilla RAG
  question_generation_rules.md      rules for drafting candidate questions (summary)
  independent_solving_protocol.md   protocol of the LLM verification step
graphrag/
  settings.yaml              Microsoft GraphRAG 2.7.2 settings used for indexing and search
  prompts/                   indexing prompts and the Local/Basic Search prompts
runs/                        execution logs, one folder per condition
  closed/                    Parametric Knowledge
  gold/                      Gold-context
  vanilla/                   Vanilla RAG; retrieval.jsonl has the 10 retrieved chunks per question
  graphrag/                  GraphRAG Local Search
  graphrag_basic/            GraphRAG Basic Search
  graphrag_ctx/              200 questions re-answered from stored Local Search contexts
  gold_gpt41/                Gold-context with gpt-4.1 as the answer generator (Table II, note c)
scripts/
  check_paper.py             recomputes the numbers reported in the paper
  score.py                   EM, Pair EM and abstention by question type
  rebuild_corpus.py          rebuilds the corpus and the masked prompt examples
  vanilla_retrieval.py       optional: reruns Vanilla RAG retrieval on a rebuilt index
MANIFEST.json                version, freeze date, models and SHA-256 hashes of all other files
```

The folder names are the ones we used while running the experiments: `closed` is Parametric Knowledge, `graphrag` is Local Search and `graphrag_basic` is Basic Search. In `runs/graphrag_ctx/sample.json`, Multi-hop means Relation Chain, and same-person and different-person mean Alias and Confusable.

Each `runs/*/answers.jsonl` has one line per question with `id`, `answer` (the raw model output) and `context_docs`, the articles whose text reached the answer generator (the gold articles for Gold-context, none for Parametric Knowledge). Basic Search lines also give `context_units`, the GraphRAG text units in the context.

## Data format

Field values are in Korean. Article IDs have the form `YY-MM-DD[NN]`: regnal year (`00` is the accession year, 1418), lunar month, day and the number of the article within that day. Leap months carry `윤` (`04-윤12-26[02]`), and the one article without a recorded day is `09-08-○○[01]`. Question IDs start with `PJ` (Identity), `TR` (Temporal) or `MH2`, `MH3`, `MH5`, `MH7` (Relation Chain, by chain length). The two questions of a pair end in `a` and `b`.

| Field and value | Meaning |
|---|---|
| `category`: `인물판정` / `시간추론` / `멀티홉` | Identity / Temporal / Relation Chain |
| `sub`: `같은 사람` / `다른 사람` | Alias / Confusable (empty for Temporal) |
| `sub`: `2hop` / `3hop` / `5hop` / `7hop` | 2-, 3-, 5-, 7-document chain |
| `relation_family`: `하루` / `달` / `해` / `순서` | day / month / year / order |
| `relation`: `전날` / `이튿날` | previous day / following day |
| `relation`: `그달 초` / `그달 말` / `전월` | early in the same month / late in the same month / previous month |
| `relation`: `그해 초` / `그해 중순` / `그해 말` | early / middle / late in the same year |
| `relation`: `전해` / `이듬해` / `후년` | previous year / following year / two years later |
| `relation`: `직전` / `직후` | immediately before / immediately after |

`questions.jsonl` has `id`, `category`, `sub`, `pair_id` (null for Relation Chain) and `question`.

`answers.jsonl` has `id`, `category`, `sub`, `pair_id`, `answer`, `answer_aliases` (other accepted forms), `evidence_ids` (gold article IDs) and `relation_family` (Temporal only). It also has `n_evidence`, `hops`, `relation`, `alias_type` and `alias_class`, which neither the paper nor the scripts use. We kept them so that the file stays byte-identical to the frozen answer key whose hash is in the run manifests.

`evidence.jsonl` has `id`, `source_ids` (the same list as `evidence_ids`) and `evidence`, a list of `{source_id, quote}`. Identity items add `alias_type`. Temporal items add `relation`, `family`, `anchor` and `target`, the last two as `{source_id, event}`. Relation Chain items add `hops` and `chain`, a list of k−1 steps `{step, from, relation, to, source_ids, quote}` for a k-document chain. Quotations are copied from the source articles and differ from them only in whitespace, except one chain quotation in MH2-007 that joins two passages with "…". `event`, `from`, `relation` and `to` are short paraphrases from the question drafts, not quotations.

`article_ids.tsv` has `id`, `n_chars` and `sha256_text`, `excluded_article_ids.tsv` has `id`, and `identity_names.tsv` has `pair`, `sub`, `name_a` and `name_b`.

## How the questions were built

The source is ITKC's modern Korean translation of the Sejong Sillok in the Korean Classics DB (https://db.itkc.or.kr): the 11,275 articles from Sejong's accession year to his ninth regnal year (1418–1427). We collected 11,275 of the 11,290 articles that the database lists for this period. The 15 we missed (03-01-25[03–04], 04-01-22[03], 04-12-22[04], 07-01-09[04–05], 08-02-17[03], 08-08-02[04–08], 08-11-18[03], 08-12-27[04–05]) are the last entries of eight days; they are not in the corpus, the wiki or any question. Before building the wiki, we removed short routine articles (brief records of regular rites, royal outings and portents, and short reports that list only appointments or punishments) with rules based on recurring phrases and article length. The rules looked only at the article text and were applied before any question was drafted. The 7,030 removed IDs are in `data/excluded_article_ids.tsv`.

From the remaining articles, Claude Opus 5 built 176 linked wiki pages following Karpathy's LLM Wiki pattern (https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f). Each wiki statement cites its source articles, and every quotation in the wiki was checked against the cited article. We used the wiki only to find question candidates and relevant articles. It is not released, and no evaluated system had access to it.

Claude Opus 5.5 agents drafted candidate questions from the alternate names, confusable people, dates and page links recorded in the wiki (`prompts/question_generation_rules.md`). A different Claude Opus 5.5 agent then answered each candidate from the question and the full gold articles alone. When the two answers disagreed, the question was checked against the articles and revised or dropped, and a revised question went through the same check again (`prompts/independent_solving_protocol.md`). Finally, two reviewers checked the question, answer and quotations of every question against the source articles and removed 25 questions, leaving 975. Questions, answers and aliases were frozen before the first evaluation run and have not changed since.

## Retrieval corpus

The retrieval corpus is the 4,273 articles that the wiki cites or that its construction log records (`data/article_ids.tsv`): the 4,245 articles from 1418–1427 left after filtering (11,275 − 7,030) and 28 articles from regnal years 10–32. All 773 gold articles are from 1418–1427. Every retrieval index in the paper was built from these articles.

We do not redistribute the article texts. To rebuild the corpus:

1. Get the articles from the Korean Classics DB under its terms of use and save them as JSONL, one article per line with `article_ref`, `date_heading` and `body_text`. The exact text format is described at the top of `scripts/rebuild_corpus.py`.
2. Run `python scripts/rebuild_corpus.py my_articles.jsonl corpus/articles.csv`. Each text is checked against the length and SHA-256 in `data/article_ids.tsv`. If all 4,273 match, the CSV is byte-identical to the file used in the paper (`corpus.articles_csv_sha256` in `MANIFEST.json`).
3. The two example articles in `graphrag/prompts/extract_graph.txt` are masked. Step 2 also fills them in, writes `graphrag/prompts/extract_graph.restored.txt` and checks it against the hash of the prompt used in the paper (`--restore-prompt corpus/articles.csv` repeats only this step). Use the restored file as `extract_graph.txt` when you index. It contains source text, so do not commit it.
4. To rebuild the GraphRAG index, put `articles.csv` in the `input/` folder of a GraphRAG 2.7.2 project that uses `graphrag/settings.yaml` and `graphrag/prompts/`, and run `graphrag index`. This calls the OpenAI API, which is billed.

## Evaluation setup

All conditions used gpt-4.1-mini (gpt-4.1-mini-2025-04-14) at temperature 0 as the answer generator, and each condition was run once.

| Condition | What the answer generator receives |
|---|---|
| Parametric Knowledge | the question only |
| Gold-context | the question and the full text of its gold articles, without retrieval |
| Vanilla RAG | the question and the 10 chunks closest to it by embedding similarity; no graph, reranking or query rewriting |
| Local Search | the context built by Microsoft GraphRAG 2.7.2 Local Search (entities, relationships, community reports and text chunks) |
| Basic Search | the context built by GraphRAG 2.7.2 Basic Search (top 10 chunks, no graph) |

| Setting | Value |
|---|---|
| Embedding model | text-embedding-3-small |
| Chunks | 1,200 tokens with a 100-token overlap; the same 4,600 chunks for Vanilla RAG and GraphRAG |
| GraphRAG index | built with gpt-4.1-mini; entity types person, official position, government office, place, country, event, institutional system, document |
| Local Search | community level 2, text-unit proportion 0.5, community-report proportion 0.25 |
| Context limit | 12,000 tokens (Local and Basic Search) |
| Output limit | 100 tokens for Parametric Knowledge, Gold-context and Vanilla RAG; none for Local and Basic Search |

Parametric Knowledge, Gold-context and Vanilla RAG use the system prompts in `prompts/`, which share one answer instruction. Their user message is the question alone (Parametric Knowledge) or `"[자료]\n" + "\n\n---\n\n".join(contexts) + "\n\n[질문]\n" + question`, where the contexts are the full gold articles in `evidence_ids` order (Gold-context) or the ten retrieved chunks in rank order (Vanilla RAG). Local and Basic Search use the default GraphRAG search prompts, with an equivalent answer instruction passed as the response type (`response_type` in their manifests).

The indexing prompts `extract_graph`, `summarize_descriptions` and `community_report_graph` were generated on this corpus with GraphRAG's `prompt-tune` command, and we edited `extract_graph` by hand afterwards. The other prompt files are GraphRAG defaults. Global Search, DRIFT Search and claim extraction were not used. Their prompt files are still named in `settings.yaml` but are not included.

Two more runs support statements in the paper. `runs/graphrag_ctx/` takes the stored Local Search contexts of 200 questions sampled by type (listed in `sample.json`) and answers them again with the Gold-context answer instruction and the 100-token limit. Its system prompt, given in its manifest, differs from `reader_gold.txt` only in describing the input as tables of entities, relationships, reports and chunks. The contexts themselves are not included. `runs/gold_gpt41/` repeats Gold-context with gpt-4.1 (gpt-4.1-2025-04-14).

## Metrics

An answer is correct under Exact Match (EM) if, after normalization, it equals the reference answer or one of its aliases. Normalization applies Unicode NFKC, removes text in parentheses, Chinese characters, punctuation and whitespace, and lowercases English, so `남지(南智)` and `남지` count as the same answer. Pair EM is the share of contrastive pairs with both questions correct, computed separately for the 150 Identity pairs and the 150 Temporal pairs.

Systems were told to answer `근거 부족` ("insufficient evidence") when the evidence was not enough. These abstentions are counted separately and scored as wrong.

Retrieval Hit Rate is the share of all 975 questions for which at least one gold article reached the answer generator's input. An article counts as retrieved if any of its chunks did. `check_paper.py` also reports the share of questions with all gold articles retrieved and, for Temporal questions, the share with both the anchor and the target article retrieved.

## Run manifests

Each `runs/*/manifest.json` records the model, temperature, output limit, prompt or prompt hash, retrieval settings and run date. `questions_sha256` and `answers_sha256` are the SHA-256 of `data/questions.jsonl` and `data/answers.jsonl` exactly as released. Both files have CRLF line endings, and `.gitattributes` keeps Git from converting them. In `runs/gold_gpt41/manifest.json`, `run_answers_sha256` is the hash of that run's own `answers.jsonl`. The `smoke_test` note means the first five questions were answered in an earlier test call with the same settings, and those answers are part of the 975. We removed API cost figures from the public manifests.

`graphrag/settings.yaml` is the run configuration with its comments removed and the LanceDB path written with a forward slash, so its hash differs from `settings_sha256_at_run`. The GraphRAG prompt files match `prompts_sha256` in `runs/graphrag/manifest.json`, except `extract_graph.txt`, which matches only after the masked examples are restored.

## Quoted source text

`data/evidence.jsonl` quotes short passages from the ITKC translation (each at most 186 characters) to document the evidence for each answer. The translation is © ITKC, and the quotations are not covered by our licenses. If the rights holder objects, we will remove the `quote` fields and keep the article IDs.

## License

`data/`, `runs/`, `prompts/` and `graphrag/settings.yaml` are released under CC BY-NC 4.0, and the code in `scripts/` under the MIT License. `graphrag/prompts/` is adapted from Microsoft GraphRAG and stays under its MIT License. See `LICENSE.md` for details.

## Citation

The paper has been submitted to ICEIC 2027. Until it is published, please cite:

```bibtex
@misc{kim2026sejongqa,
  title        = {SejongQA: An Evidence-Grounded Diagnostic Benchmark for Korean Historical Question Answering},
  author       = {Kim, Hyonam and Choi, Seungyun and Sun, Wookyung and Lee, Jungwon},
  year         = {2026},
  howpublished = {\url{https://github.com/dawnnam02/SejongQA-ICEIC}},
  note         = {Submitted to ICEIC 2027}
}
```

## Contact

Hyonam Kim, Chung-Ang University (dawnnam@cau.ac.kr). Please report errors in the data as GitHub issues.
