# LLM Pricing & Benchmarks Open Dataset

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
[![Dataset License: CC BY 4.0](https://img.shields.io/badge/Dataset-CC%20BY%204.0-lightgrey.svg)](https://creativecommons.org/licenses/by/4.0/)
[![Model Count](https://img.shields.io/badge/Models-23%20Frontier-brightgreen.svg)](#dataset-schema)
[![Website: modelpricing.xyz](https://img.shields.io/badge/Website-modelpricing.xyz-indigo.svg)](https://modelpricing.xyz)

Standardized, schema-validated dataset of frontier foundation model API pricing, KV prompt caching discounts, context windows, output limits, and verified coding benchmark scores. Curated and maintained by [ModelPricing.xyz](https://modelpricing.xyz).

Visit the live interactive token budget calculator and prompt caching simulator at **[https://modelpricing.xyz](https://modelpricing.xyz)**.

---

## Overview

Modern LLM token economics can no longer be assessed using simple nominal per-million token rates. Real-world API expenditure is heavily influenced by:
- **Prompt Caching Economics**: Caching discounts of up to 90% (e.g., Anthropic Claude 3.5/4.5 Sonnet, DeepSeek V3/R1, Google Gemini implicit caching).
- **Reasoning Token Overhead**: Chain-of-thought tokens (e.g., OpenAI o3-mini, DeepSeek R1) billed at full completion token rates.
- **Context Boundaries**: Context windows extending from 64k to 2M+ tokens.

This repository provides developers, AI researchers, and cloud finops engineers with machine-readable, schema-validated datasets in JSON and CSV formats.

---

## Dataset Files

- **`data/models.json`**: Complete array of 23 foundation model specifications conforming to Draft-07 JSON Schema.
- **`data/models.csv`**: Synchronized tabular CSV export suitable for spreadsheet, SQL, or Pandas analysis.
- **`schemas/catalog.schema.json`**: Official JSON Schema specification validating the models catalog array.
- **`schemas/model.schema.json`**: Official JSON Schema specification validating individual model objects.
- **`.github/workflows/sync-pricing.yml`**: Continuous integration workflow validating schema compliance and JSON/CSV parity.

---

## Dataset Schema

| Field | Type | Required | Description | Example |
|---|---|---|---|---|
| `id` | `string` | **Yes** | Unique slug identifier (kebab-case) | `"claude-4-5-sonnet"` |
| `name` | `string` | **Yes** | Official foundation model display name | `"Claude 4.5 Sonnet"` |
| `provider` | `string` | **Yes** | Model vendor (`OpenAI`, `Anthropic`, `Google`, `DeepSeek`, `Meta`, `xAI`, `Mistral`, `Alibaba`) | `"Anthropic"` |
| `category` | `string` | **Yes** | Architectural tier (`Flagship`, `Reasoning`, `Fast & Cheap`, `Coding`) | `"Reasoning"` |
| `inputPricePerMillion` | `number` | **Yes** | Nominal price per 1,000,000 un-cached prompt tokens in USD | `3.00` |
| `outputPricePerMillion` | `number` | **Yes** | Nominal price per 1,000,000 generated completion tokens in USD | `15.00` |
| `cachedInputPricePerMillion` | `number \| null` | No | Discounted price per 1,000,000 prompt-cached tokens in USD | `0.30` |
| `contextWindow` | `integer` | **Yes** | Maximum total context window in tokens | `200000` |
| `maxOutputTokens` | `integer` | **Yes** | Maximum completion token generation limit per call | `64000` |
| `officialSourceUrl` | `string` | No | Canonical link to vendor documentation or pricing card | `"https://www.anthropic.com/pricing"` |
| `mmluScore` | `number \| null` | No | Standardized MMLU benchmark score (0.0 to 100.0) | `91.8` |
| `codingBenchmark` | `string \| null` | No | Verified SWE-bench or HumanEval score | `"SWE-bench: 76.5%"` |
| `description` | `string` | **Yes** | Concise architectural summary and performance highlights | `"Anthropic’s benchmark-leading hybrid reasoning model..."` |
| `bestFor` | `string` | **Yes** | Recommended production workloads and deployment targets | `"Agentic software engineering, large-scale code synthesis..."` |

---

## Code Usage Examples

### 1. Python (requests & pandas)

Fetch the live dataset and load directly into a Pandas DataFrame for cost modeling:

```python
import pandas as pd
import requests

# Fetch JSON directly from GitHub Raw or local repository
url = "https://raw.githubusercontent.com/modelpricing/llm-pricing-benchmarks/main/data/models.json"
response = requests.get(url)
data = response.json()

# Load into pandas DataFrame
df = pd.DataFrame(data)

# Compute 10M token blended cost (7M prompt tokens with 60% cache hit + 3M output tokens)
def calc_blended_10m(row):
    cached_rate = row['cachedInputPricePerMillion'] if pd.notnull(row['cachedInputPricePerMillion']) else row['inputPricePerMillion']
    prompt_cost = (7 * 0.40 * row['inputPricePerMillion']) + (7 * 0.60 * cached_rate)
    output_cost = 3 * row['outputPricePerMillion']
    return prompt_cost + output_cost

df['blended_10m_usd'] = df.apply(calc_blended_10m, axis=1)

# Display top 5 cost-efficient frontier models
print(df[['name', 'provider', 'category', 'blended_10m_usd']].sort_values('blended_10m_usd').head(5))
```

### 2. TypeScript / JavaScript (Node.js)

Load and query the catalog locally or over HTTPS:

```typescript
import fs from 'node:fs';
import path from 'node:path';

interface AIModel {
  id: string;
  name: string;
  provider: string;
  category: string;
  inputPricePerMillion: number;
  outputPricePerMillion: number;
  cachedInputPricePerMillion: number | null;
  contextWindow: number;
  maxOutputTokens: number;
  officialSourceUrl?: string;
  mmluScore?: number | null;
  codingBenchmark?: string | null;
  description: string;
  bestFor: string;
}

// Read local dataset file
const filePath = path.resolve('data/models.json');
const models: AIModel[] = JSON.parse(fs.readFileSync(filePath, 'utf8'));

// Filter for reasoning models supporting prompt caching
const reasoningModels = models
  .filter((m) => m.category === 'Reasoning' && m.cachedInputPricePerMillion !== null)
  .map((m) => ({
    name: m.name,
    provider: m.provider,
    cacheSavingsPercent: Math.round(
      (1 - m.cachedInputPricePerMillion! / m.inputPricePerMillion) * 100
    ),
  }));

console.log('Reasoning Models with Prompt Caching Discounts:');
console.table(reasoningModels);
```

### 3. cURL & jq

Query models on the command line using cURL and jq:

```bash
# Query all models with input prices under $1.00 per million tokens
curl -s https://raw.githubusercontent.com/modelpricing/llm-pricing-benchmarks/main/data/models.json \
  | jq '.[] | select(.inputPricePerMillion < 1.00) | {id: .id, name: .name, input: .inputPricePerMillion, output: .outputPricePerMillion}'

# Extract CSV directly
curl -s https://raw.githubusercontent.com/modelpricing/llm-pricing-benchmarks/main/data/models.csv | head -n 10
```

---

## Schema Validation & CI

All pull requests and data updates are verified using JSON Schema Draft-07 via `ajv-cli`:

```bash
# Install dependencies
npm install

# Validate catalog against JSON Schemas
npm run validate

# Run all schema validations
npm run validate:all

# Run integration tests
npm test
```

---

## Licensing

- **Code, Schema, and Automation**: Licensed under the **[MIT License](LICENSE)**.
- **Dataset Content & Aggregations**: Licensed under the **[Creative Commons Attribution 4.0 International (CC BY 4.0)](https://creativecommons.org/licenses/by/4.0/)** license.
  - When utilizing this dataset in research, applications, or articles, please provide attribution to [ModelPricing.xyz](https://modelpricing.xyz).
