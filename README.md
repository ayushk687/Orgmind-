# OrgMind: A Company Brain on FalkorDB

**Track: Company Brain** (Graph Hacks: Context for AI Agents)

OrgMind connects people, teams, projects, documents, tickets, chat messages and decisions in one
FalkorDB knowledge graph. An agent answers *who owns what*, *who knows what* and *why was this decided*,
and every answer is produced by Cypher traversals and cited back to source documents.

> The graph is the product: ownership, expertise scores and decision lineage are all traversals.
> Remove FalkorDB and the agent has nothing to answer from.

## Setup (local)
```bash
git clone <your-repo-url> && cd orgmind
docker compose up -d                  # FalkorDB on :6379 (browser UI on :3000)
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env                  # ANTHROPIC_API_KEY is optional
python -m scripts.ingest              # loads synthetic data into the graph
uvicorn app.main:app --reload         # open http://localhost:8000
```
Without an API key the agent returns answers composed directly from query results.
With `ANTHROPIC_API_KEY` set, Claude only rephrases those graph facts.

## Graph data model
**Nodes:** `Person{id,name,role}`, `Team`, `Project`, `Document{title,date}`, `Ticket{title,date}`,
`Message{text,channel,date}`, `Decision{summary,rationale,date}`, `Topic{name}`

**Relationships**
| Edge | Meaning |
|---|---|
| `(Person)-[:MEMBER_OF]->(Team)` | team membership |
| `(Person)-[:OWNS]->(Project)` | ownership |
| `(Person)-[:AUTHORED]->(Document\|Message)` | authorship |
| `(Person)-[:ASSIGNED_TO]->(Ticket)` | work assignment |
| `(Document\|Ticket\|Message)-[:ABOUT]->(Topic)` | topic tagging (drives "who knows what") |
| `(Document\|Ticket\|Message)-[:PART_OF]->(Project)` | project context |
| `(Decision)-[:ABOUT]->(Topic)`, `[:ABOUT_PROJECT]->(Project)` | decision scope |
| `(Decision)-[:DECIDED_BY]->(Person)` | who decided |
| `(Decision)-[:SUPPORTED_BY]->(Document\|Ticket\|Message)` | evidence trail |
| `(Decision)-[:SUPERSEDES]->(Decision)` | decision history |

## Cypher the product depends on
All queries live in [`app/queries.py`](app/queries.py). Key ones:
```cypher
-- Who owns X
MATCH (p:Person)-[:OWNS]->(pr:Project) WHERE toLower(pr.name)=toLower($project) RETURN p.name

-- Who knows topic X (expertise = activity linked to the topic)
MATCH (p:Person)-[:AUTHORED|ASSIGNED_TO]->(x)-[:ABOUT]->(t:Topic {name:$topic})
RETURN p.name, count(DISTINCT x) AS score ORDER BY score DESC LIMIT 5

-- Why was a decision made (trace to evidence and what it replaced)
MATCH (d:Decision)-[:ABOUT]->(t:Topic {name:$topic})
OPTIONAL MATCH (d)-[:DECIDED_BY]->(p:Person)
OPTIONAL MATCH (d)-[:SUPPORTED_BY]->(e)
OPTIONAL MATCH (d)-[:SUPERSEDES]->(old:Decision)
RETURN d.summary, d.rationale, collect(p.name), collect(coalesce(e.title,e.text)), old.summary

-- Decision lineage (variable-length traversal)
MATCH path=(d:Decision)-[:SUPERSEDES*1..5]->(old:Decision) RETURN path
```

## Demo script
1. "Who owns Billing Service?"
2. "Who knows Kubernetes?"
3. "Why did we migrate billing to PostgreSQL?" (shows rationale, deciders, evidence, and the superseded MongoDB decision)

## Data
All data in `data/seed.json` is synthetic (fictional company "Nimbus Labs").

## AI-use disclosure
Claude (Anthropic) was used as a coding assistant to scaffold this project. The team reviewed the code
and can explain the graph model, queries and architecture.

## Roadmap
Live connectors (Slack, Jira, Google Docs), LLM-based entity and decision extraction, time-decayed expertise scores.
