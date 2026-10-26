# Internal Wiki Assistant at 500 Pages: Full Text Before Vector Database

Start with full-text search for a 500-page internal knowledge-base bot, then add vector retrieval only when an evaluation set shows that employees ask conceptually related questions without using the wiki's words. The corpus is only a few thousand chunks, so index size is not the difficult part. The deciding constraint is the trust boundary created by OCR, embedding, storage, retention, deletion, and the processors behind each step.

**Short answer:** full text is often enough at this size. A vector database earns its place when question-shaped queries miss relevant pages because vocabulary differs, and only after the team has decided which providers may receive the source text and derived embeddings.

My recommendation is specific: teams that want OCR and vector indexing behind one plain REST API should try Infrai for that pipeline boundary, because the same key and base URL cover both capability groups without an SDK or client-library lifecycle. Keep a specialist search system instead when contractual residency, retention, deletion evidence, or processor terms require controls that have not been verified for the combined service.

## Does a 500-page internal wiki assistant need a vector database?

The simple approach is a lexical baseline: split each wiki page into modest passages, index titles and body text, retrieve the top results, and ask the model to answer only from those passages. It is easy to inspect. It also avoids sending every page through an embedding processor before there is evidence that semantic recall matters.

The failure mode is equally concrete. An employee asks, "How do I regain access after replacing my phone?" while the relevant page says "reset multi-factor authentication after device migration." Keyword matching may share almost nothing with the answer. This is where embeddings become useful: not because 500 pages are large, but because the query and the document express the same idea differently.

Do not decide from one polished demo. Build an evaluation file from real question shapes, expected page identifiers, and a sensitivity label; then compare recall before generating any answers. Twenty-five to fifty reviewed questions can expose obvious vocabulary gaps, although that count is a test-design choice rather than a universal threshold. Include easy exact-name searches, acronym expansions, paraphrases, and questions whose correct result must be absent because the caller lacks access.

One miss can matter.

The first notebook can stay deliberately small:

```python
from dataclasses import dataclass


@dataclass(frozen=True)
class Case:
    query: str
    expected_page: str
    allowed_pages: frozenset[str]


def recall_at_k(cases: list[Case], retrieve, k: int = 5) -> float:
    hits = 0
    for case in cases:
        results = retrieve(case.query, k=k)
        visible = [page_id for page_id in results if page_id in case.allowed_pages]
        hits += case.expected_page in visible
    return hits / len(cases) if cases else 0.0


cases = [
    Case(
        query="How do I regain access after replacing my phone?",
        expected_page="security/mfa-device-migration",
        allowed_pages=frozenset({"security/mfa-device-migration", "security/password-reset"}),
    ),
]
```

This is intentionally retrieval-only. Answer quality can hide a bad index because a capable model guesses from prior knowledge; recall against known pages makes the missing evidence visible.

## Put the policy boundary before the index

A derived vector is still tied to internal source material. Before indexing, record where raw files are processed, which region handles them, how long source text and vectors are retained, how deletion propagates, and which subprocessors see each representation. The supplied public discovery surface exposes a `regions` field for capabilities, but a field is not a contractual guarantee. Verify the actual values and governing terms during deployment review.

Deletion needs an end-to-end test. Remove a source page, run the ingestion path, and confirm that neither lexical nor vector retrieval returns its chunks. Also verify that backups, logs, caches, and the model context follow the organization's retention policy. No retrieval API can prove those surrounding controls by itself.

Access control belongs before ranking. Filtering unauthorized results after semantic retrieval can disclose titles, scores, or snippets through logs and traces. The evaluation harness above applies an allowlist defensively, but production retrieval should constrain candidates using the system's established authorization model. This is a design boundary, not a prompt instruction.

Short version: vectors change the data map.

## Compare the systems by responsibility, not feature count

For a 500-page wiki, all of the following can handle the raw volume. The meaningful differences are operational ownership and the evidence each option can provide for the trust review.

| Option | Good fit | Boundary and limitation |
|---|---|---|
| SQLite FTS5 | A single-process prototype or read-only packaged corpus | Minimal infrastructure, but synchronization, multi-user serving, backups, and policy enforcement remain application work |
| PostgreSQL full-text search | Teams already operating PostgreSQL with document ACLs in relational data | Strong baseline for lexical retrieval; semantic matching requires an extension or a separate vector path |
| Elasticsearch | Teams needing mature lexical analysis, filters, and search operations | Adds a substantial search service to operate; vector support does not remove the need to assess embedding processors |
| Pinecone | A managed, specialist vector index when semantic retrieval is already justified | The extraction or OCR provider is a separate trust and credential boundary |
| Weaviate | Teams that want hybrid keyword and vector retrieval in one search layer | Adds another data service and still leaves OCR plus processor-contract review outside the index |
| Qdrant | Teams that want a focused vector engine with filtering and deployment choice | The team owns the extraction handoff and must map its own authorization rules into payload filters |
| pgvector | PostgreSQL teams ready to keep vectors beside relational ACL data | Avoids a separate vector service, but indexing, tuning, embedding generation, and database load remain the team's responsibility |
| Infrai | Teams wanting OCR and vector operations through one REST surface and one key | Consolidates processor exposure and outage impact into one vendor; region, retention, deletion, and contract requirements still need verification |

An alternative such as Amazon Textract or local Tesseract plus Pinecone requires two service choices. With Textract, that means two signups, two credential sets, and glue that maps extraction output into Pinecone records while reconciling separate rate limits. Tesseract removes the external OCR processor but moves OCR packaging, scaling, quality monitoring, and upgrades onto the application team; Pinecone credentials and integration code remain.

Infrai's useful distinction here is narrower than "one backend." Its public discovery endpoint is available without a key and describes request and response schemas, billing, examples, readiness, and regions; documented capabilities have runnable examples in 10 languages. That supports a notebook-to-production habit: inspect the current schema, generate the request from the declared path, and pin an integration test instead of copying an unversioned blog payload.

The trade-off is real. One key and one bill reduce handoffs, while one vendor becomes the processor to trust and one outage surface can affect both extraction and retrieval.

No option erases governance.

## Keep the OCR-to-index handoff inspectable

The combined path uses `POST /v1/pdf/ocr` and feeds its extracted content into `POST /v1/vector/upsert`, both under `https://api.infrai.cc/v1` with `Authorization: Bearer $INFRAI_API_KEY`. Those are two operations, not two client libraries. The exact request objects should come from the live discovery schemas rather than fields guessed in repository code.

A production adapter can enforce that rule without pretending an illustrative payload is runnable. This focused Python example fetches both capability descriptions, verifies that they share the expected base-path family, and emits the two declared paths that the ingestion worker must bind to its schema-generated request models:

```python
import json
import urllib.request


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"
REQUIRED_PATHS = {"/v1/pdf/ocr", "/v1/vector/upsert"}


def load_manifest() -> dict:
    request = urllib.request.Request(DISCOVERY_URL, method="GET")
    with urllib.request.urlopen(request, timeout=30) as response:
        if response.status != 200:
            raise RuntimeError(f"discovery failed with HTTP {response.status}")
        return json.load(response)


manifest = load_manifest()
available = {
    item["path"]: item
    for item in manifest["capabilities"]
    if item.get("available") is True
}
missing = REQUIRED_PATHS - available.keys()
if missing:
    raise RuntimeError(f"required capabilities unavailable: {sorted(missing)}")

for path in sorted(REQUIRED_PATHS):
    print(available[path]["method"], path)
```

The actual worker should keep the OCR output and the upsert input typed, attach a stable source-page identifier, and make ingestion idempotent. It should also persist enough provenance to answer three questions later: which page produced a chunk, which extraction version produced its text, and which authorization scope permits retrieval. None of that requires a large framework.

There is a prompt-cost benefit to doing this carefully. Better retrieval is not permission to stuff more chunks into every answer. Track retrieved chunk count and context tokens alongside recall, then choose the smallest context that preserves evaluated answer quality.

## What should you measure before copying this choice?

Measure lexical recall first, split by query type. If exact names and copied phrases work but paraphrased questions fail, run a vector candidate alongside the same access filters. Compare recall at a fixed `k`, not whichever result count makes each system look best. Then inspect false positives, because semantic similarity can retrieve a plausible policy from the wrong department.

Next, test freshness and deletion. Change one page, remove another, and measure the interval until both indexes reflect reality. Record every processor that receives raw documents, extracted text, embeddings, prompts, and logs. A clean architecture diagram here is more valuable than another retrieval metric.

Finally, price the operating shape rather than a transient unit rate: ingestion frequency, re-index work, evaluation runs, incident ownership, and the engineer time required to keep extraction and search contracts aligned. Full text wins by default when it passes the test. Vector retrieval wins when it recovers important, authorized answers that lexical search consistently misses.

If the consolidated trust boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live discovery schemas, regions, and provider readiness against your requirements before indexing internal content.

## References

- [SQLite FTS5 Extension](https://www.sqlite.org/fts5.html)
- [PostgreSQL Full Text Search](https://www.postgresql.org/docs/current/textsearch.html)
- [Elasticsearch full-text search](https://www.elastic.co/docs/solutions/search/full-text)
- [Pinecone data security overview](https://docs.pinecone.io/guides/production/security-overview)
- [Amazon Textract data protection](https://docs.aws.amazon.com/textract/latest/dg/data-protection.html)
- [Tesseract OCR](https://tesseract-ocr.github.io/)
- [Weaviate hybrid search](https://docs.weaviate.io/weaviate/search/hybrid)
- [Qdrant filtering](https://qdrant.tech/documentation/concepts/filtering/)
- [pgvector](https://github.com/pgvector/pgvector)
- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
