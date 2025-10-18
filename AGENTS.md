# Azure Logic Apps MCP Server - Agent Guide

Structured instructions for AI agents to configure Azure Logic Apps as MCP servers.

## Agent Workflow

When a user asks to set up an Azure Logic App as an MCP server:

1. **Gather Information** - Ask for Azure resources and preferences
2. **Execute Setup** - Run configuration steps
3. **Test** - Verify the MCP server works
4. **Guide User** - Show how to connect with VS Code/Claude

## Gather Required Information

Ask the user conversationally:

**Azure Logic App:**

"What is your Azure Logic App subscription ID, resource group, and Logic App name?"

- `SUBSCRIPTION_ID`: Azure subscription ID
- `RESOURCE_GROUP`: Resource group name
- `LOGIC_APP_NAME`: Logic App (Standard) name

**Authentication:**

"Do you want **anonymous authentication** (for development/testing) or **OAuth 2.0** (for production)?"

- Anonymous: Quick setup, no authentication
- OAuth 2.0: Secure, requires Azure AD app registration

**App Registration (OAuth only):**

If OAuth, suggest: `{LOGIC_APP_NAME}-mcp-oauth`

**Tool Creation (Optional):**

"Would you like me to create an MCP tool now?"

If yes, suggest tool name format: `{verb}-{resource}` (e.g., `get-customer`, `send-email`)

## Configuration Steps

### Anonymous Authentication Setup

```bash
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

Wait 30 seconds for restart.

### Test Configuration

```bash
HOSTNAME=$(az logicapp show --name $LOGIC_APP_NAME --resource-group $RESOURCE_GROUP --query "defaultHostName" -o tsv)

curl -X POST "https://${HOSTNAME}/api/mcp" \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "initialize",
    "params": {
      "protocolVersion": "2024-11-05",
      "capabilities": {},
      "clientInfo": {"name": "agent-test", "version": "1.0.0"}
    }
  }'
```

**Expected:** JSON response with `serverInfo` containing "Logic Apps Remote MCP Server"

## Creating MCP Tools

### Upload Workflow via Kudu

```bash
# Upload workflow.json
curl -X PUT "https://${KUDU_URL}/api/vfs/site/wwwroot/<workflow-name>/workflow.json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "If-Match: *" \
  --data-binary @<workflow-name>/workflow.json

# Restart
az logicapp restart --name $LOGIC_APP_NAME --resource-group $RESOURCE_GROUP
```

Wait 30 seconds.

### Required Workflow Structure

**CRITICAL:** Use `"kind": "Stateless"` for MCP tools.

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "contentVersion": "1.0.0.0",
    "triggers": {
      "manual": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "type": "object",
            "properties": {
              "param": {
                "type": "string",
                "title": "Display Name",
                "description": "Help text for AI and users"
              }
            },
            "required": []
          }
        }
      }
    },
    "actions": {
      "Response": {
        "type": "Response",
        "kind": "Http",
        "inputs": {
          "statusCode": 200,
          "headers": {"Content-Type": "application/json"},
          "body": {"result": "response"}
        },
        "runAfter": {}
      }
    }
  },
  "kind": "Stateless"
}
```

### Schema Guidelines

- `title`: Display name
- `description`: Help text for AI
- `enum`: Dropdown options
- `required`: Array of required params

### Complete Workflow Template

Here's a complete working example (greeting tool with multi-language support):

```json
{
  "definition": {
    "$schema": "https://schema.management.azure.com/providers/Microsoft.Logic/schemas/2016-06-01/workflowdefinition.json#",
    "contentVersion": "1.0.0.0",
    "actions": {
      "Compose_Greeting": {
        "type": "Compose",
        "inputs": "@if(equals(coalesce(triggerBody()?['language'], 'English'), 'Spanish'), concat('¡Hola, ', coalesce(triggerBody()?['name'], 'World'), '!'), if(equals(triggerBody()?['language'], 'French'), concat('Bonjour, ', coalesce(triggerBody()?['name'], 'World'), '!'), if(equals(triggerBody()?['language'], 'German'), concat('Guten Tag, ', coalesce(triggerBody()?['name'], 'World'), '!'), concat('Hello, ', coalesce(triggerBody()?['name'], 'World'), '!'))))",
        "runAfter": {}
      },
      "Response": {
        "type": "Response",
        "kind": "Http",
        "inputs": {
          "statusCode": 200,
          "headers": {
            "Content-Type": "application/json"
          },
          "body": {
            "success": true,
            "greeting": "@{outputs('Compose_Greeting')}",
            "details": {
              "language": "@{coalesce(triggerBody()?['language'], 'English')}",
              "recipient": "@{coalesce(triggerBody()?['name'], 'World')}",
              "timestamp": "@{utcNow()}"
            }
          }
        },
        "runAfter": {
          "Compose_Greeting": [
            "Succeeded"
          ]
        }
      }
    },
    "outputs": {},
    "triggers": {
      "manual": {
        "type": "Request",
        "kind": "Http",
        "inputs": {
          "schema": {
            "type": "object",
            "properties": {
              "name": {
                "type": "string",
                "title": "Name",
                "description": "Name of the person to greet. Defaults to 'World' if not provided."
              },
              "language": {
                "type": "string",
                "title": "Language",
                "description": "Language for the greeting. Supported: English, Spanish, French, German.",
                "enum": [
                  "English",
                  "Spanish",
                  "French",
                  "German"
                ]
              }
            },
            "required": []
          }
        }
      }
    }
  },
  "kind": "Stateless"
}
```

Save this as `<workflow-name>/workflow.json` and upload via Kudu.

### Tool Naming Best Practices

- Use lowercase with hyphens: `search-documents`
- Start with verb: `get-`, `list-`, `create-`, `update-`, `delete-`
- Be specific: `create-sales-order` not `create-order`
- Keep short: max 3-4 words

## OAuth 2.0 Configuration (Production)

For production deployments, configure OAuth 2.0 authentication.

### Step 1: Create Azure AD App Registration

```bash
TENANT_ID=$(az account show --query tenantId -o tsv)

APP_RESPONSE=$(az ad app create \
  --display-name "${LOGIC_APP_NAME}-mcp-oauth" \
  --sign-in-audience "AzureADMyOrg" \
  --query "{appId:appId}" -o json)

CLIENT_ID=$(echo $APP_RESPONSE | jq -r '.appId')
echo "Client ID: $CLIENT_ID"
```

### Step 2: Configure App Registration

```bash
# Set Application ID URI
az ad app update --id $CLIENT_ID --identifier-uris "api://$CLIENT_ID"

# Add user_impersonation scope
SCOPE_ID=$(uuidgen)
cat > /tmp/oauth2_permissions.json <<EOF
{
  "oauth2PermissionScopes": [{
    "adminConsentDescription": "Allow access to MCP tools on behalf of signed-in user",
    "adminConsentDisplayName": "Access MCP tools",
    "id": "$SCOPE_ID",
    "isEnabled": true,
    "type": "User",
    "userConsentDescription": "Allow access to MCP tools on your behalf",
    "userConsentDisplayName": "Access MCP tools",
    "value": "user_impersonation"
  }]
}
EOF

az ad app update --id $CLIENT_ID --set api=@/tmp/oauth2_permissions.json

# Create service principal (CRITICAL - enables user sign-in)
az ad sp create --id $CLIENT_ID

# Add VS Code redirect URIs
az ad app update --id $CLIENT_ID \
  --web-redirect-uris "http://127.0.0.1:33418" "https://vscode.dev/redirect"
```

### Step 3: Configure Easy Auth

**CRITICAL:** Must enable OAuth discovery for VS Code auto-configuration.

```bash
cat > /tmp/auth_config.json <<EOF
{
  "properties": {
    "platform": {"enabled": true},
    "globalValidation": {
      "requireAuthentication": false,
      "unauthenticatedClientAction": "AllowAnonymous"
    },
    "identityProviders": {
      "azureActiveDirectory": {
        "enabled": true,
        "registration": {
          "openIdIssuer": "https://login.microsoftonline.com/$TENANT_ID/v2.0",
          "clientId": "$CLIENT_ID"
        },
        "validation": {
          "allowedAudiences": ["api://$CLIENT_ID/"]
        },
        "login": {
          "disableWWWAuthenticate": false
        }
      }
    }
  }
}
EOF

az rest --method put \
  --url "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.Web/sites/${LOGIC_APP_NAME}/config/authsettingsV2?api-version=2021-02-01" \
  --body @/tmp/auth_config.json
```

**Key Settings:**

- `requireAuthentication: false` - Required for MCP OAuth flow
- `disableWWWAuthenticate: false` - Enables OAuth discovery (required for VS Code)
- `allowedAudiences: ["api://$CLIENT_ID/"]` - Note trailing slash

### Step 4: Update host.json

```bash
HOSTNAME=$(az logicapp show --name $LOGIC_APP_NAME --resource-group $RESOURCE_GROUP --query "defaultHostName" -o tsv)

cat > /tmp/host_oauth.json <<EOF
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
        "authentication": {"type": "oauth2"},
        "ProtectedResourceMetadata": {
          "BearerMethodsSupported": ["header"],
          "ScopesSupported": ["api://${CLIENT_ID}/user_impersonation"],
          "Resource": "https://${HOSTNAME}/api/mcp",
          "AuthorizationServers": ["https://login.microsoftonline.com/${TENANT_ID}/v2.0"]
        }
      }
    }
  }
}
EOF

curl -X PUT "https://${KUDU_URL}/api/vfs/site/wwwroot/host.json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -H "If-Match: *" \
  --data-binary @/tmp/host_oauth.json
```

### Step 5: Restart and Verify

```bash
az logicapp restart --name $LOGIC_APP_NAME --resource-group $RESOURCE_GROUP
```

Wait 30 seconds.

### Step 6: Verify OAuth Discovery

```bash
curl -i https://${HOSTNAME}/api/mcp
```

**Expected:** `WWW-Authenticate` header with OAuth metadata URL. This enables VS Code auto-discovery.

### Provide OAuth Info to User

```text
MCP Server URL: https://<hostname>/api/mcp
Client ID: <client-id>
Tenant ID: <tenant-id>
Scope: api://<client-id>/user_impersonation

VS Code 1.102+: Just add the URL - OAuth is auto-discovered!
```

## Guide User to Connect

### VS Code (Recommended)

1. Command Palette → "MCP: Add Server"
2. Enter: `https://{logic-app-hostname}/api/mcp`
3. (OAuth) VS Code auto-discovers settings and prompts sign-in
4. Tools appear automatically

### Claude Desktop

**Anonymous:**

```json
{
  "mcpServers": {
    "logic-apps": {
      "url": "https://{hostname}/api/mcp",
      "transport": "streamable-http"
    }
  }
}
```

**OAuth:** Claude Desktop should auto-discover OAuth via WWW-Authenticate header.

### MCP Inspector

```bash
npx @modelcontextprotocol/inspector
# URL: https://{hostname}/api/mcp
# Transport: Streamable HTTP
```

## Error Handling

### "InvalidFlowExtensionRequestRoute"

**Cause:** MCP endpoints not enabled

**Fix:**

1. Verify API version is `2021-02-01`
2. Re-upload `host.json`
3. Restart Logic App, wait 30 seconds

### "DirectApiInvalidAuthorizationScheme"

**Cause:** Authentication misconfiguration

**Fix:**

1. Verify `authentication` is inside `McpServerEndpoints`
2. Check `type` is `"anonymous"` or `"oauth2"`
3. Re-upload `host.json`, restart

### OAuth: AADSTS650052

**Cause:** Service principal not created

**Fix:**

```bash
az ad sp create --id {CLIENT_ID}
```

### OAuth: Discovery not working

**Cause:** `disableWWWAuthenticate: true`

**Fix:**

Verify Easy Auth has `disableWWWAuthenticate: false`

```bash
az rest --method get \
  --url "/subscriptions/${SUBSCRIPTION_ID}/resourceGroups/${RESOURCE_GROUP}/providers/Microsoft.Web/sites/${LOGIC_APP_NAME}/config/authsettingsV2?api-version=2021-02-01" \
  --query "properties.identityProviders.azureActiveDirectory.login.disableWWWAuthenticate"
```

Should return `false`.

## Agent Checklist

When setting up MCP server:

- ✅ Ask user for all required information interactively
- ✅ Suggest good names (app registrations, tools)
- ✅ Execute all configuration steps
- ✅ Verify setup works (test initialize)
- ✅ Guide user to connect (VS Code/Claude)
- ✅ Provide clear OAuth info if applicable

## Resources

- [Microsoft Docs: MCP Server Setup](https://learn.microsoft.com/en-us/azure/logic-apps/set-up-model-context-protocol-server-standard)
- [MCP Specification](https://modelcontextprotocol.io/)
- [MCP Inspector](https://github.com/modelcontextprotocol/inspector)
