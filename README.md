# Open WebUI Docker Setup

This is a Docker Compose setup for [Open WebUI](https://github.com/open-webui/open-webui), a feature-rich, self-hosted WebUI for LLMs with support for Ollama and OpenAI-compatible APIs.

## Features

- Self-hosted web interface for LLM interactions
- Ollama integration for local models
- OAuth/SSO authentication with Authentik
- RAG (Retrieval Augmented Generation) support
- Code interpreter with Python execution
- Docker named volumes for persistent storage

## Prerequisites

- Docker and Docker Compose installed
- Ollama running (locally or accessible via network)
- (Optional) Authentik instance for OAuth/SSO

## Quick Start

### 1. Configure Environment Variables

Copy the `.env` file and update the required values:

```bash
# Generate a secure secret key
WEBUI_SECRET_KEY=$(openssl rand -base64 32)
```

**Required configurations:**
- `WEBUI_SECRET_KEY`: Generate using `openssl rand -base64 32`
- `WEBUI_URL`: Your installation URL (e.g., `http://localhost:3000`)

**Optional OAuth/SSO configurations:**
- `OAUTH_CLIENT_ID`: OAuth client ID from Authentik
- `OAUTH_CLIENT_SECRET`: OAuth client secret from Authentik
- `OPENID_PROVIDER_URL`: Your Authentik OpenID provider URL
- `OPENID_REDIRECT_URI`: Redirect URI (typically `{WEBUI_URL}/oauth/authentik/callback`)

### 2. Start the Services

```bash
docker compose up -d
```

### 3. Access the Application

Open your browser and navigate to:
```
http://localhost:3000
```

(Or the port specified in `WEBUI_PORT`)

## Configuration

### Environment Variables

All configuration is managed through the `.env` file. Key variables include:

| Variable | Description | Default |
|----------|-------------|---------|
| `WEBUI_PORT` | Host port to expose | `3000` |
| `WEBUI_VERSION` | Docker image tag | `main` |
| `OLLAMA_BASE_URL` | Ollama API endpoint | `http://host.docker.internal:11434` |
| `ENABLE_SIGNUP` | Allow user registration | `false` |
| `DEFAULT_USER_ROLE` | Default role for new users | `pending` |
| `ENABLE_CODE_INTERPRETER` | Enable Python code execution | `true` |
| `RAG_EMBEDDING_MODEL` | Model for embeddings | `nomic-embed-text` |
| `RAG_FILE_MAX_SIZE` | Max upload size in MB | `10` |
| `WEBUI_SESSION_COOKIE_SECURE` | Enable secure cookies (HTTPS) | `true` |
| `DATA_VOLUME_PATH` | Custom data directory path (optional) | _(empty = named volume)_ |
| `WORKSPACE_VOLUME_PATH` | Custom workspace directory path (optional) | _(empty = named volume)_ |

See the [official documentation](https://docs.openwebui.com/getting-started/env-configuration) for all available options.

### Volumes

The setup supports two volume configuration methods:

#### Option 1: Named Volumes (Default - Recommended)

Docker named volumes are used by default for portability across systems:

- `open-webui-data`: Application data, uploads, cache
- `open-webui-workspace`: User workspace files
- `/tmp`: Temporary directory (for code interpreter)

**Advantages:**
- Portable across different operating systems
- Managed by Docker (easier backups with Docker tools)
- No permission issues

**Backup named volumes:**
```bash
# Backup data volume
docker run --rm -v open-webui-data:/data -v $(pwd):/backup alpine tar czf /backup/open-webui-data-backup.tar.gz -C /data .

# Backup workspace volume
docker run --rm -v open-webui-workspace:/data -v $(pwd):/backup alpine tar czf /backup/open-webui-workspace-backup.tar.gz -C /data .
```

**Restore named volumes:**
```bash
# Restore data volume
docker run --rm -v open-webui-data:/data -v $(pwd):/backup alpine tar xzf /backup/open-webui-data-backup.tar.gz -C /data

# Restore workspace volume
docker run --rm -v open-webui-workspace:/data -v $(pwd):/backup alpine tar xzf /backup/open-webui-workspace-backup.tar.gz -C /data
```

#### Option 2: Bind Mounts (Custom Paths)

If you prefer to store data in specific directories on your host:

1. **Set paths in `.env` file:**
   ```env
   # Windows (use forward slashes)
   DATA_VOLUME_PATH=C:/Docker/open-webui/data
   WORKSPACE_VOLUME_PATH=C:/Docker/open-webui/workspace

   # Linux
   DATA_VOLUME_PATH=/home/user/Docker/open-webui/data
   WORKSPACE_VOLUME_PATH=/home/user/Docker/open-webui/workspace
   ```

2. **Create directories:**
   ```bash
   # Linux/Mac
   mkdir -p /home/user/Docker/open-webui/data
   mkdir -p /home/user/Docker/open-webui/workspace

   # Windows (PowerShell)
   New-Item -ItemType Directory -Path "C:\Docker\open-webui\data" -Force
   New-Item -ItemType Directory -Path "C:\Docker\open-webui\workspace" -Force
   ```

3. **Edit `compose.yaml`:**
   - Comment out the named volume lines (lines 10-11)
   - Uncomment the bind mount lines (lines 13-14)

   ```yaml
   volumes:
     # Named volumes (default - portable across systems)
     # - open-webui-data:/app/backend/data
     # - open-webui-workspace:/app/backend/workspace
     # Bind mounts (uncomment and comment above if using custom paths from .env)
     - ${DATA_VOLUME_PATH}:/app/backend/data
     - ${WORKSPACE_VOLUME_PATH}:/app/backend/workspace
     # Temporary directory mount (for code interpreter and temp operations)
     - /tmp:/tmp
   ```

4. **Restart the container:**
   ```bash
   docker compose down
   docker compose up -d
   ```

**Advantages:**
- Data stored in known, accessible locations
- Easy to backup with standard tools (rsync, robocopy, etc.)
- Can share data across multiple containers easily

**Backup bind mounts:**
```bash
# Linux/Mac
tar czf open-webui-backup.tar.gz /home/user/Docker/open-webui/

# Windows (PowerShell with 7-Zip)
7z a open-webui-backup.7z "C:\Docker\open-webui\"
```

## OAuth/SSO Setup (Authentik)

1. Create an OAuth2/OpenID provider in Authentik
2. Configure the redirect URI: `{WEBUI_URL}/oauth/authentik/callback`
3. Copy the Client ID and Client Secret to your `.env` file
4. Set the OpenID provider URL (e.g., `https://auth.example.com/application/o/open-webui/`)
5. Restart the container: `docker compose restart`

## Connecting to Ollama

### Ollama on Host Machine

The default configuration uses `host.docker.internal:11434` to connect to Ollama running on the host:

```env
OLLAMA_BASE_URL=http://host.docker.internal:11434
```

### Ollama in Docker

If running Ollama in a separate container on the same network:

```env
OLLAMA_BASE_URL=http://ollama:11434
```

Then uncomment the `depends_on` section in `compose.yaml`.

## Management Commands

### View Logs
```bash
docker compose logs -f
```

### Restart Services
```bash
docker compose restart
```

### Stop Services
```bash
docker compose down
```

### Stop and Remove Volumes (CAUTION: Deletes all data)
```bash
docker compose down -v
```

### Update to Latest Version
```bash
docker compose pull
docker compose up -d
```

## Security Considerations

1. **Always use a strong `WEBUI_SECRET_KEY`** - Generate with `openssl rand -base64 32`
2. **Keep `.env` file secure** - Never commit it to version control
3. **Use HTTPS in production** - Set `WEBUI_SESSION_COOKIE_SECURE=true`
4. **Review OAuth settings** - Ensure redirect URIs match your deployment
5. **Restrict user signup** - Set `ENABLE_SIGNUP=false` unless needed
6. **Use pending role** - Set `DEFAULT_USER_ROLE=pending` to review new users

## Troubleshooting

### Cannot connect to Ollama

**Problem**: "Failed to connect to Ollama"

**Solution**:
- Verify Ollama is running: `ollama list`
- Check the `OLLAMA_BASE_URL` in `.env`
- On Windows/Mac, use `http://host.docker.internal:11434`
- Test connectivity: `docker compose exec open-webui curl http://host.docker.internal:11434/api/tags`

### OAuth authentication fails

**Problem**: OAuth redirect errors

**Solution**:
- Verify `WEBUI_URL` matches your actual URL
- Check redirect URI in Authentik matches `{WEBUI_URL}/oauth/authentik/callback`
- Ensure all OAuth variables are set in `.env`
- Check logs: `docker compose logs open-webui`

### Permission denied on volumes

**Problem**: Cannot write to volumes

**Solution**:
- Ensure Docker has proper permissions
- On Linux, check volume ownership
- Try recreating volumes: `docker compose down -v && docker compose up -d`

## Documentation

- [Open WebUI Documentation](https://docs.openwebui.com/)
- [Environment Configuration](https://docs.openwebui.com/getting-started/env-configuration)
- [GitHub Repository](https://github.com/open-webui/open-webui)

## License

Open WebUI is licensed under the MIT License. See the [official repository](https://github.com/open-webui/open-webui) for details.
