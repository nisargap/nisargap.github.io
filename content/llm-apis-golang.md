+++
title = "Calling LLM APIs in Go: A Server-Side Guide"
date = 2026-02-28
description = "A practical walkthrough of integrating OpenAI and other LLM APIs into Go server applications, covering streaming, tool use, retries, and production concerns."
[taxonomies]
tags = ["golang", "llm", "openai", "api", "backend", "ai"]
categories = ["Engineering"]
+++

Go is an excellent language for building LLM-powered backends. Its concurrency model, strong standard library, and predictable performance make it a natural fit for the kind of long-running, I/O-heavy workloads that LLM inference involves. But the ecosystem is younger than Python's, and most tutorials assume you're writing a Jupyter notebook, not a production service.

This post covers what you actually need to know to call LLM APIs from a Go server — from basic request/response to streaming, tool use, error handling, and production concerns.

<!-- more -->

## Setting Up: The OpenAI Go SDK

The official OpenAI Go SDK is maintained by OpenAI and is the cleanest starting point. Install it with:

```bash
go get github.com/openai/openai-go
```

The SDK wraps the REST API and handles authentication, serialization, and most of the boilerplate. It also works with any OpenAI-compatible API endpoint (Azure OpenAI, Groq, Together AI, Ollama, etc.) by overriding the base URL.

Create a client:

```go
package main

import (
    "github.com/openai/openai-go"
    "github.com/openai/openai-go/option"
)

func newClient() *openai.Client {
    return openai.NewClient(
        option.WithAPIKey(os.Getenv("OPENAI_API_KEY")),
    )
}
```

Store your key in an environment variable — never hardcode it or commit it.

## Basic Chat Completion

The core primitive is the chat completion endpoint. You pass a list of messages and get a response back:

```go
package main

import (
    "context"
    "fmt"
    "os"

    "github.com/openai/openai-go"
    "github.com/openai/openai-go/option"
)

func main() {
    client := openai.NewClient(option.WithAPIKey(os.Getenv("OPENAI_API_KEY")))

    completion, err := client.Chat.Completions.New(context.Background(), openai.ChatCompletionNewParams{
        Model: openai.F(openai.ChatModelGPT4o),
        Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
            openai.SystemMessage("You are a helpful assistant."),
            openai.UserMessage("What is the capital of France?"),
        }),
    })
    if err != nil {
        panic(err)
    }

    fmt.Println(completion.Choices[0].Message.Content)
}
```

The SDK uses a typed `F()` wrapper for optional fields, which keeps the API explicit about what's being set versus what's left as the zero value. It takes a little getting used to but removes an entire class of subtle bugs.

## Streaming Responses

For user-facing applications, you almost always want to stream the response rather than wait for the full completion. A `gpt-4o` response can take several seconds — streaming lets you start rendering immediately.

```go
func streamCompletion(ctx context.Context, client *openai.Client, userPrompt string) error {
    stream := client.Chat.Completions.NewStreaming(ctx, openai.ChatCompletionNewParams{
        Model: openai.F(openai.ChatModelGPT4o),
        Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
            openai.UserMessage(userPrompt),
        }),
    })

    for stream.Next() {
        chunk := stream.Current()
        if len(chunk.Choices) > 0 {
            delta := chunk.Choices[0].Delta.Content
            fmt.Print(delta) // or write to an http.ResponseWriter
        }
    }

    return stream.Err()
}
```

### Streaming Over HTTP with Server-Sent Events

When building an HTTP endpoint, stream the LLM output to the client using Server-Sent Events (SSE). This lets browsers and clients consume the stream in real time without needing WebSockets:

```go
func handleChat(w http.ResponseWriter, r *http.Request) {
    w.Header().Set("Content-Type", "text/event-stream")
    w.Header().Set("Cache-Control", "no-cache")
    w.Header().Set("Connection", "keep-alive")

    flusher, ok := w.(http.Flusher)
    if !ok {
        http.Error(w, "streaming not supported", http.StatusInternalServerError)
        return
    }

    client := openai.NewClient(option.WithAPIKey(os.Getenv("OPENAI_API_KEY")))
    stream := client.Chat.Completions.NewStreaming(r.Context(), openai.ChatCompletionNewParams{
        Model: openai.F(openai.ChatModelGPT4o),
        Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
            openai.UserMessage(r.URL.Query().Get("q")),
        }),
    })

    for stream.Next() {
        chunk := stream.Current()
        if len(chunk.Choices) > 0 {
            content := chunk.Choices[0].Delta.Content
            fmt.Fprintf(w, "data: %s\n\n", content)
            flusher.Flush()
        }
    }

    if err := stream.Err(); err != nil {
        fmt.Fprintf(w, "event: error\ndata: %s\n\n", err.Error())
        flusher.Flush()
    }

    fmt.Fprintf(w, "data: [DONE]\n\n")
    flusher.Flush()
}
```

The client context (`r.Context()`) propagates cancellation — if the user closes the browser tab, the stream stops.

## Tool Use (Function Calling)

Tool use lets the model decide when to call external functions. You define the tools, the model emits a structured call, and your code executes it and feeds the result back. It's the foundation of agent-style applications.

```go
// Define a tool
weatherTool := openai.ChatCompletionToolParam{
    Type: openai.F(openai.ChatCompletionToolTypeFunction),
    Function: openai.F(openai.FunctionDefinitionParam{
        Name:        openai.F("get_weather"),
        Description: openai.F("Get the current weather for a city"),
        Parameters: openai.F(openai.FunctionParameters{
            "type": "object",
            "properties": map[string]interface{}{
                "city": map[string]string{
                    "type":        "string",
                    "description": "The city name",
                },
            },
            "required": []string{"city"},
        }),
    }),
}

// Call the model
completion, err := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
    Model: openai.F(openai.ChatModelGPT4o),
    Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
        openai.UserMessage("What's the weather in Tokyo?"),
    }),
    Tools: openai.F([]openai.ChatCompletionToolParam{weatherTool}),
})
```

When the model wants to call a tool, it returns a message with `tool_calls`. You extract the arguments, run your function, and loop back with the result:

```go
func runAgentLoop(ctx context.Context, client *openai.Client, userMessage string) (string, error) {
    messages := []openai.ChatCompletionMessageParamUnion{
        openai.UserMessage(userMessage),
    }

    for {
        completion, err := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
            Model:    openai.F(openai.ChatModelGPT4o),
            Messages: openai.F(messages),
            Tools:    openai.F([]openai.ChatCompletionToolParam{weatherTool}),
        })
        if err != nil {
            return "", err
        }

        msg := completion.Choices[0].Message

        // No tool calls: we're done
        if len(msg.ToolCalls) == 0 {
            return msg.Content, nil
        }

        // Append the assistant message with tool calls
        messages = append(messages, msg.ToParam())

        // Execute each tool call
        for _, call := range msg.ToolCalls {
            result, err := dispatchToolCall(call)
            if err != nil {
                result = fmt.Sprintf("error: %v", err)
            }
            messages = append(messages, openai.ToolMessage(call.ID, result))
        }
        // Loop: send the tool results back to the model
    }
}
```

Keep the loop bounded — add a `maxIterations` counter to avoid infinite loops from misbehaving models.

## Error Handling and Retries

LLM APIs are remote HTTP services. They fail. Handle errors deliberately:

```go
import "github.com/openai/openai-go/apierror"

func callWithRetry(ctx context.Context, client *openai.Client, params openai.ChatCompletionNewParams) (*openai.ChatCompletion, error) {
    const maxAttempts = 3
    backoff := time.Second

    for attempt := range maxAttempts {
        result, err := client.Chat.Completions.New(ctx, params)
        if err == nil {
            return result, nil
        }

        var apiErr *apierror.Error
        if errors.As(err, &apiErr) {
            switch apiErr.StatusCode {
            case 429: // Rate limited
                time.Sleep(backoff)
                backoff *= 2
                continue
            case 500, 502, 503: // Server errors — retry
                if attempt < maxAttempts-1 {
                    time.Sleep(backoff)
                    backoff *= 2
                    continue
                }
            case 400, 401, 403, 404: // Client errors — don't retry
                return nil, err
            }
        }

        return nil, err
    }

    return nil, fmt.Errorf("max retry attempts reached")
}
```

Key error categories to handle:
- **429 Rate limit**: Back off and retry. Consider tracking your token usage to avoid hitting limits in the first place.
- **500/502/503 Server errors**: Transient — safe to retry with backoff.
- **400 Bad request**: Your input is malformed. Don't retry.
- **401/403 Auth errors**: Check your API key. Don't retry.
- **context.DeadlineExceeded**: The request timed out. Decide based on your SLA whether to retry.

## Calling Other LLM Providers

Most LLM providers expose an OpenAI-compatible API. You can use the same SDK with a different base URL:

```go
// Anthropic via their OpenAI-compatible endpoint
claudeClient := openai.NewClient(
    option.WithAPIKey(os.Getenv("ANTHROPIC_API_KEY")),
    option.WithBaseURL("https://api.anthropic.com/v1/"),
)

// Groq (fast inference for open-source models)
groqClient := openai.NewClient(
    option.WithAPIKey(os.Getenv("GROQ_API_KEY")),
    option.WithBaseURL("https://api.groq.com/openai/v1/"),
)

// Local Ollama instance
ollamaClient := openai.NewClient(
    option.WithAPIKey("ollama"), // Ollama doesn't require a real key
    option.WithBaseURL("http://localhost:11434/v1/"),
)
```

This means you can write provider-agnostic code and swap out the model at runtime.

## Structured Output

When you need the model to return JSON that matches a specific schema — for extraction, classification, or data pipelines — use structured output mode:

```go
type SentimentResult struct {
    Sentiment string  `json:"sentiment"` // "positive", "negative", "neutral"
    Score     float64 `json:"score"`     // 0.0 to 1.0
    Reason    string  `json:"reason"`
}

func analyzeSentiment(ctx context.Context, client *openai.Client, text string) (*SentimentResult, error) {
    schema := map[string]interface{}{
        "type": "object",
        "properties": map[string]interface{}{
            "sentiment": map[string]string{"type": "string", "enum": []string{"positive", "negative", "neutral"}},
            "score":     map[string]string{"type": "number"},
            "reason":    map[string]string{"type": "string"},
        },
        "required":             []string{"sentiment", "score", "reason"},
        "additionalProperties": false,
    }

    completion, err := client.Chat.Completions.New(ctx, openai.ChatCompletionNewParams{
        Model: openai.F(openai.ChatModelGPT4oMini),
        Messages: openai.F([]openai.ChatCompletionMessageParamUnion{
            openai.SystemMessage("Analyze the sentiment of the provided text."),
            openai.UserMessage(text),
        }),
        ResponseFormat: openai.F[openai.ChatCompletionNewParamsResponseFormatUnion](
            openai.ResponseFormatJSONSchemaParam{
                Type: openai.F(openai.ResponseFormatJSONSchemaTypeJSONSchema),
                JSONSchema: openai.F(openai.ResponseFormatJSONSchemaJSONSchemaParam{
                    Name:   openai.F("sentiment_result"),
                    Schema: openai.F[interface{}](schema),
                    Strict: openai.F(true),
                }),
            },
        ),
    })
    if err != nil {
        return nil, err
    }

    var result SentimentResult
    if err := json.Unmarshal([]byte(completion.Choices[0].Message.Content), &result); err != nil {
        return nil, fmt.Errorf("failed to parse structured output: %w", err)
    }

    return &result, nil
}
```

With `strict: true`, the model is constrained to the schema. You'll get valid JSON every time or an error, never a partial or malformed response.

## Managing Context and Token Counts

LLM calls are priced and rate-limited by token count. For long-running conversations or agent loops, you need to manage context explicitly:

```go
// Rough token estimation (actual count requires tiktoken or API call)
func estimateTokens(messages []openai.ChatCompletionMessageParamUnion) int {
    total := 0
    for _, msg := range messages {
        // ~4 chars per token is a rough heuristic
        // Use a proper tokenizer in production
        total += len(fmt.Sprintf("%v", msg)) / 4
    }
    return total
}

// Trim oldest messages when approaching context limit
func trimMessages(
    messages []openai.ChatCompletionMessageParamUnion,
    maxTokens int,
) []openai.ChatCompletionMessageParamUnion {
    for estimateTokens(messages) > maxTokens && len(messages) > 1 {
        // Always keep the system message (index 0)
        messages = append(messages[:1], messages[2:]...)
    }
    return messages
}
```

For production, use a proper tokenizer like `github.com/pkoukk/tiktoken-go` instead of character estimation.

## Production Concerns

A few things that matter once you're past the prototype stage:

**Timeouts**: LLM calls can take 30–60 seconds for long completions. Set explicit timeouts on your context and make sure your load balancer and reverse proxy aren't killing connections before the model finishes.

```go
ctx, cancel := context.WithTimeout(r.Context(), 90*time.Second)
defer cancel()
```

**Concurrency**: The OpenAI client is safe for concurrent use. You can share a single client across goroutines. Don't create a new client per request.

**Caching**: Identical prompts produce equivalent responses. For deterministic use cases (classification, extraction on the same input), cache results in Redis or a simple in-memory store keyed by a hash of the prompt.

**Observability**: Log token usage from every response. OpenAI returns this in `completion.Usage`. Unexpected spikes tell you something's wrong before your bill does.

```go
log.Printf("request=%s prompt_tokens=%d completion_tokens=%d total_tokens=%d",
    requestID,
    completion.Usage.PromptTokens,
    completion.Usage.CompletionTokens,
    completion.Usage.TotalTokens,
)
```

**Cost control**: Set a `MaxTokens` limit on completions. Without it, a model that decides to write an essay will run up your costs and block downstream consumers.

## Putting It Together

Go's strengths — explicit error handling, goroutines, and a clean HTTP server model — align well with what LLM integration actually requires in production: reliable retries, streaming I/O, and concurrent request handling. The patterns here will take you from a working prototype to something you can run in production.

The key things to get right from the start: use context cancellation everywhere, handle rate limits gracefully, stream responses for user-facing flows, and instrument token usage. Everything else — tool use, structured output, multi-provider support — builds naturally on top of these foundations.
