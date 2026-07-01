# Provider Adapters reference

Import tool definitions from any LLM provider as collections, or export collections as tool definitions. Supported providers: `openai`, `anthropic`, `google`, `mcp`. The SKILL.md **Provider Adapters** section covers the concept; read this file for the command forms.

## Import (creates collections from tool definitions)

```bash
# OpenAI
rekor providers import openai --database <ws> --tools '[{"type":"function","function":{"name":"create_invoice","parameters":{"type":"object","properties":{"customer":{"type":"string"},"amount":{"type":"number"}},"required":["customer"]}}}]'

# Anthropic
rekor providers import anthropic --database <ws> --tools '[{"name":"create_invoice","input_schema":{"type":"object","properties":{"customer":{"type":"string"}}}}]'

# MCP
rekor providers import mcp --database <ws> --tools '[{"name":"create_invoice","inputSchema":{"type":"object","properties":{"customer":{"type":"string"}}}}]'

# From file
rekor providers import openai --database <ws> --tools @tools.json
```

## Export (get collections as tool definitions)

```bash
rekor providers export openai --database <ws>
rekor providers export anthropic --database <ws> --collections invoices,customers
rekor providers export mcp --database <ws> --output tools.json
```

## Import tool call (create a document from provider-native format)

```bash
# OpenAI tool call → document
rekor providers import-call openai invoices --database <ws> \
  --data '{"arguments":{"customer":"Acme","amount":5000}}' \
  --external-id call_abc123 --external-source openai

# Anthropic tool call → document
rekor providers import-call anthropic invoices --database <ws> \
  --data '{"input":{"customer":"Acme","amount":5000}}'
```
