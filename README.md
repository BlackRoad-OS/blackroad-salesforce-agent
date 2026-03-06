# BlackRoad Salesforce Agent

Autonomous agent that maximizes a single Salesforce API seat. Instead of paying per human user, this agent handles CRM operations programmatically — 24/7, at API speed.

## Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    BLACKROAD SALESFORCE AGENT                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐              │
│  │   OAuth      │  │   SOQL      │  │   Bulk      │              │
│  │   Manager    │  │   Engine    │  │   Processor  │              │
│  └──────┬───────┘  └──────┬──────┘  └──────┬──────┘              │
│         │                 │                 │                     │
│  ┌──────┴─────────────────┴─────────────────┴──────┐             │
│  │              Salesforce REST API v59.0           │             │
│  └──────────────────────┬──────────────────────────┘             │
│                         │                                        │
│  ┌──────────────────────┴──────────────────────────┐             │
│  │               Task Queue (SQLite)                │             │
│  │   • Priority-based ordering                      │             │
│  │   • Automatic retries (configurable)             │             │
│  │   • Per-agent task assignment                    │             │
│  └──────────────────────────────────────────────────┘             │
│                                                                  │
│  ┌──────────────────────────────────────────────────┐            │
│  │              Agent (ThreadPoolExecutor)           │            │
│  │   • Configurable worker threads                  │            │
│  │   • Adaptive polling                             │            │
│  │   • Graceful shutdown (SIGINT/SIGTERM)           │            │
│  └──────────────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────────────┘
```

## Features

- **OAuth 2.0 Authentication** — Username-password flow with automatic token refresh and disk caching
- **SFDX CLI Auth Reuse** — Reuse existing Salesforce CLI tokens (no Connected App needed for dev)
- **Full CRUD Operations** — Create, Read, Update, Delete, Upsert on all Salesforce objects
- **SOQL Query Engine** — Queries with automatic pagination via `query_all()`
- **SOSL Search** — Full-text search across objects
- **Composite API** — Multiple operations in a single API call
- **Bulk API 2.0** — Insert, update, upsert, delete for large record sets (10,000+ records/batch)
- **Task Queue** — SQLite-backed priority queue with retry logic
- **Autonomous Mode** — Threaded agent pulls tasks from queue and processes them 24/7
- **Daemon Mode** — Systemd-compatible daemon for production deployment

## Project Structure

```
src/
├── __init__.py                   # Public API exports
├── main.py                       # Entry point with YAML config loading
├── daemon.py                     # Systemd daemon mode
├── run_sfdx.py                   # SFDX auth mode with interactive CLI
├── auth/
│   └── oauth.py                  # OAuth 2.0 + SFDX token management
├── api/
│   ├── client.py                 # REST API client (CRUD, SOQL, Composite)
│   └── bulk.py                   # Bulk API 2.0 client
├── agents/
│   └── salesforce_agent.py       # Core autonomous agent
└── queue/
    └── task_queue.py             # SQLite-backed task queue

tests/
└── test_queue.py                 # Task queue tests

scripts/
├── deploy.sh                     # Single-node deployment
└── swarm.sh                      # Multi-instance agent controller

config/
└── config.example.yaml           # Configuration template
```

## Installation

```bash
git clone https://github.com/BlackRoad-OS/blackroad-salesforce-agent.git
cd blackroad-salesforce-agent

pip install -r requirements.txt

# Configure
cp config/config.example.yaml config/config.yaml
# Edit config.yaml with your Salesforce credentials
```

## Configuration

```yaml
salesforce:
  client_id: "YOUR_CONNECTED_APP_CLIENT_ID"
  client_secret: "YOUR_CONNECTED_APP_CLIENT_SECRET"
  username: "your.email@company.com"
  password: "your_password"
  security_token: "your_security_token"
  domain: "login"  # "login" for production, "test" for sandbox

agent:
  max_workers: 4
  poll_interval: 1.0
  batch_size: 200

queue:
  backend: "sqlite"
  db_path: "~/.blackroad/task_queue.db"
```

Environment variables override config values: `SF_CLIENT_ID`, `SF_CLIENT_SECRET`, `SF_USERNAME`, `SF_PASSWORD`, `SF_SECURITY_TOKEN`, `SF_DOMAIN`.

## Usage

### Direct API Access

```python
from src.agents import SalesforceAgent
from src.agents.salesforce_agent import AgentConfig

config = AgentConfig(
    client_id="...",
    client_secret="...",
    username="...",
    password="...",
    security_token="..."
)
agent = SalesforceAgent(config)
agent.auth.authenticate()

# Query
records = agent.query(
    "SELECT Id, Name FROM Account WHERE CreatedDate = TODAY"
)

# Create
record_id = agent.create('Account', {'Name': 'Acme Corp'})

# Update
agent.update('Account', record_id, {'Name': 'Acme Corporation'})

# Delete
agent.delete('Account', record_id)
```

### Autonomous Mode

```python
# Start the agent — it will pull tasks from the queue and process them
agent.start()

# Submit tasks to the queue (from another process or thread)
agent.submit_task("create", "Account", {"Name": "New Account"})
agent.submit_task("query", "Account", {"soql": "SELECT Id FROM Account LIMIT 10"})
```

### SFDX Auth Mode (Development)

```bash
# Reuse your existing SF CLI auth — no Connected App required
python -m src.run_sfdx --username your.email@company.com --test
```

### Daemon Mode

```bash
python -m src.daemon --username your.email@company.com
```

### Running from Config

```bash
python -m src.main --config config/config.yaml --workers 8
```

## Running Tests

```bash
python -m pytest tests/ -v
```

## Deployment

```bash
# Deploy single instance
./scripts/deploy.sh

# Deploy multiple instances
./scripts/deploy.sh --count 10

# Manage swarm
./scripts/swarm.sh status
./scripts/swarm.sh start 50
./scripts/swarm.sh scale 100
./scripts/swarm.sh stop
```

## API Limits

Salesforce API limits per edition:
- **15,000+ API calls/day** on basic plans
- **Bulk API**: 10,000 records/batch
- **Composite API**: Multiple operations per call

The agent optimizes usage through request batching, bulk operations, and composite requests.

## License

BlackRoad OS, Inc. — Proprietary. See [LICENSE](LICENSE).
