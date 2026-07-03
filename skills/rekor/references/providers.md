# Provider Adapters reference

Import tool definitions from any LLM provider as record_types, or export record_types as tool definitions. Supported providers: `openai`, `anthropic`, `google`, `mcp`. The SKILL.md **Provider Adapters** section covers the concept; read this file for the command forms.

## Import (creates record_types from tool definitions)

```bash
# OpenAI
rekor providers import openai --base <ws> --tools '[{"type":"function","function":{"name":"create_invoice","parameters":{"type":"object","properties":{"customer":{"type":"string"},"amount":{"type":"number"}},"required":["customer"]}}}]'

# Anthropic
rekor providers import anthropic --base <ws> --tools '[{"name":"create_invoice","input_schema":{"type":"object","properties":{"customer":{"type":"string"}}}}]'

# MCP
rekor providers import mcp --base <ws> --tools '[{"name":"create_invoice","inputSchema":{"type":"object","properties":{"customer":{"type":"string"}}}}]'

# From file
rekor providers import openai --base <ws> --tools @tools.json
```

## Export (get record_types as tool definitions)

```bash
rekor providers export openai --base <ws>
rekor providers export anthropic --base <ws> --record_types invoices,customers
rekor providers export mcp --base <ws> --output tools.json
```

## Import tool call (create a document from provider-native format)

```bash
# OpenAI tool call → document
rekor providers import-call openai invoices --base <ws> \
  --data '{"arguments":{"customer":"Acme","amount":5000}}' \
  --external-id call_abc123 --external-source openai

# Anthropic tool call → document
rekor providers import-call anthropic invoices --base <ws> \
  --data '{"input":{"customer":"Acme","amount":5000}}'
```
