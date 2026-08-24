Build an Enterprise-Grade Code Knowledge Graph + GraphRAG System
1. Objective
Build a modular, production-oriented prototype of a Code Intelligence / Code Knowledge Graph system.
The system should ingest a source-code repository, analyze the code, construct a knowledge graph representing the structure and relationships within the codebase, create semantic embeddings for relevant code chunks, and allow users to ask natural-language questions about the codebase.
The system must combine:
1. Deterministic static code analysis
2. A structural knowledge graph
3. Semantic/vector retrieval
4. Graph traversal
5. LLM-based reasoning and answer generation
6. Source-code provenance/citations
7. Incremental repository indexing
8. A clean abstraction layer so the current in-memory databases can later be replaced by production databases
The initial implementation should support Python only.
The architecture must be designed so JavaScript and Java can be added later without rewriting the core system.

2. Critical Architectural Requirement
The initial implementation must NOT depend directly on Neo4j, Qdrant, Pinecone, PostgreSQL, or any other production database.
Use:
* Python
* In-memory graph store
* In-memory vector store
* In-memory metadata/repository store where appropriate
However, the application must use clean interfaces/abstractions so that the storage implementations can later be replaced.
For example:
class GraphStore(ABC):
    ...

class VectorStore(ABC):
    ...

class RepositoryStore(ABC):
    ...
Initial implementations:
class InMemoryGraphStore(GraphStore):
    ...

class InMemoryVectorStore(VectorStore):
    ...

class InMemoryRepositoryStore(RepositoryStore):
    ...
Later implementations should be possible:
class Neo4jGraphStore(GraphStore):
    ...

class QdrantVectorStore(VectorStore):
    ...

class PostgresRepositoryStore(RepositoryStore):
    ...
The rest of the application must depend only on the interfaces, never on the concrete storage implementations.
This requirement is extremely important.

3. High-Level Architecture
Implement the following architecture:
                         ┌──────────────────────┐
                         │      User / UI       │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │       FastAPI        │
                         │      API Layer       │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │     Query Engine     │
                         │                      │
                         │ Query classification│
                         │ Retrieval            │
                         │ Graph traversal      │
                         │ Context construction │
                         └──────────┬───────────┘
                                    │
                    ┌───────────────┼────────────────┐
                    │               │                │
                    ▼               ▼                ▼
             Graph Retrieval   Vector Retrieval   Symbol Search
                    │               │                │
                    └───────────────┼────────────────┘
                                    ▼
                         ┌──────────────────────┐
                         │    Evidence Builder  │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         ┌──────────────────────┐
                         │         LLM          │
                         │ Answer generation    │
                         └──────────┬───────────┘
                                    │
                                    ▼
                         Answer + source citations
Repository ingestion:
Git Repository
      │
      ▼
Repository Scanner
      │
      ▼
Python Parser / Tree-sitter
      │
      ▼
Language-Neutral Intermediate Representation
      │
      ├───────────────┐
      ▼               ▼
Graph Builder    Chunk Builder
      │               │
      ▼               ▼
GraphStore       Embedding Model
      │               │
      ▼               ▼
InMemoryGraph    InMemoryVectorStore

4. Technology Requirements
Use Python as the implementation language.
Recommended stack:
Python 3.11+
FastAPI
Pydantic
Tree-sitter
tree-sitter-python
GitPython
pytest
httpx
numpy
For embeddings, create an abstraction:
class EmbeddingProvider(ABC):
    def embed(self, text: str) -> list[float]:
        ...
Provide at least:
class OpenAIEmbeddingProvider(EmbeddingProvider):
    ...
Also provide:
class MockEmbeddingProvider(EmbeddingProvider):
    ...
The mock provider should allow the entire system to run without API keys.
For the LLM, create:
class LLMProvider(ABC):
    def generate(self, prompt: str, ...) -> str:
        ...
Provide:
class OpenAILLMProvider(LLMProvider):
    ...

class MockLLMProvider(LLMProvider):
    ...
The system must be runnable entirely offline using the mock providers.

5. Repository Ingestion
Create a repository ingestion pipeline.
The pipeline should accept either:
1. A local repository path
2. A Git repository URL
Example:
POST /repositories/index
Request:
{
  "repository_path": "/path/to/repository"
}
Eventually support:
{
  "repository_url": "https://github.com/example/project.git"
}
For the initial implementation, local repositories are sufficient.

6. Repository Scanner
Implement a repository scanner that:
* Recursively scans files
* Detects programming languages
* Ignores irrelevant directories
* Ignores binary files
* Ignores .git
* Supports configurable ignore patterns
Default ignored directories:
.git
node_modules
venv
.venv
__pycache__
dist
build
target
.idea
.vscode
Create:
RepositoryScanner
with:
scan(repository_path) -> list[SourceFile]
SourceFile should contain:
class SourceFile(BaseModel):
    path: str
    language: str
    content: str
    hash: str
    size: int

7. Python Parsing
Use Tree-sitter for Python parsing.
Do NOT build the Python parser entirely using regular expressions.
Create:
PythonCodeParser
The parser should extract:
* Modules
* Classes
* Functions
* Methods
* Imports
* Variables where practical
* Function calls
* Class inheritance
* Decorators
* Function parameters
* Return annotations
* Type annotations
* Docstrings
* Comments where useful
The parser should produce a language-neutral representation.

8. Language-Neutral Intermediate Representation
Create an intermediate representation independent of Python.
Example:
class Symbol:
    id: str
    repository_id: str
    file_path: str
    name: str
    qualified_name: str
    symbol_type: SymbolType
    language: str
    start_line: int
    end_line: int
    signature: str | None
    source_code: str
    docstring: str | None
Symbol types:
class SymbolType(str, Enum):
    FILE = "file"
    MODULE = "module"
    CLASS = "class"
    FUNCTION = "function"
    METHOD = "method"
    VARIABLE = "variable"
Relationships:
class RelationshipType(str, Enum):
    CONTAINS = "contains"
    DEFINES = "defines"
    IMPORTS = "imports"
    CALLS = "calls"
    EXTENDS = "extends"
    IMPLEMENTS = "implements"
    USES = "uses"
    REFERENCES = "references"
    OVERRIDES = "overrides"
    TESTS = "tests"
Represent relationships as:
class Relationship:
    source_id: str
    relationship_type: RelationshipType
    target_id: str
    metadata: dict

9. Stable IDs
Every graph entity must have a deterministic stable ID.
Do NOT use random UUIDs for symbols that should remain stable across re-indexing.
For example:
repository_id:src/payment/service.py:class:PaymentService
and:
repository_id:src/payment/service.py:method:PaymentService.process_payment
Use stable IDs wherever possible.
This is required for incremental indexing.

10. Graph Schema
Create a graph containing at least:
Nodes
Repository
File
Module
Class
Function
Method
Variable
Test
CodeChunk
Relationships
Repository -[CONTAINS]-> File

File -[DEFINES]-> Class

File -[DEFINES]-> Function

Class -[DEFINES]-> Method

File -[IMPORTS]-> Module

Function -[CALLS]-> Function

Method -[CALLS]-> Method

Class -[EXTENDS]-> Class

Function -[USES]-> Variable

Test -[TESTS]-> Function

CodeChunk -[REPRESENTS]-> Function

CodeChunk -[REPRESENTS]-> Method

CodeChunk -[REPRESENTS]-> Class
Keep the schema extensible.

11. In-Memory Graph Database
Implement an in-memory graph database abstraction.
class GraphStore(ABC):

    def add_node(self, node: GraphNode) -> None:
        ...

    def add_relationship(self, relationship: Relationship) -> None:
        ...

    def get_node(self, node_id: str) -> GraphNode | None:
        ...

    def get_neighbors(
        self,
        node_id: str,
        relationship_type: str | None = None,
        direction: str = "both"
    ) -> list[GraphNode]:
        ...

    def find_nodes(
        self,
        node_type: str,
        properties: dict
    ) -> list[GraphNode]:
        ...

    def remove_node(self, node_id: str) -> None:
        ...

    def remove_relationship(...):
        ...

    def clear_repository(self, repository_id: str):
        ...
Implement:
InMemoryGraphStore
Use efficient Python data structures.
For example:
nodes: dict[str, GraphNode]
outgoing: dict[str, list[Relationship]]
incoming: dict[str, list[Relationship]]
Do not implement graph traversal by scanning every node.

12. In-Memory Vector Database
Implement:
class VectorStore(ABC):

    def upsert(self, item: VectorItem) -> None:
        ...

    def search(
        self,
        query_vector: list[float],
        top_k: int = 10,
        filters: dict | None = None
    ) -> list[VectorSearchResult]:
        ...

    def delete(self, item_id: str) -> None:
        ...

    def clear_repository(self, repository_id: str):
        ...
Implement:
InMemoryVectorStore
Initially use cosine similarity.
Use NumPy for efficient vector operations.
Do not use a production vector database yet.

13. Code Chunking
Do not blindly split code into arbitrary token chunks.
Prefer semantic chunks.
A chunk should normally correspond to:
Function
Method
Class
Module-level code
Important configuration block
Example:
class CodeChunk:
    id: str
    repository_id: str
    file_path: str
    symbol_id: str | None
    start_line: int
    end_line: int
    content: str
    language: str
    metadata: dict
For large functions/classes, split them into smaller chunks while preserving:
* Symbol name
* File path
* Parent class
* Start/end lines
* Function signature

14. Embedding Pipeline
For every meaningful code chunk:
CodeChunk
    ↓
EmbeddingProvider
    ↓
vector
    ↓
VectorStore
Embedding text should include contextual metadata.
For example:
Repository: ecommerce
File: src/payment/service.py
Class: PaymentService
Function: process_payment
Language: Python

Code:
...
This is better than embedding raw source code alone.

15. Graph + Vector Relationship
Every CodeChunk should be connected to the symbol it represents.
Example:
CodeChunk
     │
     └── REPRESENTS → PaymentService.process_payment
The vector store should also store:
chunk_id
repository_id
file_path
symbol_id
start_line
end_line
This allows:
Vector search
      ↓
chunk_id
      ↓
symbol_id
      ↓
graph traversal
This connection is fundamental to the retrieval system.

16. Query Engine
Implement:
QueryEngine
It should support three retrieval modes.
Mode 1: Structural
For queries such as:
What calls PaymentService?

Which classes extend BaseRepository?

Where is UserService defined?

Which functions are in payment.py?
Use graph retrieval.

Mode 2: Semantic
For queries such as:
Where is authentication handled?

How are payments processed?

Where do we validate JWT tokens?
Use vector retrieval.

Mode 3: Hybrid
For queries such as:
What happens when a user places an order?

Explain the payment flow.

What components are affected if PaymentService changes?
Use:
Vector search
+
Graph traversal

17. Query Classification
Create:
QueryClassifier
It should classify a query into:
class QueryType(str, Enum):
    STRUCTURAL = "structural"
    SEMANTIC = "semantic"
    HYBRID = "hybrid"
Initially implement deterministic heuristics where practical.
Examples:
"what calls X?"
→ structural

"where is X defined?"
→ structural

"how does authentication work?"
→ semantic/hybrid

"what happens when a user places an order?"
→ hybrid
Eventually allow an LLM to perform classification.
Do not make the LLM mandatory for the initial version.

18. Safe Graph Query Operations
Do NOT initially allow the LLM to execute arbitrary graph queries.
Create a controlled graph-query API:
find_symbol(name)

find_definition(symbol_id)

find_callers(symbol_id)

find_callees(symbol_id)

find_implementations(symbol_id)

find_subclasses(symbol_id)

find_usages(symbol_id)

find_tests(symbol_id)

find_dependencies(symbol_id)

find_dependents(symbol_id)

trace_execution(symbol_id, depth)

find_file_symbols(file_path)
These functions internally use the GraphStore.
This gives us a safe abstraction that can later be translated into Cypher when using Neo4j.

19. Graph Traversal
Implement bounded traversal.
For example:
graph.traverse(
    start_node_id,
    relationship_types=[
        "calls",
        "uses",
        "imports"
    ],
    max_depth=3
)
Never allow unbounded traversal.
Every graph retrieval operation must have:
max_depth
max_nodes
timeout
where applicable.

20. Hybrid Retrieval
Implement:
HybridRetriever
Pipeline:
User Query
    │
    ├───────────────┐
    ▼               ▼
Vector Search   Symbol Search
    │               │
    └───────┬───────┘
            ▼
       Relevant Symbols
            │
            ▼
      Graph Expansion
            │
            ▼
      Relevant Code
            │
            ▼
         Evidence
The hybrid retriever should:
1. Perform vector search
2. Retrieve top-k chunks
3. Resolve chunks to symbols
4. Expand the graph around those symbols
5. Retrieve relevant neighboring symbols
6. Retrieve their source code
7. Deduplicate evidence
8. Rank evidence
9. Return a structured evidence object

21. Evidence Model
Create an explicit evidence model.
class Evidence:
    source_type: str
    repository_id: str
    file_path: str
    symbol_id: str | None
    symbol_name: str | None
    start_line: int
    end_line: int
    content: str
    relevance_score: float
    relationship_path: list[str]
Never send arbitrary retrieved text to the LLM without provenance.

22. Context Builder
Create:
ContextBuilder
It should transform evidence into structured LLM context.
Example:
Repository: ecommerce

Evidence 1:
File: src/payment/service.py
Symbol: PaymentService.process_payment
Lines: 42-67

[code]

Evidence 2:
File: src/payment/gateway.py
Symbol: PaymentGateway.charge
Lines: 10-38

[code]

Graph relationships:

PaymentController
    CALLS
PaymentService.process_payment

PaymentService.process_payment
    CALLS
PaymentGateway.charge
The context should be bounded by a configurable token/character budget.

23. LLM Answer Generation
Create:
AnswerGenerator
The system prompt should instruct the LLM:
* Answer only from supplied repository evidence
* Do not invent files, functions, or behavior
* Clearly distinguish evidence from inference
* Include file names
* Include line ranges
* Mention uncertainty when evidence is insufficient
* Never claim a function exists unless supported by evidence
* Prefer concise technical explanations
* Explain graph relationships when relevant
Example answer:
Authentication is primarily handled by AuthService.

The flow appears to be:

LoginController
  → AuthMiddleware
  → AuthService.authenticate()
  → JWTValidator.validate()

Evidence:

1. src/auth/service.py:31-67
2. src/auth/middleware.py:12-48
3. src/auth/jwt.py:5-40

24. Citation System
Every answer must be traceable back to source code.
Create a citation format:
[src/auth/service.py:31-67]
The backend should return structured citations:
class Citation:
    file_path: str
    start_line: int
    end_line: int
    symbol_id: str | None
API response:
{
  "answer": "...",
  "citations": [
    {
      "file_path": "src/auth/service.py",
      "start_line": 31,
      "end_line": 67,
      "symbol_id": "..."
    }
  ]
}

25. FastAPI API
Create these initial endpoints.
Repository
POST /repositories/index

GET /repositories

GET /repositories/{repository_id}

DELETE /repositories/{repository_id}
Query
POST /query
Example:
{
  "repository_id": "repo-123",
  "query": "How does authentication work?"
}
Response:
{
  "answer": "...",
  "query_type": "hybrid",
  "citations": [],
  "evidence": []
}
Graph
GET /repositories/{repository_id}/graph
GET /repositories/{repository_id}/symbols/{symbol_id}
GET /repositories/{repository_id}/symbols/{symbol_id}/callers
GET /repositories/{repository_id}/symbols/{symbol_id}/callees

26. Ingestion Pipeline
Create:
RepositoryIndexer
Pipeline:
Repository
    ↓
Scan
    ↓
Detect changed files
    ↓
Parse
    ↓
Extract symbols
    ↓
Resolve relationships
    ↓
Update graph
    ↓
Create code chunks
    ↓
Generate embeddings
    ↓
Update vector store
    ↓
Mark indexing complete
Return an indexing result:
class IndexResult:
    repository_id: str
    files_processed: int
    files_added: int
    files_modified: int
    files_deleted: int
    symbols_created: int
    relationships_created: int
    chunks_created: int
    duration_seconds: float

27. Incremental Indexing
This is mandatory.
Do not rebuild the entire repository if only one file changes.
Compute a hash for every source file.
Maintain:
file_path
content_hash
last_indexed_hash
On subsequent indexing:
same hash
→ skip

different hash
→ reparse

file disappeared
→ remove corresponding graph nodes/chunks/vectors
When a file changes:
1. Remove old symbols belonging to the file
2. Remove old relationships originating from those symbols
3. Remove old code chunks
4. Remove old vectors
5. Parse the new file
6. Create new symbols
7. Resolve relationships
8. Create new chunks
9. Generate new embeddings

28. Dependency Resolution
For Python initially, attempt to resolve:
from foo.bar import Baz

import foo
to actual repository symbols when possible.
Do not require perfect static resolution.
If resolution is uncertain, preserve the relationship as unresolved metadata instead of inventing a target.
For example:
PaymentService
   CALLS
unresolved: gateway.charge
Never fabricate relationships.

29. Python-Specific Analysis
Support:
Imports
import requests
from services.payment import PaymentService
Classes
class PaymentService(BaseService):
Functions
def process_payment(payment):
Methods
class Payment:
    def validate(self):
Calls
service.process_payment()
Decorators
@app.post("/payment")
Type annotations
def process(payment: Payment) -> Transaction:
Docstrings
Extract them as metadata.

30. API Endpoint Extraction
Detect common Python web frameworks where practical.
Initially support FastAPI patterns such as:
@app.get("/users")
@app.post("/users")
@router.get("/users/{id}")
Create:
Endpoint
nodes.
Example:
Endpoint("/users")
      │
      └── HANDLED_BY
              ↓
      UserController.get_users
This should make queries like:
"What happens when GET /users is called?"
possible.

31. Future Language Support
Do NOT mix Python-specific logic throughout the application.
Define:
class LanguageParser(ABC):

    def parse(
        self,
        source_file: SourceFile
    ) -> ParsedFile:
        ...
Implement:
PythonParser(LanguageParser)
Later:
JavaScriptParser(LanguageParser)
JavaParser(LanguageParser)
The rest of the system must work with:
ParsedFile
Symbol
Relationship
CodeChunk
and not with Python AST objects.

32. Project Structure
Use a clean project structure such as:
code-intelligence/
│
├── app/
│   ├── main.py
│   │
│   ├── api/
│   │   ├── repositories.py
│   │   ├── query.py
│   │   └── graph.py
│   │
│   ├── domain/
│   │   ├── models.py
│   │   ├── enums.py
│   │   └── interfaces.py
│   │
│   ├── ingestion/
│   │   ├── scanner.py
│   │   ├── indexer.py
│   │   └── incremental.py
│   │
│   ├── parsing/
│   │   ├── base.py
│   │   ├── python_parser.py
│   │   └── registry.py
│   │
│   ├── graph/
│   │   ├── store.py
│   │   └── memory_store.py
│   │
│   ├── vector/
│   │   ├── store.py
│   │   └── memory_store.py
│   │
│   ├── embeddings/
│   │   ├── base.py
│   │   ├── openai.py
│   │   └── mock.py
│   │
│   ├── llm/
│   │   ├── base.py
│   │   ├── openai.py
│   │   └── mock.py
│   │
│   ├── retrieval/
│   │   ├── graph.py
│   │   ├── vector.py
│   │   ├── hybrid.py
│   │   └── ranking.py
│   │
│   ├── query/
│   │   ├── classifier.py
│   │   ├── engine.py
│   │   └── operations.py
│   │
│   └── services/
│       ├── repository_service.py
│       └── query_service.py
│
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
│
├── examples/
│
├── scripts/
│
├── requirements.txt
├── pyproject.toml
├── .env.example
├── README.md
└── docker-compose.yml

33. Dependency Injection
Do not instantiate storage providers directly throughout the application.
Bad:
class QueryService:
    def __init__(self):
        self.graph = InMemoryGraphStore()
Better:
class QueryService:

    def __init__(
        self,
        graph_store: GraphStore,
        vector_store: VectorStore,
        llm: LLMProvider,
        embedding_provider: EmbeddingProvider,
    ):
        self.graph_store = graph_store
        self.vector_store = vector_store
        self.llm = llm
        self.embedding_provider = embedding_provider
Use FastAPI dependency injection to provide implementations.

34. Configuration
Use environment variables.
Example:
LLM_PROVIDER=mock
EMBEDDING_PROVIDER=mock
GRAPH_STORE=memory
VECTOR_STORE=memory
Later:
LLM_PROVIDER=openai
EMBEDDING_PROVIDER=openai
GRAPH_STORE=neo4j
VECTOR_STORE=qdrant
The application code should not need to change.

35. Security Requirements
Because this is intended to become an enterprise application:
Do not log source code unnecessarily.
Do not include API keys in source code.
Use environment variables/secrets.
Every repository must have an isolated repository ID.
Every graph and vector operation must accept a repository scope.
Never perform cross-repository retrieval.
Never send unrelated repositories' source code to the LLM.
Design the interfaces so tenant isolation can later be added.

36. Observability
Add structured logging.
Every query should have:
request_id
repository_id
query
query_type
vector_search_latency
graph_search_latency
llm_latency
number_of_chunks
number_of_graph_nodes
number_of_tokens
Do not log raw source code by default.

37. Error Handling
Handle:
* Invalid repository
* Unsupported language
* Parser failure
* Embedding failure
* LLM failure
* Empty repository
* Empty search results
* Invalid symbol
* Graph traversal limit
* Embedding dimension mismatch
The system should degrade gracefully.
If the LLM is unavailable, return retrieved evidence rather than crashing.


41. CLI
Provide a CLI.
Example:
python -m app.cli index ./my-repository
and:
python -m app.cli query ./my-repository \
    "How does authentication work?"
Also support:
python -m app.cli graph ./my-repository
to print basic graph statistics.

42. Graph Statistics
Provide:
Repositories: 1
Files: 42
Classes: 73
Functions: 184
Methods: 291
Relationships: 912
Code chunks: 486
Expose this through:
GET /repositories/{repository_id}/stats

43. Retrieval Ranking
Implement a basic ranking strategy.
For hybrid retrieval, combine:
semantic_similarity
+
symbol relevance
+
graph distance
+
relationship relevance
For example:
final_score =
    0.50 * semantic_score
  + 0.25 * symbol_score
  + 0.15 * graph_score
  + 0.10 * relationship_score
Make the weights configurable.
Do not hard-code them throughout the application.

44. Avoid Hallucinated Relationships
This is a hard requirement.
If the parser cannot determine whether:
A → CALLS → B
then do not create the relationship as fact.
Use:
unresolved_reference
or omit the relationship.
The graph must represent known facts.
LLM-derived semantic relationships must be clearly distinguished from deterministic relationships.

45. Deterministic vs Semantic Graph Information
Support metadata indicating how a relationship was generated.
Example:
relationship.metadata = {
    "source": "tree_sitter",
    "confidence": 1.0
}
For LLM-derived information:
relationship.metadata = {
    "source": "llm",
    "confidence": 0.82
}
Never treat an LLM inference as equivalent to static-analysis evidence.

46. Repository Versioning
Associate indexed data with:
repository_id
commit_hash
branch
Example:
Repository
    id: repo-123

Index
    repository_id: repo-123
    commit: abc123
    branch: main
This should make future version-aware queries possible.
For now, indexing the current working tree is sufficient, but design the models for commit/version support.

47. Future Database Migration
The following migration should require implementing adapters, not rewriting the application.
Current:
GraphStore
    └── InMemoryGraphStore

VectorStore
    └── InMemoryVectorStore
Future:
GraphStore
    ├── InMemoryGraphStore
    └── Neo4jGraphStore

VectorStore
    ├── InMemoryVectorStore
    ├── QdrantVectorStore
    └── PgVectorStore
Do not expose database-specific objects outside their adapter.
For example, do not allow Neo4j Node objects to leak into domain code.
Return domain objects such as:
GraphNode
Relationship
VectorSearchResult
instead.

48. Important Design Principle
The system should have these layers:
                    API
                     │
                     ▼
                  Services
                     │
                     ▼
                  Domain
                     │
          ┌──────────┴──────────┐
          ▼                     ▼
       Retrieval            Ingestion
          │                     │
          ▼                     ▼
       Interfaces           Interfaces
          │                     │
          ▼                     ▼
     Storage adapters      Parser adapters
The domain must not know whether the underlying database is:
in-memory
Neo4j
PostgreSQL
Qdrant
Pinecone

49. Initial Scope Restrictions
Do NOT initially implement:
* Java parsing
* JavaScript parsing
* Kubernetes deployment
* distributed indexing
* Neo4j
* Qdrant
* PostgreSQL
* complex authentication
* arbitrary LLM-generated Cypher
* autonomous code modification
* code execution
* sandbox execution
Focus on making the Python implementation excellent.
Architecture should support those features later.

50. Expected End-to-End Flow
After implementation, the following should work:
1. User provides a Python repository.

2. System scans the repository.

3. Tree-sitter parses Python files.

4. Symbols are extracted.

5. Relationships are extracted.

6. Graph is constructed in InMemoryGraphStore.

7. Semantic code chunks are created.

8. Embeddings are generated.

9. Embeddings are stored in InMemoryVectorStore.

10. User asks:

    "How does authentication work?"

11. QueryClassifier determines:

    HYBRID

12. Vector search retrieves relevant authentication chunks.

13. Chunks resolve to graph symbols.

14. Graph traversal discovers related components.

15. EvidenceBuilder creates grounded evidence.

16. LLM receives:

    - question
    - relevant source code
    - graph relationships
    - file paths
    - line numbers

17. LLM produces an answer.

18. Answer contains source references.

Example:

    Authentication starts at
    src/api/auth_controller.py:20-41.

    The controller delegates to
    src/services/auth_service.py:31-72.

    JWT validation is performed by
    src/security/jwt.py:10-38.

19. API returns structured answer + citations.

51. Deliverables
Implement the complete working project.
Provide:
1. Full source code
2. README.md
3. requirements.txt or pyproject.toml
4. .env.example
5. FastAPI API
6. In-memory graph database
7. In-memory vector database
8. Tree-sitter Python parser
9. Repository scanner
10. Incremental indexing
11. Semantic chunking
12. Embedding abstraction
13. LLM abstraction
14. Structural retrieval
15. Vector retrieval
16. Hybrid retrieval
17. Query classification
18. Evidence builder
19. Source citations
20. Unit tests
21. Integration tests
22. Example test repository
23. Evaluation questions
24. Architecture documentation


54. Final Architectural Goal
The final architecture should conceptually be:
                    CODE REPOSITORY
                           │
                           ▼
                    ┌──────────────┐
                    │ Code Scanner │
                    └──────┬───────┘
                           │
                           ▼
                    ┌──────────────┐
                    │ Tree-sitter  │
                    └──────┬───────┘
                           │
                           ▼
                 Language-neutral IR
                           │
                 ┌─────────┴─────────┐
                 ▼                   ▼
          Structural Graph       Code Chunks
                 │                   │
                 ▼                   ▼
        InMemoryGraphStore    EmbeddingProvider
                                     │
                                     ▼
                              InMemoryVectorStore
                                     │
                                     │
                                     ▼
                              ┌─────────────┐
                              │ Query Engine│
                              └──────┬──────┘
                                     │
                         ┌───────────┼───────────┐
                         ▼           ▼           ▼
                       Graph       Vector      Hybrid
                       Search      Search      Search
                         │           │           │
                         └───────────┼───────────┘
                                     ▼
                              Evidence Builder
                                     │
                                     ▼
                                    LLM
                                     │
                                     ▼
                          Grounded Answer
                                     │
                                     ▼
                          Source Citations
The most important architectural property is:
                     APPLICATION
                          │
                          ▼
                     INTERFACES
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
       In-memory today          Production later
              │                       │
              ▼                       ▼
     InMemoryGraphStore        Neo4jGraphStore
     InMemoryVectorStore      QdrantVectorStore
The application should not care which implementation is underneath.
Build the initial version around this principle.
