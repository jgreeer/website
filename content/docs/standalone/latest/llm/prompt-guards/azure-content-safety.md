---
title: Azure AI Content Safety
weight: 25
description: Apply Azure AI Content Safety to filter LLM requests and responses for harmful content, blocklist violations, and jailbreak attempts.
---

[Azure AI Content Safety](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview) provides content filtering across categories (Hate, SelfHarm, Sexual, Violence), custom blocklist matching, and jailbreak detection. You configure a Content Safety resource in Azure and apply it to LLM traffic passing through the agentgateway proxy. When a request or response violates a content safety policy, agentgateway blocks the interaction and returns an error.

Azure AI Content Safety is model-agnostic and can be applied to any Large Language Model (LLM), whether it is hosted on Azure, another cloud provider (like AWS or Google), or on-premises.

## Before you begin

1. {{< reuse "agw-docs/snippets/prereq-agentgateway.md" >}}
2. Create an Azure Content Safety resource in the [Azure portal](https://portal.azure.com/#create/Microsoft.CognitiveServicesContentSafety) or via the Azure CLI:
   ```sh
   az cognitiveservices account create \
     --name <resource-name> \
     --resource-group <resource-group> \
     --kind ContentSafety \
     --sku S0 \
     --location <location>
   ```

3. Retrieve your resource endpoint hostname:
   ```sh
   az cognitiveservices account show \
     --name <resource-name> \
     --resource-group <resource-group> \
     --query "properties.endpoint" -o tsv
   ```

4. Authenticate with Azure using the standard [Azure credential chain](https://learn.microsoft.com/en-us/azure/developer/go/azure-sdk-authentication). By default, agentgateway uses the implicit Azure credential chain (managed identity, environment variables, or Azure CLI credentials). No explicit secret is required if your environment is already authenticated.

## Configure Azure AI Content Safety

Configure the `guardrails` field under `llm.models[]` in your agentgateway configuration. You can apply Azure AI Content Safety to the `request` phase, the `response` phase, or both.

```yaml
cat <<'EOF' > config.yaml
# yaml-language-server: $schema=https://agentgateway.dev/schema/config
llm:
  models:
  - name: "*"
    provider: openAI
    params:
      model: gpt-4o-mini
      apiKey: "$OPENAI_API_KEY"
    guardrails:
      request:
      - azureContentSafety:
          endpoint: your-resource-name.cognitiveservices.azure.com
          analyzeText:
            severityThreshold: 2
          detectJailbreak: {}
      response:
      - azureContentSafety:
          endpoint: your-resource-name.cognitiveservices.azure.com
          analyzeText:
            severityThreshold: 2
EOF
```

| Setting | Description |
| -- | -- |
| `endpoint` | **(Required)** The Azure Content Safety endpoint hostname, such as `my-resource.cognitiveservices.azure.com`. |
| `analyzeText` | Configuration for the [Analyze Text API](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/quickstart-text). Detects harmful content categories and blocklist matches. |
| `analyzeText.severityThreshold` | Severity threshold (0–6). Content at or above this level is blocked. Default: `2`. |
| `analyzeText.apiVersion` | API version to use. Default: `2024-09-01`. |
| `analyzeText.blocklistNames` | List of custom blocklist names to check against. |
| `analyzeText.haltOnBlocklistHit` | When `true`, further analysis stops if a blocklist is hit. |
| `detectJailbreak` | Configuration for the [Jailbreak Detection API](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/quickstart-jailbreak). Only applicable to request guards. Use `{}` for defaults. |
| `detectJailbreak.apiVersion` | API version to use. Default: `2024-02-15-preview`. |

When a request or response matches a content safety policy, agentgateway blocks the interaction and returns an error such as: `The request was rejected due to inappropriate content`.

## Test the guardrail

Send a request containing harmful content to verify the guard blocks it:

```sh
curl http://localhost:4000/v1beta/openai/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
    "model": "",
    "messages": [
      {
        "role": "user",
        "content": "I want to harm someone"
      }
    ]
  }'
```

Expected output:
```console
The request was rejected due to inappropriate content
```
