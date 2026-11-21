# Bollywood GraphRAG: Q&A over a Knowledge Graph, Evaluated

Ask questions about Hindi cinema in plain English. The system finds
connected facts in a Neo4j graph of movies, people, and awards, then a
language model writes the answer.

**Stack:** Neo4j · Docker · Python · FastAPI · Streamlit · Ollama (Llama 3.2)

## Why this matters
Keyword search finds text that looks like the question. It struggles with
questions that depend on connections, like "Which films did this actor
and this composer both work on?" A graph stores those connections
directly, so multi-hop questions become traceable lookups.

## What I did
- Deployed the full stack locally: Neo4j (Docker), FastAPI, Streamlit
- Loaded the graph: 26 movies, 47 people, 25 awards, 10 production houses,
  121 relationships
- Replaced the paid OpenAI API with local models (nomic-embed-text and
  Llama 3.2 via Ollama), removing API cost
- Resolved a dependency conflict (pydantic vs langchain) that blocked
  installation
- Tested answers against Cypher ground truth and added a prompt guardrail
- Corrected the UI caption, which claimed "GPT-4o" and "zero
  hallucination"

## Evaluation
I ran 5 multi-hop questions and checked answers against Cypher queries.

| Finding | Detail |
|---|---|
| Retrieval | Found the right entities every time |
| Answer quality | Weak point: the small model mixed up roles and added unsupported facts |
| Example | "Films by Aamir Khan and AR Rahman": the graph returns 2 (Ghajini, Lagaan). Before the prompt change the model listed 5 to 6. After it, 3. |
| Data gap | No gender field, so "actresses" questions cannot be answered reliably |

## Product takeaways
- Retrieval and answer generation fail independently, so test both
- "Zero hallucination" is not a claim a small model can support
- Next steps: add missing fields, try a stronger model, and automate
  comparison against Cypher ground truth

## Graph vs. keyword search
Graph search handles multi-hop questions better, but needs upfront data
modelling and ongoing upkeep.

## Credits
Built on the open-source Bollywood GraphRAG project by Ramendra611
(Codeverra, https://codeverra.com). The graph data, pipeline, and
original design are theirs. The setup, local-model switch, fixes,
evaluation, and write-up above are mine.

---

# Project documentation

## Knowledge Graph Ontology

```
NODES
Person          {name, born, profession, hometown}
Movie           {title, year, genre, box_office_crore, description}
ProductionHouse {name, founded, founder, hq}
Award           {name, category, year}

RELATIONSHIPS
(Person) -[:ACTED_IN {character, lead_role}]-> (Movie)
(Person) -[:DIRECTED]->                        (Movie)
(Person) -[:COMPOSED_MUSIC_FOR]->              (Movie)
(Person) -[:WON]->                             (Award)
(Movie)  -[:WON]->                             (Award)
(ProductionHouse) -[:PRODUCED]->               (Movie)
```

## How the pipeline works

```
User question
   │
   ▼
1. Vector search     Embed the question, find the top-k similar nodes
   │                 (embeddings.py, nomic-embed-text via Ollama)
   ▼
2. Graph traversal   Walk 2 hops from each node, collect facts as triples
   │                 (graphrag.py)
   ▼
3. Answer generation Llama 3.2 answers from that context only
                     (graphrag.py, via Ollama)
```

## Setup (macOS, local, no paid API)

### Prerequisites

| Tool | Notes |
|---|---|
| Docker Desktop | Must be running |
| Python 3.11 | 3.13 fails on pinned dependencies |
| Ollama | Free, from ollama.com |

### 1. Clone and configure

```bash
git clone https://github.com/sat05/bollywood-graph-rag.git
cd bollywood-graph-rag
cp .env.example .env
```

No OpenAI key is needed. The Neo4j defaults in `.env` work as they are.

### 2. Start Neo4j

```bash
docker compose up neo4j -d
```

Open http://localhost:7474 (user: `neo4j`, password: `bollywood2024!`).

### 3. Install Ollama models

```bash
ollama pull llama3.2
ollama pull nomic-embed-text
```

### 4. Python environment

```bash
conda create -n bollywood python=3.11 -y
conda activate bollywood
pip install -r requirements.txt
```

### 5. Load the graph and embeddings

```bash
cd src
python loader.py
python embeddings.py
```

### 6. Run the backend and the app

Terminal 1:
```bash
uvicorn api:app --reload --port 8000
```

Terminal 2:
```bash
conda activate bollywood
cd src
streamlit run app.py
```

Open http://localhost:8501.

> The original project also ships a `docker compose up --build` route.
> I have not tested it with the Ollama setup above, so use the manual
> steps.

## Try these

- "Which films did Aamir Khan and AR Rahman collaborate on?"
- "Which movies produced by Yash Raj Films have Shah Rukh Khan?"
- "What national awards have films from Aamir Khan Productions won?"

## API endpoints

```bash
curl -X POST http://localhost:8000/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "Which films did Aamir Khan direct?", "top_k": 3, "hops": 2}'

curl "http://localhost:8000/search?q=revenge+thriller&label=Movie"
curl "http://localhost:8000/graph/Dangal?label=Movie&hops=2"
curl "http://localhost:8000/person/AR%20Rahman/filmography"
curl "http://localhost:8000/stats"
```

## Project structure

```
bollywood-graph-rag/
├── docker-compose.yml
├── requirements.txt
├── .env.example
├── Dockerfile.api
├── Dockerfile.streamlit
└── src/
    ├── db.py            Neo4j connection wrapper
    ├── loader.py        Load ontology and data into Neo4j
    ├── embeddings.py    Compute and store node embeddings
    ├── graphrag.py      The GraphRAG pipeline
    ├── api.py           FastAPI backend
    ├── app.py           Streamlit frontend
    └── data/
        └── bollywood_data.py
```

## Extending the project

- **More movies:** add entries in `src/data/bollywood_data.py`, then re-run
  `loader.py` and `embeddings.py`.
- **New relationship type:** add rows in `bollywood_data.py`, add a loader
  function in `loader.py`, and call it from `load_all()`.
- **Different LLM:** `graphrag.py` and `embeddings.py` use the OpenAI client
  pointed at Ollama (`base_url="http://localhost:11434/v1"`). Any
  OpenAI-compatible endpoint works with a base URL and model name change.

## Cypher queries to try in Neo4j Browser

```cypher
// Films where Aamir Khan acted and AR Rahman composed
MATCH (:Person {name:'Aamir Khan'})-[:ACTED_IN]->(m:Movie)<-[:COMPOSED_MUSIC_FOR]-(:Person {name:'AR Rahman'})
RETURN m.title

// Directors who also acted in their own films
MATCH (p:Person)-[:DIRECTED]->(m:Movie)<-[:ACTED_IN]-(p)
RETURN p.name, m.title

// Shortest path between two people
MATCH path = shortestPath(
  (a:Person {name:'Shah Rukh Khan'})-[*]-(b:Person {name:'AR Rahman'}))
RETURN [n IN nodes(path) | coalesce(n.name, n.title)] AS path, length(path) AS hops
```
