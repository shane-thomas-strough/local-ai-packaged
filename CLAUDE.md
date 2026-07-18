# CLAUDE.md

> **Cross-project doctrine:** this project follows the one engineering law — reason from
> first principles on the moat, ride the paved road on the scaffolding, never hand back a
> blocker you haven't tried to break yourself, and verify against ground truth, not memory.
> It lives in [`../AI_CODING_DOCTRINE.md`](../AI_CODING_DOCTRINE.md) (root of `projects_root`)
> and applies here.

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Quick Start

### One-Click Launcher
The easiest way to start the Local AI Package:

**Windows:**
- Double-click `START_LOCAL_AI.bat`
- Or run `python launcher.py` for GUI mode
- Or run `powershell -ExecutionPolicy Bypass -File START_LOCAL_AI.ps1`

**Mac/Linux:**
- Run `./start_local_ai.sh`
- Or run `python3 launcher.py` for GUI mode

The launcher will:
1. Check Docker is running (starts it if needed)
2. Generate secure secrets and create .env file automatically
3. Start all services with progress monitoring
4. Open browser tabs when services are ready
5. Save credentials to `credentials.txt`

### Command Line Options
```bash
# Run with GUI interface (default)
python launcher.py

# Run in CLI mode
python launcher.py --cli

# Specify GPU profile
python launcher.py --cli --profile gpu-nvidia

# Don't auto-open browsers
python launcher.py --cli --no-browser
```

## Common Development Commands

### Starting Services (Manual)
```bash
# Start all services with GPU support (Nvidia)
python start_services.py --profile gpu-nvidia

# Start with AMD GPU support (Linux only)
python start_services.py --profile gpu-amd  

# Start with CPU only
python start_services.py --profile cpu

# Start with external Ollama (Mac users)
python start_services.py --profile none

# Start in public environment (closes all ports except 80/443)
python start_services.py --profile gpu-nvidia --environment public
```

### Managing Docker Services
```bash
# Stop all services
docker compose -p localai --profile <profile> down

# View logs for a specific service
docker logs <container-name>

# Update containers to latest versions
docker compose -p localai -f docker-compose.yml --profile <profile> pull

# Restart a specific service
docker compose -p localai restart <service-name>

# View running containers
docker ps --filter "label=com.docker.compose.project=localai"
```

### Service URLs (Local Development)
- n8n: http://localhost:5678
- Open WebUI: http://localhost:8080
- Flowise: http://localhost:3001 (if exposed)
- Supabase Studio: http://localhost:54323
- Ollama: http://localhost:11434
- Neo4j: http://localhost:7474
- Qdrant: http://localhost:6333
- SearXNG: http://localhost:8081
- Langfuse: http://localhost:3000

## Architecture Overview

This project is a Docker Compose-based AI development environment that combines multiple services:

**Core Services:**
- **n8n**: Low-code workflow automation platform with AI integrations
- **Supabase**: PostgreSQL database with vector store capabilities and authentication
- **Ollama**: Local LLM runtime for serving models like Llama, Qwen, etc.
- **Open WebUI**: ChatGPT-like interface for interacting with local models and n8n agents

**Additional Services:**
- **Flowise**: No/low-code AI agent builder
- **Qdrant**: High-performance vector database for RAG applications
- **Neo4j**: Graph database for knowledge graphs (GraphRAG, LightRAG)
- **SearXNG**: Privacy-focused metasearch engine
- **Langfuse**: LLM observability and monitoring
- **Caddy**: Reverse proxy with automatic HTTPS for production deployments

**Key Architecture Points:**
- All services run in a unified Docker project named "localai"
- Supabase services are loaded first via `supabase/docker/docker-compose.yml`
- The main `docker-compose.yml` includes Supabase and adds AI services
- Services communicate via Docker networking (e.g., n8n connects to Ollama at `http://ollama:11434`)
- Profile-based GPU support (gpu-nvidia, gpu-amd, cpu, none)
- Environment-based exposure (private: all ports, public: only 80/443)

## Configuration Files

- `.env`: Main configuration with secrets for all services (copy from `.env.example`)
- `docker-compose.yml`: Primary service definitions
- `docker-compose.override.*.yml`: Environment-specific overrides
- `searxng/settings.yml`: SearXNG configuration (auto-generated from settings-base.yml)
- `n8n/backup/workflows/`: Pre-configured n8n workflows for RAG agents

## Important Implementation Notes

- When connecting services, use Docker service names as hostnames (e.g., `db` for Postgres, `ollama` for Ollama)
- Supabase Postgres credentials in n8n: Host must be `db`, not `localhost`
- For Mac users running Ollama locally: Update n8n's OLLAMA_HOST to `host.docker.internal:11434`
- SearXNG requires special handling on first run (cap_drop temporarily removed by start_services.py)
- All secrets in `.env` must be generated securely using `openssl rand -hex 32` - never use example values in production
- Avoid special characters like `@` in Postgres password to prevent connection issues
- The start_services.py script handles Supabase repo cloning automatically using sparse checkout

## N8N Integration with Open WebUI

The `n8n_pipe.py` file contains a function that bridges Open WebUI with n8n workflows:
1. Install the function in Open WebUI (Workspace -> Functions)
2. Set the n8n webhook URL from your workflow's production URL
3. The function appears as a model option in Open WebUI's dropdown

## Pre-configured Workflows

The `n8n/backup/workflows/` directory contains three production-ready RAG agent workflows:
- `V1_Local_RAG_AI_Agent.json`: Basic RAG implementation with document ingestion
- `V2_Local_Supabase_RAG_AI_Agent.json`: Supabase-powered RAG with vector storage
- `V3_Local_Agentic_RAG_AI_Agent.json`: Advanced agentic RAG with tool use capabilities

## Testing and Debugging

```bash
# Test service connectivity
docker exec localai-n8n-1 ping ollama

# Check Postgres connection from n8n
docker exec localai-n8n-1 psql -h db -U postgres -d postgres -c "SELECT version();"

# Monitor resource usage
docker stats --filter "label=com.docker.compose.project=localai"

# Access service shells
docker exec -it localai-n8n-1 /bin/sh
docker exec -it localai-db-1 psql -U postgres
```

## Production Deployment

For production deployments using Caddy:
1. Set DOMAIN_NAME and ACME_EMAIL in .env
2. Use --environment public flag to restrict port exposure
3. Caddy automatically handles SSL certificates via Let's Encrypt
4. Only ports 80 and 443 are exposed externally
5. All services remain accessible internally via Docker networking