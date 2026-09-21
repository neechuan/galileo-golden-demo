Ollama on an Amazon EC2 instance lets you host open-source large language models (like Llama 3.1, Qwen 2.5, or DeepSeek-R1) in the cloud with full control over your hardware and data.
https://github.com/rungalileo/galileo-golden-demo

# System
- EC2 instance type: g4dn.xlarge
- Operating system: Ubuntu 26.06 LTS
- SSD: 64GB (Minimum)


#  Install NVIDIA Drivers (GPU Instances Only)
```bash
sudo apt update
sudo apt install -y nvidia-driver-535-server nvidia-cuda-toolkit
sudo reboot    
```

#  Install and Run Ollama
```bash
curl -fsSL https://ollama.com/install.sh | sh
```bash
>>> Installing ollama to /usr/local
>>> Downloading ollama-linux-amd64.tar.zst
######################################################################## 100.0%
>>> Creating ollama user...
>>> Adding ollama user to render group...
>>> Adding ollama user to video group...
>>> Adding current user to ollama group...
>>> Creating ollama systemd service...
>>> Enabling and starting ollama service...
Created symlink '/etc/systemd/system/default.target.wants/ollama.service' → '/etc/systemd/system/ollama.service'.
>>> NVIDIA GPU installed.
```

# Configure Remote/Public Access (Optional): By default, Ollama binds only to 127.0.0.1 (localhost). To let external apps or a web UI talk to it, edit the systemd service
```bash
sudo systemctl edit ollama

[Service]
Environment="OLLAMA_HOST=0.0.0.0:11434"

OR
sudo vi /etc/systemd/system/ollama.service
```
[Unit]
Description=Ollama Service
After=network-online.target

[Service]
ExecStart=/usr/local/bin/ollama serve
User=ollama
Group=ollama
Restart=always
RestartSec=3
Environment="PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/usr/games:/usr/local/games:/snap/bin"
Environment="OLLAMA_HOST=0.0.0.0:11434"

[Install]
WantedBy=default.target
```

sudo systemctl daemon-reload
sudo systemctl restart ollama
```

# Verify Ollama is running
```bash
curl http://localhost:11434/api/tags
{"models":[]}
curl http://localhost:11434
Ollama is running
```

# Default chat model (agent + tool calling)
```bash
ollama pull gemma4
```

# Embedding model for RAG (required for retrieval)
```bash
ollama pull nomic-embed-text

ollama run gemma4 "Hello! Are you running on AWS?"

ollama list
```


# Optional additional chat models
```bash
ollama pull mistral
```

# Do keep in mind that you will need to download any other models you want to test with; also, the model needs to have tool and reasoning capabilities. Gemma4 was the only model to work consistently during testing, so make sure to test any other models thoroughly.
Ollama serves models at http://localhost:11434 by default. Override the URL in .streamlit/secrets.toml if needed:
```bash
ollama_base_url = "http://localhost:11434"
ollama_default_chat_model = "gemma4"
```

## Install Dockers
Install docker by following https://docs.docker.com/engine/install/ubuntu/

Run Docker Without Sudo
To avoid typing sudo for every command, add your user to the docker group:
```bash
sudo usermod -aG docker $USER
newgrp docker
docker run hello-world
```


# Start PostgreSQL with pgvector (Docker)
```bash
docker pull pgvector/pgvector:pg16

docker run -e POSTGRES_USER=postgres \
           -e POSTGRES_PASSWORD=mypassword \
           -e POSTGRES_DB=vectordb \
           --name golden-demo-postgres \
           -p 5432:5432 \
           -d pgvector/pgvector:pg16

docker ps
CONTAINER ID   IMAGE                    COMMAND                  CREATED         STATUS         PORTS                                         NAMES
42d2df2dfd90   pgvector/pgvector:pg16   "docker-entrypoint.s…"   7 seconds ago   Up 6 seconds   0.0.0.0:5432->5432/tcp, [::]:5432->5432/tcp   golden-demo-postgres
```

# Enable the pgvector extension
```bash
docker exec -it golden-demo-postgres psql -U postgres -d vectordb -c "CREATE EXTENSION IF NOT EXISTS vector;"
```

# Set PostgreSQL password for the 'postgres' user
```bash
docker exec -it golden-demo-postgres psql -U postgres
psql (16.15 (Debian 16.15-1.pgdg12+2))
Type "help" for help.

postgres=# ALTER USER postgres WITH PASSWORD 'postgres';
ALTER ROLE
postgres=# \q
```

# Clone the Galileo Golden Demo repository
```bash
git clone https://github.com/neechuan/galileo-golden-demo.git
cd galileo-golden-demo
```

# Install Python3 VENV
```bash
sudo apt install python3.14-venv
```

# Set up virtual environment
```bash
python3 -m venv venv
source venv/bin/activate 
```

# Install requirements
```bash
pip install -r requirements.txt
```

# Configure secrets Create .streamlit/secrets.toml with your API keys:
cd .streamlit 
cp secrets.toml.template secrets.toml          
vi secrets.toml

```
# API Keys
# -----------------------------------------------------------------------------
# Ollama (local LLM — no API key required)
ollama_base_url = "http://localhost:11434"
ollama_default_chat_model = "gemma4"
# Separate from chat models — used for RAG embeddings (run: ollama pull nomic-embed-text)
ollama_embedding_model = "nomic-embed-text"

# OpenAI (hosted LLM — required when using Hosted provider in the UI)
# openai_api_key = "..."
#openai_default_chat_model = "gpt-4o"
#openai_embedding_model = "text-embedding-3-large"

# AWS Bedrock (hosted LLM via AWS — required when using Bedrock provider in the UI)
# Auth uses a Bedrock API key (bearer token); no SigV4 access keys needed.
# Loaded into env as AWS_BEARER_TOKEN_BEDROCK / AWS_REGION.

# Paste your Amazon Bedrock API key
#bedrock_api_key = ""

# AWS region for Bedrock inference
#aws_region = "us-east-1"

#bedrock_default_chat_model = "mistral.ministral-3-14b-instruct"

# Separate from chat models — used for RAG embeddings (its own {domain}_bedrock_index)
#bedrock_embedding_model = "amazon.titan-embed-text-v2:0"

# Galileo Configuration
# -----------------------------------------------------------------------------
# Each domain automatically gets its own Galileo project:
#   Default: "galileo-demo-{domain_name}"
#   Example: finance domain → "galileo-demo-finance"
#
# To override the default project name for a domain, edit:
#   domains/{domain}/config.yaml
# -----------------------------------------------------------------------------

# To create an API Key, login to the Galileo UI Console, click on your username (could be on the upper right or bottom left corner, depending on the UI skin you're using)
# and click on API Keys. Then click on the  +Create new Key button.
galileo_api_key = "<YOUR_GALILEO_API_KEY>"

# Console URL for your Galileo instance
#galileo_console_url = "https://console....galileocloud.io/"
galileo_console_url = "https://console.multitenant.galileocloud.io/"  # Multitenant

# Agent Control (runtime guardrails)
# Enable controls in the Galileo UI -> Project -> Log Stream -> Controls

galileo_api_url = "https://api.multitenant.galileocloud.io"
agent_control_url = "https://console.multitenant.galileocloud.io/api/agent-control"

agent_control_agent_name = "golden-demo-agent"
agent_control_runtime_auth_mode="jwt"
agent_control_api_key_header = "Galileo-API-Key"
agent_control_target_type="log_stream"


# PostgreSQL Configuration (pgvector)
# -----------------------------------------------------------------------------
# PostgreSQL with pgvector extension for vector storage.
# See README for Docker setup instructions.
postgres_host = "localhost"
postgres_port = "5432"
postgres_user = "postgres"
postgres_password = "postgres"
postgres_db = "vectordb"

# Environment Configuration
# -----------------------------------------------------------------------------
# This is no longer configurable; don't change this value
# environment = "local"
# No longer used; don't change this value.
environment = "local"

# Live Finance Data (Optional)
# -----------------------------------------------------------------------------
# Alpha Vantage API key for backup stock data source.
# Get a free key at: https://www.alphavantage.co/support/#api-key
# Note: Yahoo Finance (yfinance) is used as the primary source and requires no key.
# PS: This key is not needed for any of the current Domains.
#alpha_vantage_api_key = ""

# Admin Configuration
# -----------------------------------------------------------------------------
# Used to authorize users for deleting experiments in the UI
admin_key = ""
```

#
```bash
python3 helpers/setup_vectordb.py bank
/home/ubuntu/galileo-golden-demo/helpers/setup_vectordb.py:39: DeprecationWarning: `langchain-community` is being sunset and is no longer actively maintained. See https://github.com/langchain-ai/langchain-community/issues/674 for details and migration guidance toward standalone integration packages.
  from langchain_community.document_loaders.csv_loader import CSVLoader
✓ Loaded configuration for domain: Bank assistant for online help and customer account support
ℹ️  OPENAI_API_KEY not configured in secrets.toml (cleared)
ℹ️  AWS_BEARER_TOKEN_BEDROCK not configured in secrets.toml (cleared)
⚠️  ADMIN_KEY not set (empty value)
🔧 Environment setup complete for domain: bank (project: galileo-demo)
Detecting configured providers from secrets.toml...
  • ollama: configured -> will build bank_local_index
  • openai: skipped (openai_api_key not set in .streamlit/secrets.toml (missing or placeholder))
  • bedrock: skipped (bedrock_api_key not set in .streamlit/secrets.toml)
Setting up vector database for domain: bank (building: ollama)
Loading documents from: domains/bank/docs
✓ Built 9 FAQ documents from qa.csv

▶ Building bank_local_index with ollama embeddings (model: nomic-embed-text)
   If the model is missing, run: ollama pull nomic-embed-text
Collection not found

Loading relational tables for bank...
✓ Loaded relational table bank_customer (30 rows) from relational_customer.csv

✅ Vector database ready for bank
   • bank_local_index  (ollama / nomic-embed-text) — 9 documents


python3 helpers/setup_vectordb.py healthcare
/home/ubuntu/galileo-golden-demo/helpers/setup_vectordb.py:39: DeprecationWarning: `langchain-community` is being sunset and is no longer actively maintained. See https://github.com/langchain-ai/langchain-community/issues/674 for details and migration guidance toward standalone integration packages.
  from langchain_community.document_loaders.csv_loader import CSVLoader
✓ Loaded configuration for domain: Healthcare assistant for online help and patient information support
ℹ️  OPENAI_API_KEY not configured in secrets.toml (cleared)
ℹ️  AWS_BEARER_TOKEN_BEDROCK not configured in secrets.toml (cleared)
⚠️  ADMIN_KEY not set (empty value)
🔧 Environment setup complete for domain: healthcare (project: galileo-demo)
Detecting configured providers from secrets.toml...
  • ollama: configured -> will build healthcare_local_index
  • openai: skipped (openai_api_key not set in .streamlit/secrets.toml (missing or placeholder))
  • bedrock: skipped (bedrock_api_key not set in .streamlit/secrets.toml)
Setting up vector database for domain: healthcare (building: ollama)
Loading documents from: domains/healthcare/docs
✓ Built 15 FAQ documents from qa.csv

▶ Building healthcare_local_index with ollama embeddings (model: nomic-embed-text)
   If the model is missing, run: ollama pull nomic-embed-text
Collection not found

Loading relational tables for healthcare...
✓ Loaded relational table healthcare_patient (30 rows) from relational_patient.csv

✅ Vector database ready for healthcare
   • healthcare_local_index  (ollama / nomic-embed-text) — 15 documents

python3 helpers/setup_vectordb.py insurance
/home/ubuntu/galileo-golden-demo/helpers/setup_vectordb.py:39: DeprecationWarning: `langchain-community` is being sunset and is no longer actively maintained. See https://github.com/langchain-ai/langchain-community/issues/674 for details and migration guidance toward standalone integration packages.
  from langchain_community.document_loaders.csv_loader import CSVLoader
✓ Loaded configuration for domain: insurance assistant for online help and customer account support
ℹ️  OPENAI_API_KEY not configured in secrets.toml (cleared)
ℹ️  AWS_BEARER_TOKEN_BEDROCK not configured in secrets.toml (cleared)
⚠️  ADMIN_KEY not set (empty value)
🔧 Environment setup complete for domain: insurance (project: galileo-demo)
Detecting configured providers from secrets.toml...
  • ollama: configured -> will build insurance_local_index
  • openai: skipped (openai_api_key not set in .streamlit/secrets.toml (missing or placeholder))
  • bedrock: skipped (bedrock_api_key not set in .streamlit/secrets.toml)
Setting up vector database for domain: insurance (building: ollama)
Loading documents from: domains/insurance/docs
✓ Built 9 FAQ documents from qa.csv

▶ Building insurance_local_index with ollama embeddings (model: nomic-embed-text)
   If the model is missing, run: ollama pull nomic-embed-text
Collection not found

Loading relational tables for insurance...
✓ Loaded relational table insurance_customer (30 rows) from relational_customer.csv

✅ Vector database ready for insurance
   • insurance_local_index  (ollama / nomic-embed-text) — 9 documents

python3 helpers/setup_vectordb.py restaurant
/home/ubuntu/galileo-golden-demo/helpers/setup_vectordb.py:39: DeprecationWarning: `langchain-community` is being sunset and is no longer actively maintained. See https://github.com/langchain-ai/langchain-community/issues/674 for details and migration guidance toward standalone integration packages.
  from langchain_community.document_loaders.csv_loader import CSVLoader
✓ Loaded configuration for domain: restaurant assistant for online help and patient information support
ℹ️  OPENAI_API_KEY not configured in secrets.toml (cleared)
ℹ️  AWS_BEARER_TOKEN_BEDROCK not configured in secrets.toml (cleared)
⚠️  ADMIN_KEY not set (empty value)
🔧 Environment setup complete for domain: restaurant (project: galileo-demo)
Detecting configured providers from secrets.toml...
  • ollama: configured -> will build restaurant_local_index
  • openai: skipped (openai_api_key not set in .streamlit/secrets.toml (missing or placeholder))
  • bedrock: skipped (bedrock_api_key not set in .streamlit/secrets.toml)
Setting up vector database for domain: restaurant (building: ollama)
Loading documents from: domains/restaurant/docs
✓ Built 19 FAQ documents from qa.csv

▶ Building restaurant_local_index with ollama embeddings (model: nomic-embed-text)
   If the model is missing, run: ollama pull nomic-embed-text
Collection not found

Loading relational tables for restaurant...
✓ Loaded relational table restaurant_schedule (40 rows) from relational_schedule.csv

✅ Vector database ready for restaurant
   • restaurant_local_index  (ollama / nomic-embed-text) — 19 documents
```

# Run the Streamlit app
```bash
streamlit run app.py
Collecting usage statistics. To deactivate, set browser.gatherUsageStats to false.

2026-09-21 04:50:25.816 Uvicorn server started on :::8501

  You can now view your Streamlit app in your browser.

  Local URL: http://localhost:8501
  Network URL: http://172.31.22.28:8501
  External URL: http://3.106.120.87:8501
```