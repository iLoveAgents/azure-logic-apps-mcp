# Azure Logic Apps MCP Server

Turn Azure Logic Apps workflows into tools that AI assistants can use through the Model Context Protocol (MCP).

## Let AI Agents Do the Work

**Don't read this whole README!** Just download [AGENTS.md](./AGENTS.md) and give it to an AI agent.

Then say: *"Set up my Azure Logic App as an MCP server"*

The agent will handle everything:

- Ask for your Azure resources
- Configure MCP endpoints
- Set up authentication (dev or production)
- Create your first tool
- Connect with VS Code or Claude

**No dependencies. No scripts. Just AGENTS.md and an AI agent.**

---

*Still want to do it manually? Continue reading...*

## Manual Setup (If You Prefer)

### Prerequisites

- Azure subscription
- Azure Logic App (Standard) - [Create one](https://learn.microsoft.com/en-us/azure/logic-apps/create-single-tenant-workflows-azure-portal)
- Azure CLI - [Install guide](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)

### 1. Enable MCP on Your Logic App

```bash
# Set your values
export SUBSCRIPTION_ID="your-subscription-id"
export RESOURCE_GROUP="your-resource-group"
export LOGIC_APP_NAME="your-logic-app-name"

# Login and get token
az login
TOKEN=$(az account get-access-token --resource https://management.azure.com --query accessToken -o tsv)

# Get Kudu URL
KUDU_URL=$(az logicapp show \
  --name $LOGIC_APP_NAME \
  --resource-group $RESOURCE_GROUP \
  --query "enabledHostNames[?contains(@, 'scm')]|[0]" \
  -o tsv)

# Create host.json
cat > host.json <<'EOF'
{
  "version": "2.0",
  "extensionBundle": {
    "id": "Microsoft.Azure.Functions.ExtensionBundle.Workflows",
    "version": "[1.*, 2.0.0)"
  },
  "extensions": {
    "workflow": {
      "McpServerEndpoints": {
        "enable": true,
        "authentication": {
          "type": "anonymous"
        }
      }
    }
  }
}
EOF

# Upload and restart
curl -X PUT "https://${KUDU_URL}/api/vfs/site/wwwroot/host.json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "If-Match: *" \
  --data-binary @host.json

az logicapp restart --name $LOGIC_APP_NAME --resource-group $RESOURCE_GROUP
```

Wait 30 seconds for the restart.

### 2. Create Your First Tool

See [AGENTS.md](./AGENTS.md) for the complete workflow template and upload instructions.

### 3. Connect to AI Assistants

**VS Code (Recommended):**

1. Command Palette → "MCP: Add Server"
2. Enter: `https://your-logic-app-name.azurewebsites.net/api/mcp`
3. Your tools appear automatically

**Claude Desktop:**

```json
{
  "mcpServers": {
    "logic-apps": {
      "url": "https://your-logic-app-name.azurewebsites.net/api/mcp",
      "transport": "streamable-http"
    }
  }
}
```

**MCP Inspector:**

```bash
npx @modelcontextprotocol/inspector
# URL: https://your-logic-app-name.azurewebsites.net/api/mcp
# Transport: Streamable HTTP
```

## Creating Tools

Each tool is a **stateless** or **Stateful** workflow with:

1. **HTTP Request Trigger** (named "manual") with JSON schema
2. **Business Logic** (any workflow actions)
3. **HTTP Response** (200 status, JSON body)

For complete workflow templates and examples, see [AGENTS.md](./AGENTS.md).

## Use Cases

- **Data Integration**: Query databases, APIs, Azure services
- **Business Automation**: Trigger workflows, send notifications
- **Data Transformation**: Convert formats, process data
- **Custom Actions**: Execute specialized business logic

## Production Security

The quick start uses **anonymous authentication** for development/testing.

**For production**, configure OAuth 2.0 authentication:

- Industry-standard security
- Azure AD integration
- User-based access control
- Scope-based permissions

See **[AGENTS.md](./AGENTS.md)** for complete OAuth 2.0 setup instructions with:

- Azure AD app registration
- Easy Auth configuration
- VS Code auto-discovery support
- Troubleshooting guide

**Note:** VS Code 1.102+ auto-discovers OAuth settings - just add the server URL and sign in!

## Troubleshooting

**Tool not appearing:**

- Verify workflow is in `<tool-name>/workflow.json`
- Check `"kind": "Stateless"` is set
- Restart Logic App and wait 30 seconds

**Authentication errors:**

- Verify `authentication` is inside `McpServerEndpoints` in host.json
- Restart Logic App after configuration changes

**For detailed troubleshooting and automated deployment, see [AGENTS.md](./AGENTS.md)**

## Resources

- [Azure Logic Apps Docs](https://learn.microsoft.com/en-us/azure/logic-apps/)
- [MCP Specification](https://modelcontextprotocol.io/)
- [MCP Server Setup Guide](https://learn.microsoft.com/en-us/azure/logic-apps/set-up-model-context-protocol-server-standard)
- [JSON Schema Docs](https://json-schema.org/)

## License

MIT
