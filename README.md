# Car GraphRAG

A full-stack GraphRAG (Graph Retrieval-Augmented Generation) application that lets you search a live car marketplace using plain, natural language instead of dropdowns and filters.

Type something like *"Show me white Citroën C3 cars with less than 100,000 km and manual transmission"* and the app translates it into a Cypher query, runs it against a Neo4j knowledge graph, and returns exact matches plus intelligently broadened "similar" results when there aren't enough exact hits.

![Python](https://img.shields.io/badge/Python-3.11-3776AB?logo=python&logoColor=white)
![FastAPI](https://img.shields.io/badge/FastAPI-0.116-009688?logo=fastapi&logoColor=white)
![Neo4j](https://img.shields.io/badge/Neo4j-Graph_DB-4581C3?logo=neo4j&logoColor=white)
![OpenAI](https://img.shields.io/badge/OpenAI-GPT--4o--mini-412991?logo=openai&logoColor=white)
![React](https://img.shields.io/badge/React-18-61DAFB?logo=react&logoColor=black)
![TypeScript](https://img.shields.io/badge/TypeScript-5-3178C6?logo=typescript&logoColor=white)
![Vite](https://img.shields.io/badge/Vite-5-646CFF?logo=vite&logoColor=white)

---

## Overview

Car GraphRAG is an end-to-end pipeline that turns unstructured classifieds listings into a queryable knowledge graph, then exposes that graph through a conversational, chat-style search UI:

1. **Scrape** — real car listings are collected from a classifieds site and cleaned into a structured CSV (`scraper.py`).
2. **Build the knowledge graph** — the CSV is converted into a graph of `Brand`, `Car`, and `Seller` nodes in Neo4j using LangChain's `LLMGraphTransformer` (`kg.ipynb`).
3. **Query with GraphRAG** — a FastAPI backend uses an LLM to translate natural language into Cypher on the fly, executes it against Neo4j, and falls back to a broadened, similarity-based query if too few exact matches are found (`backend/relaxation.py`).
4. **Search & browse** — a React chat interface presents results as an AI conversation, complete with car cards, images, and a "would you like to refine your search" follow-up.

## Key Features

- **Natural language to Cypher** — no query language or filter UI required; users type what they want in plain English.
- **Two-stage retrieval** — an exact-match Cypher query is always run first; if results are sparse, a second query automatically relaxes numeric ranges (kilometers, year, price) and drops non-essential filters to surface similar cars.
- **Domain-aware prompt rules** — handles unit variants (`kw` vs `ks`), numbers written in words, typos, singular/plural mismatches, and synonym mapping (e.g. "gearbox", "transmission", "gear" → `gearbox`) so the LLM produces schema-correct Cypher.
- **JWT authentication** — user registration and login with hashed passwords (`passlib` + `bcrypt`) and OAuth2-style bearer tokens.
- **Conversational UI** — a ChatGPT-style interface with animated "thinking" states, result cards, and a persistent chat history sidebar.
- **Custom scraper** — polite, rate-limited scraping with data normalization (phone number cleanup, category translation, price parsing).

## Architecture

```
┌──────────────┐      natural language        ┌───────────────────┐
│              │ ───────────────────────────▶ │                    │
│  React SPA   │                               │   FastAPI Backend  │
│ (Vite + TS)  │ ◀─────────────────────────── │                    │
└──────────────┘      JSON results             └─────────┬──────────┘
                                                           │
                                          NL → Cypher (LLM)│  JWT auth (SQLite)
                                                           ▼
                                                 ┌───────────────────┐
                                                 │   Neo4j Graph DB   │
                                                 │ Brand / Car /      │
                                                 │ Seller nodes       │
                                                 └─────────▲──────────┘
                                                           │
                                          LLMGraphTransformer (kg.ipynb)
                                                           │
                                                 ┌───────────────────┐
                                                 │  scraper.py → CSV  │
                                                 │  (raw listings)    │
                                                 └───────────────────┘
```

## Tech Stack

| Layer            | Technology                                                             |
|-------------------|-------------------------------------------------------------------------|
| Frontend          | React 18, TypeScript, Vite, Tailwind CSS, shadcn/ui, TanStack Query    |
| Backend           | FastAPI, Pydantic, Uvicorn                                             |
| Graph database    | Neo4j                                                                  |
| LLM / GraphRAG    | OpenAI (`gpt-4o-mini`), LangChain (`LLMGraphTransformer`)              |
| Auth              | SQLAlchemy + SQLite, JWT (`python-jose`), `passlib`/`bcrypt`           |
| Data pipeline     | `requests`, `BeautifulSoup4`, `pandas`                                 |

## Project Structure

```
GraphRAG_Project/
├── backend/
│   ├── main.py         # FastAPI app, CORS, routes
│   ├── relaxation.py    # Natural language → Cypher (GraphRAG core)
│   ├── auth.py          # Registration, login, JWT issuing/validation
│   ├── database.py      # Neo4j driver connection
│   ├── config.py        # Environment-based configuration
│   ├── models.py        # SQLAlchemy user model
│   └── userdb.py         # SQLite engine/session setup
├── frontend/
│   └── src/
│       ├── components/  # Chat UI, hero section, sidebar, car cards
│       ├── pages/        # Login, Register, Index, NotFound
│       ├── services/     # Axios API client
│       └── types/        # Shared TypeScript types
├── scraper.py            # Scrapes and cleans car listings into CSV
├── kg.ipynb              # Builds the Neo4j knowledge graph from the CSV
├── cars_reklama5.csv     # Sample scraped dataset
└── requirements.txt
```

## Getting Started

### Prerequisites

- Python 3.11+
- Node.js 18+ (with `npm` or `bun`)
- A running Neo4j instance (local or [Aura](https://neo4j.com/cloud/platform/aura-graph-database/))
- An OpenAI API key

### 1. Backend setup

```bash
pip install -r requirements.txt
```

Create a `.env` file in the project root (see `.env.example`):

```env
OPENAI_API_KEY=
NEO4J_URI=
NEO4J_USER=
NEO4J_PASSWORD=
NEO4J_DATABASE=

SECRET_KEY=
ALGORITHM=HS256
```

Run the API:

```bash
uvicorn backend.main:app --reload --port 8080
```

### 2. Build the knowledge graph

With Neo4j running and `.env` configured, open `kg.ipynb` and run it end-to-end. It will:

1. Load `cars_reklama5.csv` (or your own scraped dataset).
2. Convert each listing into graph documents via `LLMGraphTransformer`.
3. Write `Brand`, `Car`, and `Seller` nodes and their relationships into Neo4j.

To scrape a fresh dataset instead of using the sample CSV:

```bash
python scraper.py
```

### 3. Frontend setup

```bash
cd frontend
npm install
npm run dev
```

Create a `.env` file in `frontend/` (see `frontend/.env.example`):

```env
VITE_API_BASE_URL="localhost:8080/api"
```

The app will be available at `http://localhost:8080` (Vite dev server).

## API

| Method | Endpoint         | Description                                              |
|--------|------------------|-----------------------------------------------------------|
| POST   | `/auth/`         | Register a new user                                       |
| POST   | `/auth/token`    | Log in and receive a JWT access token                      |
| GET    | `/`              | Get the currently authenticated user                       |
| POST   | `/api/search`    | Run a natural language search and return exact + similar matches |

## Data Model

```
(:Brand {name})
(:Car   {model, kilometers, car_body, color, engine_power, fuel,
         gearbox, image_url, year, show_class, registration,
         registered_to, link, price})
(:Seller {name, phone})

(:Car)-[:BELONGS_TO]->(:Brand)
(:Car)-[:IS_SOLD_BY]->(:Seller)
```

## Notes

This project was built as a hands-on exploration of GraphRAG — combining LLM-driven query generation with a graph database instead of the more common vector-store RAG pattern — applied to a real-world scraped dataset end to end, from data collection to a usable search UI.
