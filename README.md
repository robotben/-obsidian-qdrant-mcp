# obsidian-qdrant-mcp

A custom MCP (Model Context Protocol) server that provides semantic search over an Obsidian vault using Qdrant vector database and Ollama embeddings.

## Overview

This server bridges claude.ai (or any MCP-compatible client) to a Qdrant vector store containing embedded Obsidian vault notes. It embeds search queries using the same Ollama model used during ingestion (`nomic-embed-text`) to ensure embedding alignment, then returns semantically similar note chunks with file paths and scores.

## Architecture

```
claude.ai → Cloudflare tunnel → obsidian-qdrant-mcp → Ollama (nomic-embed-text)
                                                      → Qdrant (obsidian collection)
```

## Tools

| Tool | Description |
|------|-------------|
| `search_vault` | Semantic search across all vault notes using natural language |
| `find_related_notes` | Given text content, find conceptually related notes |
| `search_vault_by_tag` | Find notes related to a specific tag or topic |
| `vault_stats` | Show collection statistics (vector count, model, config) |

## Requirements

- Python 3.12+
- Running Qdrant instance (with an `obsidian` collection pre-indexed)
- Running Ollama instance with `nomic-embed-text` model pulled
- Docker (for containerized deployment)

## Configuration

All configuration is via environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `OLLAMA_BASE_URL` | `http://192.168.245.62:11434` | Ollama API URL |
| `EMBED_MODEL` | `nomic-embed-text` | Embedding model name |
| `QDRANT_HOST` | `192.168.245.187` | Qdrant host |
| `QDRANT_PORT` | `6333` | Qdrant port |
| `COLLECTION_NAME` | `obsidian` | Qdrant collection name |
| `DEFAULT_TOP_K` | `5` | Default number of results |
| `MCP_PORT` | `3000` | Port to serve MCP on |

## Deployment

### Docker Compose (recommended)

Add to your existing `docker-compose.yml`:

```yaml
  qdrant-mcp:
    build: ./qdrant-mcp
    container_name: qdrant-mcp
    restart: unless-stopped
    ports:
      - "3011:3000"
    networks:
      - n8n-network
```

Then build and start:
```bash
docker compose build qdrant-mcp
docker compose up -d qdrant-mcp
docker logs qdrant-mcp -f
```

### Standalone

```bash
pip install -r requirements.txt
OLLAMA_BASE_URL=http://your-ollama:11434 \
QDRANT_HOST=your-qdrant \
python server.py
```

## Vault Ingestion

This server requires the vault to be pre-indexed into Qdrant. Use the companion ingestion script:

```bash
# Full import
python obsidian_ingest.py

# Incremental (modified files only)
python obsidian_ingest.py --incremental

# Weekly full reset (via cron)
python obsidian_ingest.py --reset
```

See [obsidian_ingest.py](../Scripts/obsidian_ingest.py) in the vault for the full ingestion pipeline.

## Connecting to claude.ai

1. Expose the server via Cloudflare tunnel (or any HTTPS reverse proxy)
2. Go to **Settings → Integrations** in claude.ai
3. Add the MCP URL: `https://qdrant-mcp.yourdomain.com/sse`

Note: The endpoint is `/sse` (Server-Sent Events transport via FastMCP).

## Notes

- The server uses SSE transport via [FastMCP](https://github.com/jlowin/fastmcp)
- Embedding alignment is critical — the same `nomic-embed-text` model must be used for both ingestion and query-time embedding
- Ollama may take 10–30 seconds to warm up the model on first request after idle
