# Azure OpenAI Integration

This document describes how to use the Azure OpenAI backend with Mini Claude Code.

## Overview

Azure OpenAI integration adds a third backend option alongside Anthropic and OpenAI, using OAuth authentication with Cisco's identity service and Azure's OpenAI endpoint.

## Features

- **OAuth Authentication**: Automatic token fetching using client credentials
- **Azure-Specific Configuration**: Custom endpoint, API version, and appkey support
- **Full Tool Support**: All existing tools work with Azure backend
- **Streaming Support**: Real-time response streaming like other backends
- **Sub-Agent Support**: Fork mode for skills and sub-agents

## Configuration

### Environment Variables

Set these environment variables to use Azure OpenAI:

```bash
# Required: OAuth credentials
export AZURE_CLIENT_ID="your-client-id"
export AZURE_CLIENT_SECRET="your-client-secret"
export AZURE_APPKEY="your-appkey"

# Optional: Override defaults
export AZURE_OPENAI_ENDPOINT="https://chat-ai.cisco.com"  # default
export AZURE_API_VERSION="2025-04-01-preview"             # default
```

### Command Line Arguments

Alternatively, pass configuration via command line:

```bash
mini-claude-py --azure \
  --azure-client-id "your-client-id" \
  --azure-client-secret "your-client-secret" \
  --model "gpt-5-nano" \
  "your prompt here"
```

## Usage Examples

### Basic Usage
#### python venv
4.1 cd python && python3 -m venv .venv
4.2 source .venv/bin/activate
4.3 .venv/bin/pip install -e .  : install libs
#### uv venv
4.1 cd python && uv sync		: install libs
4.2 source .venv/bin/activate

```bash
# Set environment variables
export AZURE_CLIENT_ID="your-client-id"
export AZURE_CLIENT_SECRET="your-client-secret"
export AZURE_APPKEY="your-appkey"

# Run with Azure
mini-claude-py --azure --model "gpt-5-nano" "Hello, Azure!"
```
```bash
# deepseek
export ANTHROPIC_API_KEY=sk-244...
export ANTHROPIC_BASE_URL=https://api.deepseek.com/anthropic
export MINI_CLAUDE_MODEL=deepseek-v4-pro[1m]
```

### Interactive REPL

```bash
# Start interactive mode with Azure
mini-claude-py --azure --model "gpt-5-nano"
```

### With Permission Modes

```bash
# Plan mode (read-only)
mini-claude-py --azure --plan --model "gpt-5-nano" "analyze this codebase"

# YOLO mode (skip confirmations)
mini-claude-py --azure --yolo --model "gpt-5-nano" "run all tests"
```

### Custom Endpoint and Version

```bash
mini-claude-py --azure \
  --azure-endpoint "https://your-custom-endpoint.azure.com" \
  --azure-api-version "2024-12-01-preview" \
  --model "gpt-4o" \
  "your prompt"
```

## How It Works

1. **Token Fetching**: When Azure mode is enabled, the agent automatically fetches an OAuth token from Cisco's identity service using the provided client credentials.

2. **Client Initialization**: The `AsyncAzureOpenAI` client is initialized with:
   - Azure endpoint URL
   - API version
   - OAuth access token

3. **Request Format**: Requests use OpenAI's message format with an additional `user` field containing the appkey:
   ```json
   {
     "appkey": "..."
   }
   ```

4. **Message History**: Azure uses the same OpenAI message format, so message history is stored in `_openai_messages`.

## Implementation Details

### Key Files Modified

1. **`agent.py`**:
   - Added `_get_azure_access_token()` function for OAuth
   - Added `use_azure` flag and Azure-specific parameters to `Agent.__init__()`
   - Added `_ensure_azure_client()` method for lazy initialization
   - Added `_chat_azure()` and `_call_azure_stream()` methods
   - Updated all `use_openai` checks to include `use_azure`

2. **`__main__.py`**:
   - Added Azure command line arguments
   - Added Azure configuration resolution
   - Updated help text with Azure examples

### OAuth Token Flow

```python
async def _get_azure_access_token(client_id: str, client_secret: str) -> str:
    url = "https://id.cisco.com/oauth2/default/v1/token"
    # Base64 encode credentials
    # POST request with grant_type=client_credentials
    # Extract access_token from response
    return api_key
```

### Client Initialization

```python
self._azure_client = AsyncAzureOpenAI(
    azure_endpoint=self._azure_endpoint,
    api_key=api_key,  # OAuth token
    api_version=self._azure_api_version,
)
```

## Differences from Other Backends

| Feature | Anthropic | OpenAI | Azure |
|---------|-----------|--------|-------|
| Authentication | API Key | API Key | OAuth Token |
| Message Format | Anthropic | OpenAI | OpenAI |
| User Field | N/A | Optional | Required (appkey) |
| Streaming | ✅ | ✅ | ✅ |
| Tools | ✅ | ✅ | ✅ |
| Extended Thinking | ✅ | ❌ | ❌ |

## Troubleshooting

### Token Fetch Failure

If you get an error about failed token fetching:
- Verify `AZURE_CLIENT_ID` and `AZURE_CLIENT_SECRET` are correct
- Check network connectivity to `https://id.cisco.com`
- Ensure credentials have not expired

### Model Not Found

If the model is not available:
- Verify the model name matches Azure's deployment
- Check with your Azure admin for available models

### Connection Issues

If you get connection errors:
- Verify `AZURE_OPENAI_ENDPOINT` is correct
- Check if the endpoint is accessible from your network
- Try using the default endpoint first

## Security Considerations

- **Never hardcode credentials**: Use environment variables or command line arguments
- **Rotate secrets regularly**: OAuth client secrets should be rotated periodically
- **Secure storage**: Store credentials in secure secret managers in production
- **Token lifetime**: Access tokens expire and are automatically refreshed on each agent initialization

## Comparison with Demo Code

The integration is based on the `chat.py` demo, with these enhancements:

1. **Full Tool Support**: Unlike the demo's simple chat, this supports all 8 tools
2. **Streaming**: Real-time response streaming
3. **Permission System**: 5 permission modes for safety
4. **Context Management**: 4-layer compression for long conversations
5. **Sub-Agents**: Fork mode for complex tasks
6. **Session Persistence**: Save and resume conversations

## Future Enhancements

Possible improvements:

1. **Token Caching**: Cache OAuth tokens until expiration
2. **Multi-Region**: Support multiple Azure regions
3. **Custom Auth**: Support different OAuth providers
4. **Retry Logic**: Enhanced retry for token refresh failures
