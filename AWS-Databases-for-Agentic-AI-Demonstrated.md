# Building Database-Connected Agentic Applications on AWS

An independently written, practical guide to architectural patterns for
AI applications that work with structured data, document retrieval,
persistent state, and controlled developer diagnostics.

> **Scope and originality:** This is general educational material. It
> does not reproduce assessment questions, answers, diagrams,
> screenshots, challenge-specific resource identifiers, configuration
> keys, or assessment instructions. Adapt the examples to your own
> environment and consult current AWS documentation.

## 1. What we are building

Consider an AI assistant for an online retailer. It should answer
questions about orders, retrieve information from product manuals,
preserve workflow state, and let authorized engineers inspect its data
services.

A robust design separates these responsibilities rather than giving a
model unrestricted access to every data source.

## 2. Reference architecture

![AWS Databases for Agentic AI Architecture](./docs/AWS%20Agentic%20AI%20Architecture%20Diagram.png)



## 3. Structured data access with a relational database

Relational databases are appropriate when an application needs
consistent access to records such as orders, customers, inventory, or
payments.

### Separate application access from administration

Create a dedicated database identity for the application and grant only
the permissions it needs.

A practical process:

1.  Identify the records and fields the assistant requires.
2.  Create a restricted identity for application access.
3.  Expose carefully designed queries or database views.
4.  Keep schema changes and administrative operations outside the
    agent's normal execution path.
5.  Test access using the same identity the application will use.

### Use purpose-built data views

Suppose an application frequently needs a summary of order activity by
product category. A view can centralize the aggregation logic.

Illustrative SQL (all names below are generic examples):

``` sql
CREATE VIEW reporting.category_activity AS
SELECT
    category,
    COUNT(*) AS order_line_count,
    SUM(quantity) AS units_sold,
    MAX(created_at) AS latest_activity
FROM commerce.order_lines
GROUP BY category;
```

Views can provide consistent business definitions, reduce query
complexity, narrow data exposure, and improve predictability. For
high-volume workloads, evaluate whether a normal view, materialized
view, summary table, or analytics store is more appropriate.

### Protect credentials

-   Store database credentials in a managed secret store.
-   Allow each compute role to retrieve only the secret it needs.
-   Rotate credentials according to operational policy.
-   Use encrypted database connections.
-   Never print secret payloads or connection strings in logs.

The secret's structure depends on the database driver and application
implementation.

### Design a safe query layer

Avoid allowing a model to execute arbitrary SQL without controls. A safe
query layer should validate table and field access, prefer parameterized
statements, restrict operations to read-only access where possible,
apply timeouts and result limits, reject unexpected statement types, and
record audit metadata without logging sensitive values.

## 4. Semantic document retrieval

Long-form content such as product manuals, policies, support guides, and
technical documents is often better handled through retrieval-augmented
generation (RAG).

### Ingestion workflow

1.  Store source documents in object storage.
2.  Connect the document location to a managed knowledge-retrieval
    service.
3.  Select an embedding model appropriate for the content and language.
4.  Split documents into useful passages.
5.  Generate embeddings for those passages.
6.  Store vectors and associated metadata in a supported vector store.
7.  Run ingestion and verify that the source material is searchable.

### Vector storage considerations

A PostgreSQL vector extension can support similarity search alongside
relational data.

Consider embedding dimensions, metadata filters, index selection,
document updates, and access restrictions. The vector column dimension
must match the selected embedding model. Document-level authorization
must be enforced by the application; storing an embedding does not
automatically secure the source document.

### Retrieval workflow

1.  The application receives a question.
2.  The retrieval component searches for relevant passages.
3.  Relevant passages are returned with source metadata.
4.  The agent uses those passages as context.
5.  The response distinguishes retrieved evidence from unsupported
    assumptions.

Keep retrieval separate from transactional SQL access. Route structured
questions to the relational data layer and document questions to the
retrieval layer.

## 5. Persistent state for an agent

An agent may need to preserve conversation history, workflow progress,
tool results, or execution checkpoints. A managed NoSQL database can
provide durable storage for this state.

Before creating a table, identify:

-   What uniquely identifies a conversation or workflow.
-   Whether multiple checkpoints exist for one conversation.
-   How records are queried during resume operations.
-   Whether state has a retention period.
-   Which fields are sensitive.

The key design must match the checkpoint library or application data
model. Verify requirements for the exact library version.

A TTL feature can remove records after a retention deadline, but
expiration is asynchronous. The application must write the expected
expiration attribute in the format required by the service.

### Validate persistence

1.  Start a conversation with a stable workflow identifier.
2.  Perform an operation that writes state.
3.  End the invocation.
4.  Start a new invocation using the same identifier.
5.  Confirm that previous state is restored.
6.  Test expiration separately.

A table that exists but cannot restore application state is not a
completed persistence integration.

## 6. Developer diagnostics with Model Context Protocol

The Model Context Protocol (MCP) provides a standard way for compatible
clients to discover and invoke tools.

For database development, an MCP server can expose controlled diagnostic
capabilities. A relational diagnostic service might support schema
inspection and approved read queries; a NoSQL diagnostic service might
expose table metadata and limited sampling.

### Design tools around safe operations

A tool definition should describe its purpose, typed inputs, required
and optional fields, constraints, maximum values, expected output, and
failure behavior.

Keep the tool schema consistent with the server implementation. If the
schema advertises a parameter name or data type that the handler does
not expect, invocation can fail.

### Enforce authorization in the server

Do not rely on the model or MCP client to enforce access controls. The
server should authenticate the caller, authorize access to the requested
resource, validate arguments, restrict access to approved data, apply
query and response limits, and avoid exposing credentials or sensitive
records.

For production systems, separate read-only diagnostics from
administrative actions. Require stronger approval for operations that
change data or schema.

### Keep diagnostic access representative

When engineers debug an agent, diagnostic access should reflect the
permissions the agent actually has. Elevated access can hide permission
problems that the application would encounter. Use separate
administrative access for maintenance tasks.

## 7. Orchestration and tool routing

The agent orchestrator coordinates the model, application tools, and
persistence layer.

  -----------------------------------------------------------------------
  User request                        Appropriate capability
  ----------------------------------- -----------------------------------
  "How many units were sold in a      Structured database query
  category?"

  "What does the installation guide   Document retrieval
  recommend?"

  "Continue the previous support      Persistent agent state
  conversation."

  "Inspect the database structure."   Authorized developer diagnostic
                                      tool
  -----------------------------------------------------------------------

Validate the requested tool, arguments, identity, and data scope before
execution. Useful controls include tool allowlists, strict input
schemas, per-tool timeouts, bounded retries, idempotency, output
redaction, and explicit handling of tool failures.

## 8. Security, reliability, and cost

### Security

-   Apply least privilege to IAM roles and database identities.
-   Keep credentials in managed secret storage.
-   Encrypt data in transit and at rest.
-   Validate authorization before retrieval and query execution.
-   Treat retrieved documents and tool output as untrusted input.
-   Avoid exposing confidential data in prompts and logs.

### Reliability

-   Set timeouts for database calls and model requests.
-   Handle throttling and transient failures.
-   Limit concurrent database connections.
-   Use bounded retries with backoff.
-   Monitor serverless execution duration and error rates.
-   Test state recovery and document re-ingestion.

### Cost and performance

-   Select database capacity modes based on traffic patterns.
-   Monitor query plans and connection usage.
-   Use aggregation or caching for repeated expensive summaries.
-   Tune vector indexes against real retrieval workloads.
-   Limit context passed to the model.
-   Track token usage, tool calls, and serverless invocation costs.

## 9. Troubleshooting patterns

  -----------------------------------------------------------------------
  Symptom                 Possible cause          Investigation
  ----------------------- ----------------------- -----------------------
  Relational connection   Network path,           Inspect connectivity,
  fails                   credentials, TLS, or    secret permissions, and
                          endpoint mismatch       runtime logs

  Query returns           Application identity    Verify database role
  permission errors       lacks access            and grants

  Queries are             Poor query plan or      Inspect query plans,
  unexpectedly slow       repeated aggregation    indexes, and
                                                  aggregation strategy

  Retrieval results are   Stale ingestion,        Review ingestion status
  irrelevant              chunking issues, or     and retrieval
                          embedding mismatch      evaluation

  Agent cannot resume a   Persistence schema or   Inspect checkpoint
  conversation            identifier mismatch     library requirements
                                                  and stored records

  Diagnostic tool         Input schema and        Compare declared types
  invocation fails        handler disagree        with server validation

  Tool exposes too much   Authorization or        Review server-side
  data                    response filtering is   permissions and output
                          incomplete              redaction

  Runtime times out       Long-running operation  Inspect execution
                          or missing timeout      traces and isolate
                                                  blocking calls
  -----------------------------------------------------------------------

## 10. Practical implementation checklist

### Application and identity

-   [ ] Requests are authenticated.
-   [ ] Authorization is enforced at API and tool layers.
-   [ ] The agent has access only to the tools required for its task.

### Structured data

-   [ ] Application database identity is separate from administration.
-   [ ] Credentials are stored outside source code.
-   [ ] Queries are validated and constrained.
-   [ ] Frequently used summaries have an appropriate data-access
    strategy.

### Document retrieval

-   [ ] Source documents are stored in object storage.
-   [ ] Ingestion completes successfully.
-   [ ] Embedding configuration matches the vector schema.
-   [ ] Retrieval quality is tested with representative questions.
-   [ ] Document-level access rules are enforced.

### Persistent state

-   [ ] State schema matches the application or checkpoint library.
-   [ ] The application can write and restore state.
-   [ ] Retention and expiration behavior are tested.

### Developer diagnostics

-   [ ] Tool schemas match handler implementations.
-   [ ] The server validates every argument.
-   [ ] Diagnostic access is authenticated and scoped.
-   [ ] Logs avoid sensitive payloads.

### Operations

-   [ ] Errors, latency, throttling, and cost are monitored.
-   [ ] Timeouts and bounded retries are configured.
-   [ ] Recovery procedures are documented.

## 11. Suggested project layout

``` text
cloud-agent-data-patterns/
├── README.md
├── docs/
│   ├── architecture.md
│   ├── security.md
│   └── troubleshooting.md
├── database/
│   ├── schema-examples/
│   └── query-examples/
├── retrieval/
│   └── ingestion-notes.md
├── agent/
│   └── state-design.md
├── diagnostics/
│   └── tool-design.md
└── examples/
    └── sample-configuration.example
```

Keep account-specific configuration, secret values, customer data,
private keys, and unredacted logs out of the repository. Commit
sanitized examples and explain how readers should supply their own
settings.

## 12. Official documentation

-   [Amazon Aurora
    PostgreSQL](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_AuroraPostgreSQL.html)
-   [Amazon Bedrock Knowledge
    Bases](https://docs.aws.amazon.com/bedrock/latest/userguide/knowledge-base.html)
-   [Amazon DynamoDB Developer
    Guide](https://docs.aws.amazon.com/amazondynamodb/latest/developerguide/Introduction.html)
-   [AWS Lambda Developer
    Guide](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html)
-   [AWS Secrets
    Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/intro.html)
-   [LangGraph
    persistence](https://docs.langchain.com/oss/python/langgraph/persistence)
-   [Model Context
    Protocol](https://modelcontextprotocol.io/introduction)

## 13. Final note

These patterns are starting points, not a substitute for reviewing
current service documentation, library compatibility, security
requirements, and workload-specific performance needs. Build and test in
a dedicated development environment before production deployment.

**Core principle:** Give the agent useful capabilities, but keep data
access, identity, authorization, and operational controls in the
surrounding system.


**Disclaimer:** This is an independently designed educational reference architecture, not a production deployment blueprint or a guarantee of copyright or licensing compliance.