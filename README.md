# Text-to-SQL RAG Agent

**Ask a PostgreSQL database a question in plain English. Get back the SQL and a real answer.**

A LangGraph-based natural-language-to-SQL system with Self-RAG schema retrieval, human-in-the-loop clarification, self-correcting SQL generation and layered security.

## Why this project

Most text-to-SQL demos generate a query and run it with no verification. This project treats the usual failure points as engineering problems:

| Failure mode | Design response |
|---|---|
| Retrieved schema is incomplete | Self-RAG: an LLM critic judges sufficiency and widens retrieval |
| Question is ambiguous | `interrupt()` pauses the graph, asks the user, resumes from a checkpoint |
| SQL fails to execute | Real database error fed back to the generator, bounded retries |
| Untrusted database or destructive SQL | Keyword filter, read-only role, generate-only mode |

## Architecture

```
User question
   |
   v
Retrieve tables (FAISS) --> Critic: sufficient? --no--> widen search (loop)
   | yes
   v
Ambiguous? --yes--> interrupt() --> user answers --> /resume (same thread_id)
   | no
   v
Generate SQL --> Keyword filter --> Read-only execution
                                      | error
                                      v
                           feed error back, retry (bounded)
   | success
   v
Plain-English answer + executed SQL
```

**Layers:** React frontend (Vercel) -> FastAPI (api/, app.py) -> LangGraph pipeline (pipeline/) -> retrieval (rag/) -> PostgreSQL.

## Key design decisions

- **Per-table schema chunks.** A table is the natural unit of meaning for SQL, and keeping columns with their table preserves join context.
- **Two-tier LLM usage.** A fast model handles classification (critique, ambiguity check); SQL generation is a dedicated call.
- **Durable pause/resume.** A PostgreSQL checkpointer keyed by `thread_id` lets a clarification arrive in a separate HTTP request, even hours later.
- **Session-scoped state.** Retrievers and SQLAlchemy engines are cached per connection string, not stored globally, so many databases and users share one code path.
- **LLM output is untrusted.** Security does not depend on model behaviour; the read-only role is the real guarantee and the keyword filter is a first line.

## Security

| Layer | What it stops |
|---|---|
| Keyword filter | DROP, DELETE, UPDATE and similar before reaching the database |
| Read-only Postgres role | Any write, enforced by the database itself |
| Generate-only mode | Execution with unknown privileges on a user-supplied database |

## Tech stack

LangGraph, Groq API (Llama 3.1), FAISS, sentence-transformers, SQLAlchemy, FastAPI, Pydantic, PostgreSQL, React, Vite, Tailwind CSS, Docker, Nginx, systemd, Let's Encrypt, Vercel, AWS EC2.

## Project structure

```
api/         FastAPI routes (/query, /resume)
pipeline/    LangGraph state, nodes, graph, LLM clients
rag/         Schema retrieval, vector store, Self-RAG critic
frontend/    React + Vite + Tailwind app
app.py       FastAPI entrypoint (CORS, router, health check)
main.py      CLI entrypoint for local testing
```

## Run locally

```bash
git clone https://github.com/Anshu1866/text_to_sql.git
cd text_to_sql
python -m venv venv && venv\Scripts\activate     # macOS/Linux: source venv/bin/activate
pip install -r requirements.txt

docker run --name pg-textsql -e POSTGRES_PASSWORD=yourpassword -e POSTGRES_DB=textsql -p 5432:5432 -d postgres
docker cp Chinook_PostgreSql.sql pg-textsql:/Chinook_PostgreSql.sql
docker exec -it pg-textsql psql -U postgres -d textsql -f /Chinook_PostgreSql.sql
```

`.env`:

```
GROQ_API_KEY=your_key_here
CHINOOK_DB_URL=postgresql://postgres:yourpassword@localhost:5432/chinook
CHINOOK_READONLY_DB_URL=postgresql://postgres:yourpassword@localhost:5432/chinook
APP_DB_URL=postgresql://postgres:yourpassword@localhost:5432/textsql
```

```bash
uvicorn app:app --reload
cd frontend && npm install && npm run dev
```

## Known limitations and roadmap

- [ ] Semantic validation: parse SQL and verify every column exists (execution success is not correctness)
- [ ] Automated tests: pytest with mocked LLM calls, plus a golden question/SQL set
- [ ] Shared session store (Redis) for multi-instance deployment
- [ ] AST-based SQL check allowing a single SELECT, row limits and statement timeouts
- [ ] Authentication, rate limiting, streaming responses

## Example Walkthrough

**1. A clear question** (runs straight through)

> "Which artist has the most albums?"

The system retrieves the relevant tables (Artist, Album), the critic confirms they are sufficient, SQL is generated, passes the keyword filter, runs on the read-only connection, and the response returns a plain-English answer together with the executed SQL.

**2. An ambiguous question** (pauses for clarification)

> "Show me the top ones"

The graph calls `interrupt()` and the API returns a clarifying question with a `thread_id`. The user replies, the frontend calls `/resume` with the same `thread_id`, and the run continues from the exact point it paused.

**3. A failing query** (self-correction)

If the generated SQL raises a database error, the actual error message is fed back to the generator and a corrected query is tried, up to the retry limit. If every attempt fails, the user gets a graceful failure message instead of a stack trace.

## API Overview

| Endpoint | Purpose |
|---|---|
| `GET /` | Health check, returns `{"status": "ok"}` |
| `POST /query` | Start a run. Returns an answer, an error, or a clarification request with a `thread_id` |
| `POST /resume` | Continue a paused run using the `thread_id` and the user's clarification |
| `GET /docs` | Interactive OpenAPI documentation (FastAPI) |

## Credits

Based on the open-source project [text-to-sql-aiagent](https://github.com/Alokik-29/text-to-sql-aiagent) by Alokik Gour.
