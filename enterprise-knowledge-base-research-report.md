# AI Research Report (Part I): A Four-Layer Enterprise Knowledge Base Strategy

Research date: October 18, 2026

## 0. Background and Core Assessment

The company wants to use AI to help the data team understand business requirements, select tables, retrieve data, and create visualizations. The existing knowledge base consists mainly of several hundred Markdown Wiki documents covering food-delivery business context, tracking-event logic, base-table semantics, metric definitions, and SQL.

Claude Code currently searches the Wiki with grep/glob. A common failure mode is that it finds the correct document but not the correct content inside it. Other issues include overly long documents, duplicated shared knowledge, obsolete documents that are not removed, and conflicting definitions across documents.

An enterprise knowledge base can therefore be considered in four layers:

1. **Knowledge content and organization:** How knowledge is separated, reused, and referenced.
2. **Knowledge discovery and retrieval:** How AI finds the correct domain, document, and section.
3. **Knowledge governance and lifecycle:** How authority, freshness, and maintenance responsibility are determined.
4. **Knowledge use and context assembly:** How relevant knowledge is supplied to AI on demand without overloading its context.

The knowledge base should store knowledge about how to understand and query data, not live business data. Actual values should always be queried from databases.

---

## 1. Knowledge Content and Organization

Common approaches evolve from free-form documentation toward modular, structured, and platform-based knowledge. They can be combined and adopted incrementally.

### Representative Industry Approaches

| Approach | Form | Advantages | Limitations | Evidence |
|---|---|---|---|---|
| Large-document Wiki | One long document per business area, combining context, definitions, tables, and SQL | Flexible authoring and the lowest migration cost | Duplicate content; difficult partial updates; AI can lose focus inside long documents | Traditional Wiki/full-text search model, included as the baseline |
| Templated documents | Retain documents but standardize summaries, tables of contents, sections, and metadata | Low migration cost; substantially improves within-document navigation | Shared definitions may still be copied into multiple documents | Anthropic Skills use YAML metadata to describe what a Skill does and when to use it, with the body containing instructions. [Anthropic documentation](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) |
| Modular knowledge packages | Separate workflows, references, SQL/scripts, and supporting material; the main document contains only task and selection logic | Shared knowledge becomes reusable; updating one module affects all consumers | Requires stable IDs, reference conventions, and directory standards | Anthropic Agent Skills separate `SKILL.md`, reference files, and scripts into composable packages. [Anthropic documentation](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) |
| Structured business terminology | Give metrics and terms canonical IDs, definitions, aliases, and explicit relationships | Creates a shared business language and reduces ambiguity in synonyms and definitions | Requires continuous maintenance by business owners | Google Dataplex Business Glossary supports categories, terms, synonyms, related terms, and up to three levels of category nesting. [Google Cloud documentation](https://cloud.google.com/dataplex/docs/create-glossary) |
| Structured data assets | Treat tables, fields, and metrics as individual assets with owners, status, lineage, and scope | Supports table discovery, field selection, and dependency understanding | Connecting data sources and maintaining relationships costs more | DataHub manages tables, fields, dashboards, lineage, and ownership as unified metadata. [DataHub documentation](https://docs.datahub.com/docs/managed-datahub/build-a-data-agent/key-concepts) |
| Metadata/context platform | Manage business and technical metadata, assets, lineage, quality, and query examples together | Provides unified data context for both people and AI | Too heavy if the immediate goal is only to fix long documents | DataHub originated at LinkedIn and unifies business and technical metadata; it had about 12.8k GitHub stars at the research snapshot. [DataHub GitHub](https://github.com/datahub-project/datahub) [Metadata standards](https://github.com/datahub-project/datahub/blob/master/docs/metadata-standards.md) |

### How the Approaches Relate

```mermaid
flowchart LR
    A["① Large-document Wiki<br/>Free-form recording"]
    B["② Templated documents<br/>Standardized structure"]
    C["③ Modular knowledge packages<br/>Separation and reuse"]
    D["④ Structured business terminology<br/>Canonical concepts"]
    E["⑤ Structured data assets<br/>Tables, fields, and lineage"]
    F["⑥ Metadata/context platform<br/>Cross-system aggregation"]

    A -->|"Standardize sections"| B
    B -->|"Separate and reference"| C
    C -->|"Add IDs and aliases"| D
    D -->|"Link technical assets"| E
    E -->|"Automate synchronization and governance"| F

    classDef current fill:#F3F4F6,stroke:#7A8494,color:#303640,stroke-width:2px;
    classDef doc fill:#EAF2FF,stroke:#4C7FE8,color:#183B73,stroke-width:2px;
    classDef module fill:#E8F7F1,stroke:#35A579,color:#155B43,stroke-width:2px;
    classDef term fill:#FFF4DD,stroke:#E3A52B,color:#754E00,stroke-width:2px;
    classDef asset fill:#FFF0E8,stroke:#E67E45,color:#743819,stroke-width:2px;
    classDef platform fill:#F2EAFE,stroke:#8A5BD1,color:#4B287E,stroke-width:2px;
    class A current;
    class B doc;
    class C module;
    class D term;
    class E asset;
    class F platform;
```

This is an incremental path, not a requirement to build the final state immediately. The company can begin with templating and modularization, then add structured metadata to core metrics, terms, and tables.

### Recommendation for the Company (Our Assessment)

The most suitable near-term path is **templating → modularization**, without immediately building a full data-catalog platform:

- Maintain reusable, cross-scenario knowledge separately, such as revenue definitions, device-model definitions, Push types, and shared table descriptions.
- Keep AI workflows as standalone modules containing only task boundaries, Steps 1–N, knowledge dependencies, and validation rules.
- Separate SQL from business introductions and reference it as examples or templates.
- First add summaries, tables of contents, and stable sections to very long documents; later extract the most frequently used or frequently updated sections.

Recommended minimum document types:

```text
concept     Business concept or background
metric      Metric and definition
event       Tracking-event definition and background
dataset     Table and field description
sql         SQL example or template
workflow    AI operating procedure
```

This step does not require a vector database. It can continue to use Markdown and the existing Wiki.

---

## 2. Knowledge Discovery and Retrieval

### Representative Industry Approaches

| Approach | Best suited for | Advantages | Limitations | Evidence |
|---|---|---|---|---|
| Directory + grep/full-text search | Table names, fields, tracking events, metric abbreviations, and SQL | Precise, transparent, and inexpensive; particularly effective for code and data-asset names | Can miss content when user wording differs significantly from document terminology | A classic full-text retrieval method; Microsoft also notes that specialized terminology, codes, and names work well with keyword search. [Microsoft documentation](https://learn.microsoft.com/en-us/azure/search/hybrid-search-overview) |
| Metadata filtering | Narrow results by domain, type, status, or owner before searching | Excludes irrelevant or obsolete content; results are explainable | Requires complete document metadata | DataHub supports filtering by domain, owner, type, glossary term, and soft-deletion status. [DataHub documentation](https://docs.datahub.com/docs/how/search) |
| Vector retrieval | Natural-language queries, synonyms, and users who do not know the formal terminology | Finds semantically similar content | Can retrieve definitions that are semantically similar but have different business conditions; less explainable | Microsoft positions vector retrieval as a semantic complement to keyword search. [Microsoft documentation](https://learn.microsoft.com/en-us/azure/search/hybrid-search-overview) |
| Hybrid retrieval + reranking | Queries containing both specialized terms and natural-language descriptions | Combines exact matching with semantic recall; reranking reduces noise | Adds indexing, tuning, latency, and evaluation costs | Azure AI Search runs full-text and vector search in parallel and merges them with RRF; Alibaba PAI-RAG exposes hybrid mode, weights, thresholds, and reranking. [Microsoft documentation](https://learn.microsoft.com/en-us/azure/search/hybrid-search-overview) [Alibaba Cloud documentation](https://www.alibabacloud.com/help/en/pai/use-cases/rag-api-interface-for-v0-4-x) |
| Contextual Retrieval | Chunks that lose their business, time, or object context after document splitting | Adds document-level context to each chunk, making it more independently retrievable | Requires additional offline model calls and still depends on source quality | Anthropic's Contextual Retrieval adds chunk-level context to both embeddings and BM25; its internal experiments report further improvements with reranking. [Anthropic engineering article](https://www.anthropic.com/engineering/contextual-retrieval) |
| Agentic Retrieval | Complex questions requiring decomposition and dynamic knowledge-source selection | Supports multiple search rounds and evidence-sufficiency checks | Higher cost, latency, and unpredictability | Microsoft describes it as a modern RAG pattern in which a model decomposes queries, runs parallel searches, and returns structured grounding data. [Microsoft documentation](https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/retrieval-augmented-generation) |

### Recommendation for the Company (Our Assessment)

The company should not jump directly from grep to full vectorization. A more appropriate progression is:

```text
Three-level directory
→ Metadata filtering
→ grep/keyword document discovery
→ Section retrieval using headings and anchors
→ Add vector retrieval for failures caused by terminology mismatch
→ Consider hybrid retrieval/reranking only after measurable evidence
```

A three-level navigation structure could be:

```text
Level 1: Business domain     Push / Revenue / Fulfillment / User / Merchant / ...
Level 2: Knowledge type      concept / metric / event / dataset / sql / workflow
Level 3: Knowledge module    metric.revenue / dataset.push_event / workflow.revenue_by_device
```

Hundreds of documents do not need to be inserted into every system prompt. The directory can be generated automatically from Markdown metadata: show only Level 1 globally, then expose Level 2 and Level 3 entries after a business domain is selected.

The first retrieval question to validate is whether AI can find the correct section, not merely the correct file. If structured directories, headings, and anchors solve most failures, vector retrieval can remain a later priority.

---

## 3. Knowledge Governance and Lifecycle

### Representative Industry Approaches

| Approach | Problem addressed | Advantages | Limitations | Evidence |
|---|---|---|---|---|
| Owner/Steward | Defines who is responsible for explanation and updates | Clear accountability; conflicts can be routed to the right person | Ownership must be updated when people change roles or leave | Google business glossaries support contacts and administrator/editor/viewer roles; DataHub treats ownership as core asset information. [Google documentation](https://cloud.google.com/dataplex/docs/create-glossary) [DataHub documentation](https://docs.datahub.com/docs/api/tutorials/owners) |
| Status and deprecation markers | Distinguishes drafts, active knowledge, history, and deprecated knowledge | AI can reduce the priority of obsolete documents | Someone must update status promptly | DataHub Agent Context uses recency, deprecation, and published/draft state as trust and ranking signals. [DataHub documentation](https://docs.datahub.com/docs/managed-datahub/build-a-data-agent/key-concepts) |
| Change notifications and dependency impact | Notifies downstream workflows when a shared definition changes | Prevents silent use of obsolete dependencies | Dependency relationships must be maintained | DataHub supports subscriptions to deprecation, schema, ownership, and glossary-term changes. [DataHub documentation](https://docs.datahub.com/docs/managed-datahub/subscription-and-notification) |
| Owner review | Triggers responsible-owner review when authoritative knowledge changes | Appropriate for metrics, core tables, and shared tracking events | Mandatory review for everything can slow down ordinary knowledge capture | GitHub CODEOWNERS assigns responsible people by directory/file and can require their approval. [GitHub documentation](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners) |
| Permission-aware index filtering | Ensures users retrieve only content they are authorized to see | Prevents the model from seeing sensitive data before attempting not to reveal it | Requires synchronization with source-system ACLs | Azure AI Search supports storing user/group IDs in the index and filtering at query time. [Microsoft documentation](https://learn.microsoft.com/en-us/azure/search/search-security-trimming-for-azure-search) |

### Recommendation for the Company (Our Assessment)

Start with minimum governance rather than a complex approval system. Every module used by AI should contain at least:

```yaml
id: metric.revenue
type: metric
domain: revenue
summary: Definition, scope, and calculation rules for the revenue metric
owner: revenue-analytics
status: active          # draft / active / deprecated
authority: canonical   # canonical / reference / example
updated_at: 2026-10-01
source: wiki-url
depends_on: []
supersedes: []
```

Based on current systems, an initial division of authoritative sources could be:

| Content | Preferred source of truth |
|---|---|
| Technical tracking-event definition | Tracking platform |
| Tracking-event design background | Corresponding requirement/design document |
| Actual table calculation and dependencies | GitHub table-building code |
| Business meaning and usage constraints of a table | Owner-assigned dataset document |
| Metric definition | `canonical` metric document |
| Analysis procedure | Workflow document |

If AI finds two conflicting documents that are both `active + canonical`, it should report the conflict and request confirmation rather than synthesizing its own answer. Existing long documents can remain available but should be marked `reference` or `deprecated` so they do not compete with new modules at equal authority.

---

## 4. Knowledge Use and Context Assembly

### Representative Industry Approaches

| Approach | Loading model | Advantages | Limitations | Evidence |
|---|---|---|---|---|
| Load the full document | Insert the complete document into context after a match | Simplest implementation | Long documents dilute important information, increase token use, and cause the model to use the wrong section even after finding the right document | Included as the traditional baseline |
| Three-level progressive loading | Persistent metadata → main instructions after activation → references/scripts on demand | Only relevant content consumes context; scales to hundreds of modules | Depends heavily on good names, descriptions, and reference paths | Anthropic Skills explicitly use three-level Progressive Disclosure: metadata is always visible, instructions load after activation, and resources load only as needed. [Anthropic documentation](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview) |
| Retrieval as a tool | AI decides when to search, which source to use, and whether to continue | Suitable for complex or incomplete-information tasks | Calls must be constrained and logged | Microsoft Agent Framework supports both pre-invocation context injection and search exposed as an on-demand tool. [Microsoft documentation](https://learn.microsoft.com/en-us/agent-framework/user-guide/agents/agent-rag) |
| Recursive/router retrieval | Find a parent node or directory first, then enter a child knowledge source | Supports hierarchical directories, multiple domains, and dependency relationships | Routing errors can cause missed results; the chain is harder to debug | LlamaIndex provides Router, Recursive, and Composable Retrievers; its repository had about 8.1k forks in the page snapshot. [LlamaIndex GitHub documentation](https://github.com/run-llama/llama_index/blob/main/docs/src/content/docs/framework/module_guides/querying/retriever/retrievers.md) |
| Deep document parsing and citations | Preserve paragraph, table, and source positions and return citations with answers | Improves verification and works well for complex documents | The current Wiki is mostly plain Markdown, so near-term value is lower than structural governance | RAGFlow emphasizes complex-document parsing, templated chunking, and citations; it had about 92.4k GitHub stars at the research snapshot. [RAGFlow GitHub](https://github.com/infiniflow/ragflow) |

### Recommendation for the Company (Our Assessment)

Use a three-level loading pattern similar to Anthropic Skills, adapted for internal knowledge:

```text
Level 1: Business-domain directory
Always visible; contains only domain names, summaries, and navigation instructions.

Level 2: Module metadata or workflow entry document
Loaded after a task matches; contains the summary, steps, dependency IDs,
validation rules, and section directory.

Level 3: Detailed knowledge content
Loaded only when needed: metric definitions, table descriptions,
tracking-event background, SQL examples, or specific sections.
```

The minimum knowledge-reading capabilities exposed to AI could be:

```text
list_domains()
list_knowledge(domain, type)
get_metadata(knowledge_id)
read_section(knowledge_id, heading)
get_dependencies(knowledge_id)
search_text(query, filters)
```

The first phase does not require a formal service. Directories and dependencies can come from Markdown frontmatter, while Claude Code continues to use glob/grep/read. The important change is to make AI follow a reading protocol: directory first, then summary, then relevant sections, and the full document only when necessary.

---

## 5. Incremental Roadmap for the Company

| Phase | Main change | What remains | What to validate |
|---|---|---|---|
| A. Structure existing documents | Add summaries, tables of contents, stable headings, and minimum metadata to frequently used long documents | Continue using the Wiki and grep/glob | Whether “right file, wrong content” failures decrease |
| B. Extract shared modules | Separate metric, dataset, event, and SQL modules; workflows reference dependencies | Retain old documents as `reference` | Whether duplication decreases and updates become more centralized |
| C. Three-level navigation and progressive loading | Automatically generate domain–type–module directories; read by section | Continue using Markdown | Whether context length and location accuracy improve |
| D. Minimum governance | Add owner, status, authority, update time, and supersession relationships | Do not require all documents to be completed at once | Whether conflicts and obsolete documents decrease |
| E. Retrieval enhancement | Test vector, hybrid retrieval, and reranking only on keyword/directory failure cases | Preserve exact retrieval | Whether new retrieval methods create measurable incremental value |

The recommended target for the current stage is a **modular Markdown knowledge base + three-level navigation + progressive loading + minimum governance**. This adopts the on-demand loading model of Anthropic Skills and the structured terminology and trust signals of Google/DataHub, while retaining the existing Wiki and grep/glob workflow. Vector retrieval should be treated as a later, measurable enhancement rather than the starting point of the knowledge-base transformation.

## References

1. Anthropic Agent Skills: https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview
2. Anthropic Contextual Retrieval: https://www.anthropic.com/engineering/contextual-retrieval
3. Microsoft Azure Hybrid Search: https://learn.microsoft.com/en-us/azure/search/hybrid-search-overview
4. Microsoft Foundry RAG / Agentic Retrieval: https://learn.microsoft.com/en-us/azure/ai-foundry/concepts/retrieval-augmented-generation
5. Microsoft Azure document-level security filtering: https://learn.microsoft.com/en-us/azure/search/search-security-trimming-for-azure-search
6. Alibaba Cloud PAI-RAG API: https://www.alibabacloud.com/help/en/pai/use-cases/rag-api-interface-for-v0-4-x
7. Google Dataplex Business Glossary: https://cloud.google.com/dataplex/docs/create-glossary
8. Google Knowledge Catalog: https://docs.cloud.google.com/knowledge-catalog/docs
9. DataHub: https://github.com/datahub-project/datahub
10. GitHub CODEOWNERS: https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/about-code-owners
11. RAGFlow: https://github.com/infiniflow/ragflow
12. LlamaIndex Retrievers: https://github.com/run-llama/llama_index/blob/main/docs/src/content/docs/framework/module_guides/querying/retriever/retrievers.md

> Note: The company/project capabilities in the tables are based on the linked official pages or GitHub repositories. The company-specific recommendations and migration sequence are our assessment based on the current situation, not official recommendations from those vendors. GitHub stars and forks change continuously; the figures shown are snapshots from October 18, 2026.
