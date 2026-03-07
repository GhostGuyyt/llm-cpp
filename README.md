# llm-cpp

llm-cpp is a suite of 12 single-header C++17 libraries for integrating large language models into native applications. Each library is a self-contained `.hpp` file — drop in what you need, define one implementation macro, and ship. No Python, no SDKs, no package manager required.

---

## The Suite

| Library | Description | Deps |
|---------|-------------|------|
| **[llm-stream](https://github.com/Mattbusel/llm-stream)** | Stream OpenAI & Anthropic responses via SSE | libcurl |
| **[llm-cache](https://github.com/Mattbusel/llm-cache)** | LRU response cache — skip identical API calls | None |
| **[llm-cost](https://github.com/Mattbusel/llm-cost)** | Token counting + cost estimation for 6 models | None |
| **[llm-retry](https://github.com/Mattbusel/llm-retry)** | Retry with exponential backoff + circuit breaker | None |
| **[llm-format](https://github.com/Mattbusel/llm-format)** | JSON schema enforcement + structured output | None |
| **[llm-embed](https://github.com/Mattbusel/llm-embed)** | Text embeddings + cosine similarity + vector store | libcurl |
| **[llm-pool](https://github.com/Mattbusel/llm-pool)** | Concurrent request pool with priority queue + rate limiting | None |
| **[llm-log](https://github.com/Mattbusel/llm-log)** | Structured JSONL logging for every LLM call | None |
| **[llm-template](https://github.com/Mattbusel/llm-template)** | Mustache-style prompt templating | None |
| **[llm-agent](https://github.com/Mattbusel/llm-agent)** | Tool-calling agent loop (OpenAI function calling) | libcurl |
| **[llm-rag](https://github.com/Mattbusel/llm-rag)** | Retrieval-augmented generation pipeline | libcurl |
| **[llm-eval](https://github.com/Mattbusel/llm-eval)** | N-run evaluation + consistency scoring + model comparison | libcurl |

---

## Quickstart

Libraries compose naturally. Here is a production-ready pattern using llm-log, llm-retry, and llm-stream together:

```cpp
// Log every call to JSONL, retry on 429/5xx, stream tokens to stdout

#define LLM_LOG_IMPLEMENTATION
#include "llm_log.hpp"

#define LLM_RETRY_IMPLEMENTATION
#include "llm_retry.hpp"

#define LLM_STREAM_IMPLEMENTATION
#include "llm_stream.hpp"

int main() {
    llm::Logger logger("calls.jsonl");

    llm::Config cfg;
    cfg.api_key = std::getenv("OPENAI_API_KEY");
    cfg.model   = "gpt-4o-mini";

    const std::string prompt = "Explain backpressure in one paragraph.";

    // Log the outgoing request
    auto log_id = logger.log_request(prompt, cfg.model);

    // Retry up to 3 times with exponential backoff on transient errors
    auto result = llm::with_retry<std::string>([&]() -> std::string {
        std::string output;

        llm::stream_openai(prompt, cfg,
            [&](std::string_view tok) {
                std::cout << tok << std::flush;
                output += tok;
            },
            [](const llm::StreamStats& s) {
                std::cout << "\n[" << s.token_count << " tokens, "
                          << s.tokens_per_sec << " tok/s]\n";
            }
        );

        return output;
    });

    // Log the response
    logger.log_response(log_id, result);
}
```

---

## Installation

Each library is a single `.hpp` file. Copy what you need:

```bash
# Network libraries (require libcurl)
curl -O https://raw.githubusercontent.com/Mattbusel/llm-stream/main/include/llm_stream.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-embed/main/include/llm_embed.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-agent/main/include/llm_agent.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-rag/main/include/llm_rag.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-eval/main/include/llm_eval.hpp

# Pure C++17 — zero external deps
curl -O https://raw.githubusercontent.com/Mattbusel/llm-cache/main/include/llm_cache.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-cost/main/include/llm_cost.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-retry/main/include/llm_retry.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-format/main/include/llm_format.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-pool/main/include/llm_pool.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-log/main/include/llm_log.hpp
curl -O https://raw.githubusercontent.com/Mattbusel/llm-template/main/include/llm_template.hpp
```

In exactly **one** `.cpp` file per library, define the implementation macro before including:

```cpp
#define LLM_LOG_IMPLEMENTATION
#define LLM_RETRY_IMPLEMENTATION
#define LLM_STREAM_IMPLEMENTATION
#include "llm_log.hpp"
#include "llm_retry.hpp"
#include "llm_stream.hpp"
```

All other translation units just `#include` without the macro.

---

## Requirements

| Requirement | Detail |
|-------------|--------|
| C++ standard | C++17 or later |
| Compiler | GCC, Clang, MSVC — all supported |
| External deps | libcurl for llm-stream, llm-embed, llm-agent, llm-rag, and llm-eval. All others: zero deps. |
| Build system | Any. Works with CMake, Make, Bazel, MSVC, plain `g++`. |

---

## License

All 12 libraries: MIT — Copyright (c) 2026 Mattbusel.
