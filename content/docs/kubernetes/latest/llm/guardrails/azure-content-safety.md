---
title: Azure AI Content Safety
weight: 25
description: Apply Azure AI Content Safety to filter LLM requests and responses for harmful content, blocklist violations, and jailbreak attempts.
---

[Azure AI Content Safety](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/overview) provides content filtering across categories (Hate, SelfHarm, Sexual, Violence), custom blocklist matching, and jailbreak detection. You configure a Content Safety resource in Azure and apply it to LLM traffic. When a request or response violates a content safety policy, the agentgateway proxy blocks the interaction and returns an error.

Azure AI Content Safety is model-agnostic and can be applied to any Large Language Model (LLM), whether it is hosted on Azure, another cloud provider (like AWS or Google), or on-premises.

## Before you begin

{{< reuse "agw-docs/snippets/agw-prereq-llm.md" >}}

## Set up Azure AI Content Safety

1. Create an Azure Content Safety resource in the [Azure portal](https://portal.azure.com/#create/Microsoft.CognitiveServicesContentSafety) or via the Azure CLI:
   ```sh
   az cognitiveservices account create \
     --name <resource-name> \
     --resource-group <resource-group> \
     --kind ContentSafety \
     --sku S0 \
     --location <location>
   ```

2. Retrieve your resource endpoint hostname:
   ```sh
   az cognitiveservices account show \
     --name <resource-name> \
     --resource-group <resource-group> \
     --query "properties.endpoint" -o tsv
   ```

3. Configure the prompt guard. Add the endpoint of your Azure Content Safety resource.
   ```yaml
   kubectl apply -f - <<EOF
   apiVersion: {{< reuse "agw-docs/snippets/trafficpolicy-apiversion.md" >}}
   kind: {{< reuse "agw-docs/snippets/trafficpolicy.md" >}}
   metadata:
     name: azure-content-safety-guard
     namespace: {{< reuse "agw-docs/snippets/namespace.md" >}}
   spec:
     targetRefs:
     - group: gateway.networking.k8s.io
       kind: HTTPRoute
       name: openai
     backend:
       ai:
         promptGuard:
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
   | `analyzeText.blocklistNames` | List of custom blocklist names to check against. |
   | `analyzeText.haltOnBlocklistHit` | When `true`, further analysis stops if a blocklist is hit. |
   | `detectJailbreak` | Configuration for the [Jailbreak Detection API](https://learn.microsoft.com/en-us/azure/ai-services/content-safety/quickstart-jailbreak). Only applicable to request guards. Use `{}` for defaults. |

   {{< callout type="info" >}}
   By default, agentgateway uses the implicit Azure credential chain (managed identity, environment variables, or Azure CLI credentials). No explicit secret is required if your environment is already authenticated. For authentication details, see the [Azure authentication documentation](https://learn.microsoft.com/en-us/azure/developer/go/azure-sdk-authentication).
   {{< /callout >}}

4. Test the guardrail. Send a request containing harmful content to verify the guard blocks it.
   {{< tabs tabTotal="2" items="OpenAI v1/chat/completions, Custom route" >}}
   {{% tab tabName="OpenAI v1/chat/completions" %}}
   **Cloud Provider LoadBalancer**:
   ```sh
   curl "$INGRESS_GW_ADDRESS:8080/v1/chat/completions" \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $OPENAI_API_KEY" \
     -d '{
       "model": "gpt-4o-mini",
       "messages": [
         {"role": "user", "content": "I want to harm someone"}
       ]
     }'
   ```
   **Port-forward**:
   ```sh
   curl "localhost:8080/v1/chat/completions" \
     -H "Content-Type: application/json" \
     -H "Authorization: Bearer $OPENAI_API_KEY" \
     -d '{
       "model": "gpt-4o-mini",
       "messages": [
         {"role": "user", "content": "I want to harm someone"}
       ]
     }'
   ```
   {{% /tab %}}
   {{% tab tabName="Custom route" %}}
   **Cloud Provider LoadBalancer**:
   ```sh
   curl "$INGRESS_GW_ADDRESS:8080/openai" \
     -H "Content-Type: application/json" \
     -d '{
       "model": "gpt-4o-mini",
       "messages": [
         {"role": "user", "content": "I want to harm someone"}
       ]
     }'
   ```
   **Port-forward**:
   ```sh
   curl "localhost:8080/openai" \
     -H "Content-Type: application/json" \
     -d '{
       "model": "gpt-4o-mini",
       "messages": [
         {"role": "user", "content": "I want to harm someone"}
       ]
     }'
   ```
   {{% /tab %}}
   {{< /tabs >}}

   Expected output:
   ```console
   The request was rejected due to inappropriate content
   ```
