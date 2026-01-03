# Project: Tamil Cinema Knowledge Graph RAG - Flask API

## Project Overview
Build a Knowledge Graph RAG (Retrieval Augmented Generation) backend API using Flask that allows users to ask natural language questions about Tamil Cinema and get accurate answers by querying a Neo4j knowledge graph.

## Learning Objectives
1. Understand Knowledge Graph pipeline
2. Build Entity and Relationship extraction using LLM
3. Store data in Neo4j graph database
4. Implement KG-RAG for intelligent Q&A
5. Create REST API with Flask

## Tech Stack
- Python 3.10+
- Flask (REST API)
- Neo4j (Graph Database)
- Anthropic Claude API (LLM for extraction & answering)
- python-dotenv (Configuration)
- spaCy (Basic NLP - optional)

## Project Structure
```
kg-rag-tamil-cinema/
├── app/
│   ├── __init__.py
│   ├── config.py              # Configuration settings
│   ├── neo4j_client.py        # Neo4j database operations
│   ├── llm_client.py          # Claude API wrapper
│   ├── extraction/
│   │   ├── __init__.py
│   │   ├── entity_extractor.py    # Extract entities from text
│   │   └── relation_extractor.py  # Extract relationships
│   ├── kg_builder/
│   │   ├── __init__.py
│   │   └── graph_builder.py   # Build graph from extractions
│   ├── rag/
│   │   ├── __init__.py
│   │   ├── query_generator.py # Convert NL to Cypher
│   │   ├── retriever.py       # Retrieve from graph
│   │   └── answer_generator.py # Generate final answer
│   └── routes/
│       ├── __init__.py
│       ├── ingest.py          # Data ingestion endpoints
│       ├── query.py           # Q&A endpoints
│       └── graph.py           # Graph exploration endpoints
├── data/
│   └── sample_documents.json  # Sample Tamil cinema data
├── scripts/
│   ├── seed_database.py       # Seed initial data
│   └── test_pipeline.py       # Test the full pipeline
├── .env.example
├── requirements.txt
├── run.py
└── README.md
```

## Environment Variables (.env)
```
# Neo4j Configuration
NEO4J_URI=bolt://localhost:7687
NEO4J_USER=neo4j
NEO4J_PASSWORD=your_password

# Claude API
ANTHROPIC_API_KEY=your_api_key

# Flask Configuration
FLASK_DEBUG=True
FLASK_PORT=5000
```

## requirements.txt
```
flask==3.0.0
flask-cors==4.0.0
neo4j==5.15.0
anthropic==0.18.0
python-dotenv==1.0.0
pydantic==2.5.0
```

## Core Components to Build

### 1. Neo4j Client (app/neo4j_client.py)
```python
"""
Neo4j database client with these methods:
- connect() / close()
- run_query(cypher, params)
- create_node(label, properties)
- create_relationship(from_node, to_node, rel_type, properties)
- get_node(label, name)
- get_neighbors(name, rel_type=None)
- find_path(name1, name2)
- search(query_text)
- get_full_graph()
- clear_database()
"""
```

### 2. LLM Client (app/llm_client.py)
```python
"""
Claude API wrapper with these methods:
- extract_entities(text) -> dict
- extract_relations(text, entities) -> list[dict]
- text_to_cypher(question, schema) -> str
- generate_answer(question, context) -> str
"""
```

### 3. Entity Extractor (app/extraction/entity_extractor.py)
```python
"""
Extract entities using Claude API.

Input: Raw text about Tamil cinema
Output: Dictionary with categorized entities

Example:
Input: "Rajinikanth starred in Jailer directed by Nelson in 2023"
Output: {
    "persons": [
        {"name": "Rajinikanth", "type": "Actor"},
        {"name": "Nelson", "type": "Director"}
    ],
    "movies": [
        {"title": "Jailer", "year": 2023}
    ],
    "organizations": [],
    "locations": []
}

Prompt Template:
'''
Extract all named entities from this Tamil cinema text.
Categorize them as: persons (actors/directors), movies, organizations, locations.

Text: {text}

Return JSON format:
{
    "persons": [{"name": "", "type": "Actor/Director"}],
    "movies": [{"title": "", "year": null}],
    "organizations": [{"name": ""}],
    "locations": [{"name": ""}]
}
'''
"""
```

### 4. Relation Extractor (app/extraction/relation_extractor.py)
```python
"""
Extract relationships between entities.

Input: Text + Extracted entities
Output: List of triples (subject, predicate, object)

Example:
Input Text: "Rajinikanth starred in Jailer directed by Nelson"
Input Entities: {persons: [Rajinikanth, Nelson], movies: [Jailer]}

Output: [
    {"subject": "Rajinikanth", "predicate": "ACTED_IN", "object": "Jailer"},
    {"subject": "Nelson", "predicate": "DIRECTED", "object": "Jailer"}
]

Supported Relationships:
- ACTED_IN (Person -> Movie)
- DIRECTED (Person -> Movie)
- PRODUCED (Organization -> Movie)
- BORN_IN (Person -> Location)
- RELEASED_IN (Movie -> Year)

Prompt Template:
'''
Given these entities and text, extract relationships as triples.

Entities: {entities}
Text: {text}

Valid relationship types: ACTED_IN, DIRECTED, PRODUCED, BORN_IN, RELEASED_IN

Return JSON:
{
    "triples": [
        {"subject": "", "predicate": "", "object": "", "properties": {}}
    ]
}
'''
"""
```

### 5. Graph Builder (app/kg_builder/graph_builder.py)
```python
"""
Build knowledge graph from extracted data.

Methods:
- build_from_text(text) -> dict (stats)
- build_from_triples(triples) -> dict
- add_entity(entity_type, properties)
- add_relationship(triple)

Flow:
1. Extract entities from text
2. Extract relationships
3. Create nodes in Neo4j
4. Create relationships in Neo4j
5. Return statistics
"""
```

### 6. Query Generator (app/rag/query_generator.py)
```python
"""
Convert natural language questions to Cypher queries.

Input: User question + Graph schema
Output: Cypher query string

Example:
Input: "What movies did Rajinikanth act in?"
Output: "MATCH (p:Person {name: 'Rajinikanth'})-[:ACTED_IN]->(m:Movie) RETURN m.title, m.year"

Graph Schema (to include in prompt):
- Node Labels: Person, Movie, Organization, Location
- Person properties: name, type (Actor/Director), born_year
- Movie properties: title, year, genre
- Relationships: ACTED_IN, DIRECTED, PRODUCED, BORN_IN

Prompt Template:
'''
Convert this question to a Cypher query for Neo4j.

Graph Schema:
{schema}

Question: {question}

Return only the Cypher query, no explanation.
'''
"""
```

### 7. Retriever (app/rag/retriever.py)
```python
"""
Retrieve relevant context from knowledge graph.

Methods:
- retrieve(question) -> dict
  1. Generate Cypher from question
  2. Execute query on Neo4j
  3. Format results as context
  
- retrieve_with_expansion(question) -> dict
  1. Basic retrieval
  2. Expand to include related nodes (1-hop neighbors)
  
Example:
Question: "Tell me about Jailer movie"
Retrieved Context: {
    "main_results": [
        {"title": "Jailer", "year": 2023, "genre": "Action"}
    ],
    "related": [
        {"name": "Rajinikanth", "role": "Actor"},
        {"name": "Nelson", "role": "Director"}
    ],
    "cypher_used": "MATCH (m:Movie {title: 'Jailer'})..."
}
"""
```

### 8. Answer Generator (app/rag/answer_generator.py)
```python
"""
Generate natural language answers using retrieved context.

Input: Question + Retrieved context
Output: Natural language answer

Prompt Template:
'''
Answer this question using ONLY the provided context from our Tamil Cinema knowledge graph.

Question: {question}

Context from Knowledge Graph:
{context}

Rules:
- Only use information from the context
- If information is not available, say so
- Be concise and accurate
- Mention specific movies, actors, directors by name

Answer:
'''
"""
```

## Flask API Endpoints

### Ingest Routes (app/routes/ingest.py)
```python
# POST /api/ingest/text
# Ingest raw text and build knowledge graph
# Body: {"text": "Rajinikanth starred in Jailer..."}
# Response: {"status": "success", "entities_added": 5, "relations_added": 3}

# POST /api/ingest/document
# Ingest a document (JSON with title and content)
# Body: {"title": "Movie Info", "content": "..."}

# POST /api/ingest/batch
# Ingest multiple texts
# Body: {"texts": ["text1", "text2"]}
```

### Query Routes (app/routes/query.py)
```python
# POST /api/query
# Ask a question and get KG-RAG answer
# Body: {"question": "What movies did Vijay act in?"}
# Response: {
#     "question": "...",
#     "answer": "Vijay acted in Leo (2023), Master (2021)...",
#     "context": {...},
#     "cypher_query": "MATCH..."
# }

# POST /api/query/cypher
# Execute raw Cypher query
# Body: {"cypher": "MATCH (n) RETURN n LIMIT 10"}

# GET /api/query/suggestions
# Get suggested questions based on graph content
```

### Graph Routes (app/routes/graph.py)
```python
# GET /api/graph/stats
# Get graph statistics
# Response: {"nodes": 50, "relationships": 120, "persons": 20, "movies": 30}

# GET /api/graph/full
# Get full graph for visualization
# Response: {"nodes": [...], "edges": [...]}

# GET /api/graph/person/<name>
# Get person details and connections
# Response: {"name": "Rajinikanth", "movies": [...], "collaborators": [...]}

# GET /api/graph/movie/<title>
# Get movie details
# Response: {"title": "Jailer", "cast": [...], "director": "Nelson"}

# GET /api/graph/path
# Find connection between two entities
# Query params: ?from=Rajinikanth&to=Vijay
# Response: {"path": ["Rajinikanth", "Jailer", "Nelson", "Beast", "Vijay"]}

# DELETE /api/graph/clear
# Clear entire graph (use with caution!)
```

## Sample Data for Testing (data/sample_documents.json)
```json
{
  "documents": [
    {
      "id": 1,
      "text": "Rajinikanth, the legendary Tamil actor, starred in Jailer (2023) which was directed by Nelson Dilipkumar. The movie became a massive blockbuster. Rajinikanth previously worked with director Shankar in Enthiran (2010) and 2.0 (2018)."
    },
    {
      "id": 2,
      "text": "Vijay's Leo (2023) was directed by Lokesh Kanagaraj. Lokesh earlier directed Vikram (2022) starring Kamal Haasan and Kaithi (2019) with Karthi. Vijay also worked with director Atlee in Mersal (2017) and Bigil (2019)."
    },
    {
      "id": 3,
      "text": "Suriya delivered powerful performances in Soorarai Pottru (2020) directed by Sudha Kongara and Jai Bhim (2021) directed by T.J. Gnanavel. Dhanush acted in Asuran (2019) and Karnan (2021), both directed by Mari Selvaraj."
    },
    {
      "id": 4,
      "text": "Mani Ratnam's epic Ponniyin Selvan 1 (2022) featured Vikram as Aditya Karikalan, Karthi as Vanthiyathevan, and Trisha as Kundavai. The historical drama was one of the biggest Tamil productions ever."
    },
    {
      "id": 5,
      "text": "Pa. Ranjith directed Rajinikanth in Kabali (2016) and Kaala (2018). Vetrimaaran is known for directing Dhanush in acclaimed films like Vada Chennai (2018) and Asuran (2019)."
    }
  ]
}
```

## Main Application (run.py)
```python
from flask import Flask
from flask_cors import CORS
from app.routes import ingest, query, graph
from app.config import Config

def create_app():
    app = Flask(__name__)
    app.config.from_object(Config)
    
    CORS(app)
    
    # Register blueprints
    app.register_blueprint(ingest.bp, url_prefix='/api/ingest')
    app.register_blueprint(query.bp, url_prefix='/api/query')
    app.register_blueprint(graph.bp, url_prefix='/api/graph')
    
    @app.route('/')
    def home():
        return {
            "message": "Tamil Cinema Knowledge Graph RAG API",
            "version": "1.0.0",
            "endpoints": {
                "ingest": "/api/ingest",
                "query": "/api/query", 
                "graph": "/api/graph"
            }
        }
    
    @app.route('/health')
    def health():
        # Check Neo4j connection
        # Return health status
        pass
    
    return app

if __name__ == '__main__':
    app = create_app()
    app.run(debug=True, port=5000)
```

## Testing Script (scripts/test_pipeline.py)
```python
"""
Test the complete KG-RAG pipeline:

1. Clear database
2. Ingest sample documents
3. Test queries:
   - "What movies did Rajinikanth act in?"
   - "Who directed Jailer?"
   - "Which director worked with both Vijay and Kamal?"
   - "Find connection between Dhanush and Karthi"
4. Print results and verify accuracy
"""
```

## Acceptance Criteria

1. ✅ Entity extraction works for Tamil cinema text
2. ✅ Relationship extraction identifies correct connections
3. ✅ Graph is properly built in Neo4j
4. ✅ Natural language questions are converted to Cypher
5. ✅ Answers are generated using graph context
6. ✅ All API endpoints return proper responses
7. ✅ Error handling for invalid queries
8. ✅ Graph visualization endpoint works

## Development Steps

Follow this order:
1. Setup project structure and install dependencies
2. Implement Neo4j client and test connection
3. Implement LLM client with Claude API
4. Build entity extractor and test
5. Build relation extractor and test
6. Implement graph builder
7. Create retriever and query generator
8. Implement answer generator
9. Create Flask routes
10. Test complete pipeline
11. Add error handling and logging

Start building!
