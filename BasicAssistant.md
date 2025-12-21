#Configure the Azure Prompt Flow Model

This step connects your assistant to Azure OpenAI and your knowledge base through a Retrieval-Augmented Generation (RAG) configuration.

1. Open **[Pulse Studio](https://cfs-client.lithium.primaverabss.com/)**
2. Navigate to **Models** → **Create**
3. Select **AzurePromptFlow** as the provider
4. Configure the endpoints with your Azure resources:

```json
{
  "Model": "gpt-4o",
  "ModelEndpoint": "https://your-openai-endpoint.openai.azure.com/",
  "ModelApiKey":    "Key-Value-inside-Pulse-Core-Team-Azure-Key-Vault",
  "AiSearchEndpoint": "https://your-search-service.search.windows.net",
  "AiSearchIndexName": "your-knowledge-base-index",
  "AiSearchApiKey":    "Key-Value-inside-Pulse-Core-Team-Azure-Key-Vault"
}
```

**Share your Azure OpenAI model key and index credentials with the Pulse Core team**
