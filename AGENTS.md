This file provides guidance to AI assistants when working with code in this repository.

## Project Overview

This is a Model Context Protocol (MCP) server that integrates Monarch Money personal finance platform with Claude Desktop. Built on the unofficial [MonarchMoney Python library](https://github.com/hammem/monarchmoney) by @hammem with full MFA support.

## Development Commands

### Setup and Installation
```bash
# Install dependencies
pip install -r requirements.txt
pip install -e .

# Set up authentication (one-time setup)
python login_setup.py
```

### Testing the Server
```bash
# Run the server directly
python src/monarch_mcp_server/server.py

# Or via uv (as configured in Claude Desktop)
uv run --with mcp[cli] --with-editable . mcp run src/monarch_mcp_server/server.py
```

### Code Quality Tools (available in dev dependencies)
```bash
# Format code
black src/

# Sort imports
isort src/

# Type checking
mypy src/

# Run tests (when implemented)
pytest
```

## Architecture

### Core Components

1. **server.py** - Main MCP server implementation
   - Uses FastMCP framework for tool registration
   - Implements `run_async()` pattern to bridge sync/async calls (MCP tools are sync, MonarchMoney API is async)
   - Thread-safe with ThreadPoolExecutor and isolated event loops per request

2. **secure_session.py** - Session management via system keyring
   - Stores authentication tokens securely using `keyring` library
   - Service ID: `com.mcp.monarch-mcp-server`
   - Avoids insecure pickle files or JSON credentials
   - Auto-cleanup of legacy session files

3. **login_setup.py** - Interactive authentication script
   - Runs outside Claude Desktop for security
   - Handles MFA/2FA flow automatically via `interactive_login()`
   - Saves session token to keyring for persistent access

### Authentication Flow

1. User runs `login_setup.py` in terminal (once)
2. Script authenticates with Monarch Money (supports MFA)
3. Session token saved to system keyring
4. MCP server loads token from keyring on startup
5. Token persists for weeks/months

### Async/Sync Bridge Pattern

MCP tools are synchronous, but MonarchMoney API is async. The `run_async()` helper function:
- Creates new thread with isolated event loop
- Runs async coroutine to completion
- Returns result to sync context
- Example: `run_async(_get_accounts())`

### Available MCP Tools

All tools follow this pattern: sync wrapper → `run_async()` → async client call → JSON response

- `setup_authentication()` - Returns setup instructions
- `check_auth_status()` - Check keyring for stored token
- `get_accounts()` - List all financial accounts
- `get_transactions()` - Fetch transactions with date/account filtering
- `get_budgets()` - Budget data with spent/remaining amounts
- `get_cashflow()` - Income/expense analysis over date ranges
- `get_account_holdings()` - Investment holdings for specific accounts
- `create_transaction()` - Add new manual transactions
- `update_transaction()` - Modify existing transactions
- `refresh_accounts()` - Request real-time account refresh

## Date Handling

- All dates use `YYYY-MM-DD` format
- Transaction amounts: positive = income, negative = expenses
- Date parameters are optional in most tools

## Configuration

### Claude Desktop Integration
Add to `~/Library/Application Support/Claude/claude_desktop_config.json` (macOS):
```json
{
  "mcpServers": {
    "Monarch Money": {
      "command": "/opt/homebrew/bin/uv",
      "args": [
        "run",
        "--with",
        "mcp[cli]",
        "--with-editable",
        "/path/to/monarch-mcp-server",
        "mcp",
        "run",
        "/path/to/monarch-mcp-server/src/monarch_mcp_server/server.py"
      ]
    }
  }
}
```

### Environment Variables (Optional)
If not using keyring, fallback to:
- `MONARCH_EMAIL` - Monarch Money email
- `MONARCH_PASSWORD` - Monarch Money password

## Dependencies

Key libraries:
- `mcp[cli]>=1.0.0` - Model Context Protocol framework
- `monarchmoney>=0.1.15` - Unofficial Monarch Money API client
- `keyring>=24.0.0` - Secure credential storage
- `pydantic>=2.0.0` - Data validation
- `gql>=3.4,<4.0` - GraphQL client (used by monarchmoney)

## Error Handling

Common patterns:
- Authentication errors → User must run `login_setup.py`
- Session expired → Re-run `login_setup.py` to refresh token
- API errors → Logged with detailed context, returned as JSON error strings
- All tool functions return strings (JSON or error messages)

## Security Notes

- Never commit session files or credentials
- Tokens stored in OS-level keyring (Keychain on macOS)
- Authentication happens outside Claude Desktop in secure terminal
- MFA/2FA fully supported through interactive flow
