# Exploring TOON: An Experimental Approach to Token Optimization

 token-efficient data formats for LLMs.


## The Token Efficiency Problem 

### Why Token Efficiency Matters

Every token you send to an LLM costs money. Modern LLM pricing (input tokens):
- Claude Sonnet 4: $3 per 1M tokens
- GPT-4o: $2.50 per 1M tokens
- Gemini 1.5 Pro: $1.25 per 1M tokens

**Scenario:** Sending 100 records for analysis at 50K calls/day

| Format | Tokens/call | Monthly tokens | Cost at $3/1M |
|--------|-------------|----------------|---------------|
| Standard JSON | 4,545 | 6.8 billion | $20,400 |
| Minified JSON | ~3,200 | 4.8 billion | $14,400 |
| TOON | 2,744 | 4.1 billion | $12,300 |

Token optimization can meaningfully reduce costs at scale.


### The JSON Problem

```json
{
  "users": [
    {"id": 1, "name": "Alice", "role": "admin"},
    {"id": 2, "name": "Bob", "role": "user"},
    {"id": 3, "name": "Charlie", "role": "user"}
  ]
}
```

**Inefficiencies:**
- Keys repeated: `id`, `name`, `role` appear 3 times each (9 repetitions)
- Structural tokens: `{}` `[]` `""` `:` `,` 
- **Total: 84 tokens for 3 simple records**


##  How TOON Works 

### TOON's Core Idea

**Declare schema once, stream values only**

```
users[3]{id,name,role}:
  1,Alice,admin
  2,Bob,user
  3,Charlie,user
```

**Result: 32 tokens — 62% reduction**

**Key innovation:** Combines YAML's indentation with CSV's tabular efficiency

### Three Array Formats

| Data Shape | TOON Format | Example |
|------------|-------------|---------|
| **Primitives** | Inline | `tags[3]: reading,gaming,coding` |
| **Uniform objects** | Tabular | `users[2]{id,name}:` + rows |
| **Mixed/nested** | List | `items[2]:` + `- item` lines |

**Decision logic:**
- All elements primitives? → Inline
- All objects, same fields, primitive values? → Tabular *(this is where big savings happen)*
- Everything else? → List (YAML-style)

### Real Benchmark - Token Savings

From official TOON repository:

| Dataset | JSON Tokens | TOON Tokens | Reduction |
|---------|-------------|-------------|-----------|
| 100 records | 4,545 | 2,744 | 39.6% |
| 60 days analytics | 10,977 | 4,507 | 58.9% |
| 100 GitHub repos | 15,145 | 8,745 | 42.3% |

**Pattern:** Uniform data structures = maximum savings


##  Hands-On Exercise 

### Setup

**Use the online playground** (zero setup):
- https://toon-playground.pages.dev/

Or install locally:
```bash
# TypeScript
npm install @toon-format/toon

# Python (beta, from GitHub)
git clone https://github.com/toon-format/toon-python.git
```


### EXERCISE: E-commerce Orders

### Convert This to TOON

```json
{
  "orders": [
    {"orderId": "ORD-001", "customer": "Alice", "total": 259.99, "status": "shipped"},
    {"orderId": "ORD-002", "customer": "Bob", "total": 89.50, "status": "pending"},
    {"orderId": "ORD-003", "customer": "Charlie", "total": 450.00, "status": "delivered"},
    {"orderId": "ORD-004", "customer": "Diana", "total": 125.75, "status": "shipped"}
  ]
}
```

**Analysis question:** What format will TOON use? Why?

### Solution

```
orders[4]{orderId,customer,total,status}:
  ORD-001,Alice,259.99,shipped
  ORD-002,Bob,89.5,pending
  ORD-003,Charlie,450,delivered
  ORD-004,Diana,125.75,shipped
```

**Token comparison:**
- JSON (minified): ~145 tokens
- TOON: ~75 tokens
- **48% reduction**



##  The Critiques and Limitations 

### The Accuracy Question

**TOON repository claims:**
```
Data extraction tasks (1000 tests, 4 models):
TOON:           73.9% accuracy
JSON compact:   70.7% accuracy
JSON standard:  69.7% accuracy
```

**Independent testing (LLM-Format-Benchmarks):**
```
Tabular data (GPT-4.1 nano):
Markdown-Table: 51.9% accuracy (25,140 tokens)
TOON:           47.5% accuracy (21,518 tokens)
CSV:            44.3% accuracy (19,524 tokens)

Nested data (GPT-5 nano):
YAML:           62.1% accuracy (42,477 tokens)
JSON:           50.3% accuracy (57,933 tokens)
TOON:           43.1% accuracy (45,436 tokens)
```

**The problem:** These results directly contradict each other.

### The Training Data Problem

**Reality check:**
- LLMs are trained on billions of tokens of JSON, YAML, XML, CSV
- TOON barely exists in public training data
- Models haven't learned TOON's structure as deeply as JSON
- Future models trained on TOON might handle it better

**The catch-22:** TOON needs to be used to get into training data, but it's risky to use before it's in training data.

### Simpler Alternatives

**Alternative 1: Minified JSON**
```json
{"orders":[{"orderId":"ORD-001","customer":"Alice","total":259.99},...]}
```
- Just remove whitespace
- ~30% token reduction vs pretty-printed JSON
- Zero learning curve, perfect model understanding
- Prompt: "Use compact JSON format" (4 tokens)

**Alternative 2: CSV (for pure tabular data)**
```
orderId,customer,total,status
ORD-001,Alice,259.99,shipped
...
```
- ~10% more efficient than TOON
- Widely understood by models
- Loses structure for nested data

**TOON's value proposition:** Falls between minified JSON and CSV. Worth it if you need structure + efficiency and your model handles it well.

### Known Limitations

**When NOT to use TOON:**

1. **Deeply nested configurations** (complex hierarchies)
   - TOON's indentation adds overhead
   - JSON may be more efficient

2. **Non-uniform data** (varying fields per item)
   - Falls back to list format
   - Minimal savings over JSON

3. **Models not trained on TOON** (most current models)
   - May parse incorrectly
   - Risk of lower accuracy

4. **Latency-critical applications**
   - Encoding adds ~1-2ms overhead
   - Some models parse JSON faster

**The honest assessment:** TOON is experimental. It has promise, but it's not proven across all models and use cases.

##  Key Takeaways 

### Decision Framework

**Should you use TOON?**

```
START HERE
│
├─ Do you send large uniform arrays to LLMs?
│  └─ NO → Stick with minified JSON
│
├─ Are you operating at scale (>10K calls/day)?
│  └─ NO → Token savings won't justify complexity
│
├─ Can you benchmark with your specific model?
│  └─ NO → Don't risk accuracy issues
│
├─ Did your benchmarks show good accuracy?
│  └─ NO → Use minified JSON instead
│
└─ YES to all above → Consider TOON
   (But monitor accuracy in production)
```

### Three Rules for Token Optimization

**Rule 1: Start simple**
- Minified JSON gets you 30% savings immediately
- No new format to learn, no accuracy risk

**Rule 2: Benchmark before adopting**
- Test with your actual model (not just any model)
- Measure both token reduction AND accuracy
- Use production-like data

**Rule 3: Understand the tradeoffs**
- Token prices are dropping ($30/1M → $3/1M in 18 months)
- Context windows are growing (4K → 200K+)
- The problem TOON solves is becoming less critical
- Weigh token savings against implementation complexity

### Resources

**Official TOON:**
- Spec: https://toonformat.dev
- Playground: https://toon-playground.pages.dev/
- TypeScript: `npm install @toon-format/toon`
- Python: https://github.com/toon-format/toon-python

**Independent benchmarks:**
- LLM Format Benchmarks: https://llm-format-benchmarks.com

**Alternatives:**
- Minified JSON: Just use `JSON.stringify(data)` (no spaces)
- CSV: For pure tabular data
