# llm-cpp

The C++ LLM toolkit. Five single-header libraries. Drop in what you need.

No Python. No SDKs. No package manager. Just copy a `.hpp` file and ship.

---

## The Suite

| Library | What it does | Get it |
|---------|-------------|--------|
| **[llm-stream](https://github.com/Mattbusel/llm-stream)** | Stream tokens from OpenAI and Anthropic APIs in real time. Callback-based, thread-safe, RAII curl handles. | `llm_stream.hpp` |
| **[llm-cache](https://github.com/Mattbusel/llm-cache)** | Thread-safe LRU response cache with TTL expiry. Wraps any LLM call — cache hits skip the API entirely. | `llm_cache.hpp` |
| **[llm-cost](https://github.com/Mattbusel/llm-cost)** | Token counting and cost estimation for 6 built-in models. Compare costs across models before you call. | `llm_cost.hpp` |
| **[llm-retry](https://github.com/Mattbusel/llm-retry)** | Exponential backoff retry with jitter + circuit breaker. Handles 429s, 5xxs, and network errors automatically. | `llm_retry.hpp` |
| **[llm-format](https://github.com/Mattbusel/llm-format)** | Schema-enforced structured output. Retries with a correction prompt until the model returns valid JSON. Ships with a full JSON parser. | `llm_format.hpp` |

---

## Quickstart

Each library is independent. Use one, use all five, combine them:

```cpp
// 1. Estimate cost before calling
#define LLM_COST_IMPLEMENTATION
#include "llm_cost.hpp"

auto tc = llm::count("Your prompt here", llm::models::GPT4O_MINI);
std::cout << llm::format_cost(tc.estimated_cost_usd) << "\n";

// 2. Cache + stream — only call the API on a miss
#define LLM_CACHE_IMPLEMENTATION
#include "llm_cache.hpp"

llm::ResponseCache cache;
std::string response = cache.get_or_compute(prompt, [&]() {
    // your streaming or blocking LLM call here
    return call_openai(prompt);
});

// 3. Retry on rate limits
#define LLM_RETRY_IMPLEMENTATION
#include "llm_retry.hpp"

auto result = llm::with_retry<std::string>([&]() {
    return call_openai(prompt);   // throws LLMError on 429/5xx
});

// 4. Enforce structured output
#define LLM_FORMAT_IMPLEMENTATION
#include "llm_format.hpp"

llm::Schema schema;
schema.name   = "PersonInfo";
schema.fields = {{"name","string",true}, {"age","number",true}};

auto fmt = llm::enforce_schema(prompt, schema, [&](const std::string& p) {
    return call_openai(p);
});
if (fmt.valid) std::cout << fmt.value["name"].as_string() << "\n";

// 5. Stream directly
#define LLM_STREAM_IMPLEMENTATION
#include "llm_stream.hpp"

llm::Config cfg;
cfg.api_key = std::getenv("OPENAI_API_KEY");
cfg.model   = "gpt-4o-mini";

llm::stream_openai("Explain recursion.", cfg,
    [](std::string_view tok) { std::cout << tok << std::flush; },
    [](const llm::StreamStats& s) {
        std::cout << "\n[" << s.token_count << " tokens, " << s.tokens_per_sec << " tok/s]\n";
    }
);
```

---

## Installation

Each library is a single `.hpp` file. Copy what you need:

```bash
# Pick and choose — or grab all five
curl -O https://raw.githubusercontent.com/Mattbusel/llm-stream/main/include/llm_stream.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-cache/main/include/llm_cache.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-cost/main/include/llm_cost.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-retry/main/include/llm_retry.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-format/main/include/llm_format.hpp
```

In exactly **one** `.cpp` file per library, define the implementation macro before including:

```cpp
#define LLM_STREAM_IMPLEMENTATION   // enables llm_stream.hpp implementation
#define LLM_CACHE_IMPLEMENTATION    // enables llm_cache.hpp implementation
// ... etc
#include "llm_stream.hpp"
#include "llm_cache.hpp"
```

All other files just `#include` without the macro.

---

## Requirements

| Requirement | Detail |
|-------------|--------|
| C++ standard | C++17 or later |
| Compiler | GCC, Clang, MSVC — all supported |
| External deps | `libcurl` for **llm-stream** only. All others: zero deps. |
| Build system | Any. Works with CMake, Make, Bazel, MSVC, plain `g++`. |

---

## Why C++ for LLM calls?

- **No runtime to ship.** Embed LLM features in a game engine, CLI tool, robotics controller, or server binary without dragging in a Python interpreter.
- **No build system changes.** Copy one file. Add `-lcurl` for streaming. Done.
- **Works where Python doesn't.** WASM targets, embedded Linux, size-constrained binaries, latency-sensitive services.
- **Zero abstraction overhead.** These headers add no latency to your hot path. The token counter runs in microseconds. The cache is a mutex + hashmap.

---

## Library Details

### [llm-stream](https://github.com/Mattbusel/llm-stream)

Streams SSE responses from OpenAI (`/v1/chat/completions`) and Anthropic (`/v1/messages`). Parses the `data:` event stream, calls your `on_token` callback for each fragment, and reports throughput stats on completion. Auto-detects provider from model name prefix.

**Deps:** libcurl (ships on macOS + most Linux; `vcpkg install curl` on Windows)

---

### [llm-cache](https://github.com/Mattbusel/llm-cache)

LRU cache backed by a `std::list` + `std::unordered_map` for O(1) get/put. Thread-safe via `std::mutex`. Configurable capacity, TTL, and case-sensitivity. `get_or_compute()` wraps any LLM call — call it identically to a direct API call, get caching for free. Includes `prompt_hash()` (FNV-1a 64-bit) for content-addressed keys.

**Deps:** none

---

### [llm-cost](https://github.com/Mattbusel/llm-cost)

Estimates token counts using a cl100k_base heuristic (±5% vs tiktoken). Prices requests against 6 built-in models. `compare_costs()` returns all models sorted cheapest-first. `assert_budget()` throws before you make an expensive API call by mistake. `format_cost()` renders `$0.0042` or `0.42¢` automatically.

**Deps:** none

---

### [llm-retry](https://github.com/Mattbusel/llm-retry)

`with_retry<T>()` wraps any callable: catches `LLMError`, checks the HTTP code against your retry list, sleeps with exponential backoff + jitter, re-throws on exhaustion. `with_failover<T>()` adds a fallback callable. `CircuitBreaker` tracks failure rate, opens after N consecutive failures, transitions to half-open after a timeout, closes on recovery. Thread-safe.

**Deps:** none

---

### [llm-format](https://github.com/Mattbusel/llm-format)

Ships a complete hand-rolled recursive descent JSON parser (`parse_json`) and serializer (`to_json`). `enforce_schema()` calls your `llm_fn`, strips markdown fences, validates the JSON against your `Schema`, and retries with a structured correction prompt on failure. Works with any API — `llm_fn` is just a `std::function<std::string(const std::string&)>`.

**Deps:** none

---

## License

All five libraries: MIT — Copyright (c) 2026 Mattbusel.
