---
title: "Stop Wasting Tokens: Build Your Own Multi-Provider LLM Gateway in Go"
date: 2026-01-26
description: "*A production-grade approach to load balancing, failover, and cost optimization across OpenAI, Azure, Gemini, and more.*  ### The £10,000 Wake-Up Call"
author: "Bhawesh Kumar Singh"
image: "images/blog/llm-gateway-go.png"
categories: ["Go", "AI", "Architecture"]
medium_url: "https://medium.com/@bhaweshkumarsingh/stop-wasting-tokens-build-your-own-multi-provider-llm-gateway-in-go-21a55832f618"
---

*Originally published on [Medium](https://medium.com/@bhaweshkumarsingh/stop-wasting-tokens-build-your-own-multi-provider-llm-gateway-in-go-21a55832f618)*

*A production-grade approach to load balancing, failover, and cost optimization across OpenAI, Azure, Gemini, and more.*

### The £10,000 Wake-Up Call

Last year, our developer team woke up to a Slack message no one wants to see — especially first thing on a Monday morning UK time:

**"OpenAI is down, and so is our entire platform."**

At the time, we were processing thousands of LLM requests per hour. Our AI stack — document extraction, summarisation, and entity resolution — was tightly coupled to OpenAI's API. No abstraction. No fallback. When OpenAI had a **47-minute outage**, we had a **47-minute outage**.

From our users' perspective across the UK and Europe, the product just stopped working. Endless loading states. From our side, the on-call engineer's weekend plans vanished, and incident calls started rolling in.

The following month, we ran into a different kind of problem.

During a traffic spike aligned with UK business hours, Azure OpenAI's rate limits kicked in harder than expected. We were exhausting our **tokens-per-minute quota in the first 30 seconds of every minute**, then sitting idle for the remaining 30.

We weren't under-provisioned — we were inefficiently capped.

The result? Wasted capacity, unpredictable latency, and a cloud bill that quietly crossed **£10,000** for functionality we couldn't fully utilise.

That was the moment we realised something fundamental:

Relying directly on a single LLM provider — without buffering, routing, or control — was an operational risk we could no longer justify.

That's when we decided to build our own **LLM Gateway**.

### Why You Need an LLM Gateway (Before You Learn the Hard Way)

If your application is calling LLM APIs directly from production code, it might work today — but you're quietly accumulating operational risk.

Most teams start simple: pick a provider, wire up the SDK, ship features. That approach gets you to market quickly. It also creates tight coupling between your product and a fast-moving, externally constrained ecosystem.

Over time, the cracks show up in four very predictable places.

### 1. Rate Limits Are Real — and Inconsistent

Every LLM provider enforces rate limits, but they do so in different ways, with different units, thresholds, and behaviours under load.
![](https://cdn-images-1.medium.com/max/1024/1*Mrh70xb1cX422Qz3-zRY2A.png)
When you hit these limits, requests don't degrade gracefully — they fail.

If your application code is calling the provider directly, those failures propagate straight to the user experience. Timeouts, retries, partial failures, and unpredictable latency become your problem to explain to customers and stakeholders.

An LLM Gateway gives you a control layer:
- Queueing instead of failing
- Backoff instead of spiking
- Routing instead of dropping requests

### 2. Provider Outages Are Not Edge Cases

LLM providers are reliable — but they are not infallible.

Over the past year, every major provider has experienced incidents affecting API availability, elevated error rates, or regional degradation. These events are unavoidable in distributed systems operating at this scale.

The real risk isn't that outages happen. The risk is designing an architecture that assumes they won't.

When your product depends on a single provider endpoint, their downtime becomes your downtime. Your incident response is limited to refreshing a status page and writing apologies.

A gateway allows you to:
- Fail over between providers
- Temporarily degrade to cheaper or faster models
- Maintain partial functionality instead of full outage

Resilience stops being theoretical and starts being engineered.

### 3. Cost Optimisation Is Impossible Without Abstraction

Not all requests deserve the same model — but most systems treat them as if they do.
![](https://cdn-images-1.medium.com/max/1024/1*yXzSCNh6ua8y4t-K0fZWcQ.png)
If every request is routed to a top-tier model by default, costs scale linearly with usage — and quickly become difficult to justify.

In practice:
- A large percentage of requests are **classification, extraction, or lightweight summarisation**
- Only a small subset truly require premium reasoning models

An LLM Gateway lets you:
- Route based on task complexity
- Apply rules or policies per endpoint
- Introduce new models without refactoring application code

Cost control becomes a system property, not a manual decision.

### 4. Latency Is a Competitive Feature

Latency isn't just about model speed — it's about geography, deployment, and routing.

Different providers perform differently across regions. An Azure deployment in one geography may consistently outperform a public API endpoint for users in another. But unless you are measuring, comparing, and routing traffic intelligently, you'll never see the benefit.

Without a gateway:
- Latency differences are invisible
- Routing decisions are static
- Performance tuning requires code changes

With a gateway:
- You can route based on region, load, or historical performance
- You can introduce caching, batching, or speculative execution
- You can optimise user experience without redeploying your application

### The Bottom Line

Calling LLM APIs directly from application code is easy — until it isn't.

As usage grows, you inherit:
- Rate-limit fragility
- Provider-level outages
- Uncontrolled cost growth
- Latency you can't optimise

An LLM Gateway doesn't add complexity for its own sake. It introduces **intentional abstraction** — a boundary that gives you control, resilience, and leverage as the ecosystem evolves.

If LLMs are becoming core infrastructure in your product, they deserve to be treated like infrastructure.

Not just an SDK call.

### The Architecture
![](https://cdn-images-1.medium.com/max/1024/1*pX1VUbcg_9K2sHRn_ntt-A.png)![](https://cdn-images-1.medium.com/max/1024/1*Tm7QdxeQ7DfzCacc0nrbtw.png)![](https://cdn-images-1.medium.com/max/1024/1*lSjw0JmkBxeVKu7pc5rlxw.png)
**Your application code doesn't need to know which provider is handling the request**. It just calls your gateway with an OpenAI-compatible API, and the gateway figures out the rest.

### Let's Build It
![](https://cdn-images-1.medium.com/max/1024/0*P36rsRR52iLPDEkO)*Photo by [Wisnu Amaludin](https://unsplash.com/@ciwis?utm_source=medium&utm_medium=referral) on [Unsplash](https://unsplash.com?utm_source=medium&utm_medium=referral)*
I'll walk you through building this in Go using the Fiber framework. We chose Go for its excellent concurrency model and low memory footprint — important when you're proxying thousands of requests per second.

### Project Structure

```text
llm-gateway/
├── cmd/
│   └── api/
│       └── main.go              # Entry point
├── domain/
│   ├── model/
│   │   ├── provider.go          # Provider data model
│   │   ├── feature.go           # Model/feature config
│   │   └── usage.go             # Usage tracking
│   ├── enum/
│   │   └── provider_type.go     # Provider type constants
│   └── query/
│       └── get_available_provider.go  # Load balancing logic
├── provider/
│   ├── openai.go                # OpenAI adapter
│   ├── azure.go                 # Azure OpenAI adapter
│   └── gemini.go                # Gemini adapter
├── handler/
│   └── chat_completion.go       # HTTP handlers
├── middleware/
│   ├── auth.go                  # Authentication
│   └── logging.go               # Request logging
└── go.mod
```

### Step 1: Define Your Provider Types
First, let's define the providers we'll support:

```go
// provider_type.go
package enum

import (
	"fmt"
	"errors"
)

type ProviderType string

const (
	OpenAIProviderType      ProviderType = "openai"
	AzureOpenAIProviderType ProviderType = "azure_openai"
	GeminiProviderType      ProviderType = "gemini"
	AnthropicProviderType   ProviderType = "anthropic"
)

func ParseProviderType(s string) (ProviderType, error) {
	switch s {
	case string(OpenAIProviderType):
		return OpenAIProviderType, nil
	case string(AzureOpenAIProviderType):
		return AzureOpenAIProviderType, nil
	case string(GeminiProviderType):
		return GeminiProviderType, nil
	case string(AnthropicProviderType):
		return AnthropicProviderType, nil
	default:
		return "", fmt.Errorf("'%s' is not a valid provider type", s)
	}
}

func (pt ProviderType) String() string {
	return string(pt)
}
```

### Step 2: Model Your Providers and Their Capabilities
Each provider has different models (features), rate limits, and configurations:

```go
// provider.go
package model

import "llm-gateway/domain/enum"

// Provider represents a configured LLM provider endpoint
type Provider struct {
	ID       string            `bson:"_id"`
	Name     string            `bson:"name"`     // e.g., "azure-east-us", "openai-primary"
	Type     enum.ProviderType `bson:"type"`
	BaseURL  string            `bson:"baseUrl"`  // API endpoint
	APIKey   string            `bson:"apiKey"`
	Features []*Feature        `bson:"features"` // Supported models
}

// GetFeature returns the feature config for a specific model
func (p *Provider) GetFeature(modelName string) *Feature {
	for _, f := range p.Features {
		if f.Name == modelName {
			return f
		}
	}
	return nil
}
```

```go
// feature.go
package model

// Feature represents a model's configuration on a provider
// This is where rate limits and routing weights live
type Feature struct {
	Name                   string  `bson:"name"`    // e.g., "gpt-4", "gpt-35-turbo"
	Version                *string `bson:"version"` // e.g., "1106", "0125"
	RequestsPerMinuteLimit *uint   `bson:"requestsPerMinuteLimit"`
	TokensPerMinuteLimit   *uint   `bson:"tokensPerMinuteLimit"`
	Weight                 *int    `bson:"weight"`  // Higher = preferred
}
```

The Weight field is crucial. It lets you express preferences:
- **Cost optimization**: Give higher weights to cheaper providers
- **Latency optimization**: Give higher weights to faster providers
- **Quality optimization**: Give higher weights to better models

```go
// usage.go
package model

import "time"

// Usage tracks each request for rate limit calculations
type Usage struct {
	ID         string         `bson:"_id"`
	ProviderID string         `bson:"providerId"`
	Feature    string         `bson:"feature"`
	Tokens     *uint          `bson:"tokens"`
	Timestamp  time.Time      `bson:"timestamp"`
	Metadata   *UsageMetadata `bson:"metadata"`
}

type UsageMetadata struct {
	ClientID        string  `bson:"clientId"`
	ClientReference *string `bson:"clientReference"`
	InputTokens     *uint   `bson:"inputTokens"`
	OutputTokens    *uint   `bson:"outputTokens"`
}
```

### Step 3: The Load Balancer (House of Magic)
The algorithm that picks the best available provider:

```go
// get_available_provider.go
package query

import (
	"context"
	"errors"
	"math"
	"time"

	"llm-gateway/domain/enum"
	"llm-gateway/domain/model"

	"go.mongodb.org/mongo-driver/bson"
	"go.mongodb.org/mongo-driver/mongo"
)

type GetMostAvailableProvider struct {
	MongoClient   *mongo.Client
	ProviderTypes []enum.ProviderType
	FeatureName   string // The model being requested (e.g., "gpt-4")
}

func NewGetMostAvailableProvider(
	mongoClient *mongo.Client,
	providerTypes []enum.ProviderType,
	featureName string,
) *GetMostAvailableProvider {
	return &GetMostAvailableProvider{
		MongoClient:   mongoClient,
		ProviderTypes: providerTypes,
		FeatureName:   featureName,
	}
}

func (q *GetMostAvailableProvider) Execute(ctx context.Context) (*model.Provider, error) {
	// Look at usage from the last minute for rate limit calculations
	since := time.Now().UTC().Add(-1 * time.Minute)

	pipeline := bson.A{
		// Step 1: Find providers that support this model
		bson.M{
			"$match": bson.M{
				"type": bson.M{
					"$in": q.ProviderTypes,
				},
				"features.name": q.FeatureName,
			},
		},
		// Step 2: Join with usage data from the last minute
		bson.M{
			"$lookup": bson.M{
				"from":         "usage",
				"localField":   "_id",
				"foreignField": "providerId",
				"as":           "recentUsage",
				"pipeline": bson.A{
					bson.M{
						"$match": bson.M{
							"timestamp": bson.M{"$gte": since},
							"feature":   q.FeatureName,
						},
					},
				},
			},
		},
		// Step 3: Extract the feature config for this model
		bson.M{
			"$addFields": bson.M{
				"featureConfig": bson.M{
					"$first": bson.M{
						"$filter": bson.M{
							"input": "$features",
							"as":    "f",
							"cond":  bson.M{"$eq": bson.A{"$$f.name", q.FeatureName}},
						},
					},
				},
			},
		},
		// Step 4: Calculate availability scores
		bson.M{
			"$addFields": bson.M{
				// How many tokens are still available this minute?
				"tokenAvailability": bson.M{
					"$ifNull": bson.A{
						bson.M{
							"$subtract": bson.A{
								"$featureConfig.tokensPerMinuteLimit",
								bson.M{"$sum": "$recentUsage.tokens"},
							},
						},
						math.MaxInt64, // No limit configured = infinite availability
					},
				},
				// How many requests are still available this minute?
				"requestAvailability": bson.M{
					"$ifNull": bson.A{
						bson.M{
							"$subtract": bson.A{
								"$featureConfig.requestsPerMinuteLimit",
								bson.M{"$size": "$recentUsage"},
							},
						},
						math.MaxInt64,
					},
				},
				// Provider weight (for preference-based routing)
				"weight": bson.M{
					"$ifNull": bson.A{"$featureConfig.weight", 0},
				},
			},
		},
		// Step 5: Sort by weight first, then by availability
		bson.M{
			"$sort": bson.D{
				{Key: "weight", Value: -1},              // Prefer higher-weighted providers
				{Key: "tokenAvailability", Value: -1},   // Then prefer more token headroom
				{Key: "requestAvailability", Value: -1}, // Then prefer more request headroom
			},
		},
		// Step 6: Only return providers with actual availability
		bson.M{
			"$match": bson.M{
				"tokenAvailability":   bson.M{"$gt": 0},
				"requestAvailability": bson.M{"$gt": 0},
			},
		},
		// Step 7: Take the best one
		bson.M{"$limit": 1},
		// Clean up temporary fields
		bson.M{
			"$project": bson.M{
				"recentUsage":         0,
				"featureConfig":       0,
				"tokenAvailability":   0,
				"requestAvailability": 0,
				"weight":              0,
			},
		},
	}

	db := q.MongoClient.Database("llm_gateway")
	cursor, err := db.Collection("providers").Aggregate(ctx, pipeline)
	if err != nil {
		return nil, err
	}

	var providers []*model.Provider
	if err := cursor.All(ctx, &providers); err != nil {
		return nil, err
	}

	if len(providers) == 0 {
		return nil, ErrNoProviderAvailable
	}

	return providers[0], nil
}

var ErrNoProviderAvailable = errors.New("no provider available for this model")
```

**Why MongoDB aggregation?**

We need to make a decision based on real-time usage data. This aggregation:
- Runs in ~5-10ms
- Considers all providers atomically
- Handles the math at the database level (no round trips)

For higher scale, you could replace this with Redis sorted sets, but MongoDB works great up to ~10K RPM.

### Step 4: The Provider Adapters

Each provider has slightly different APIs. We normalize them:

```go
// interface.go
package provider

import (
	"context"

	"llm-gateway/domain/enum"
)

// ChatCompletionRequest is our normalized request format
type ChatCompletionRequest struct {
	Model       string    `json:"model"`
	Messages    []Message `json:"messages"`
	Temperature *float64  `json:"temperature,omitempty"`
	MaxTokens   *int      `json:"max_tokens,omitempty"`
	Stream      bool      `json:"stream,omitempty"`
}

type Message struct {
	Role    string `json:"role"`
	Content string `json:"content"`
}

// ChatCompletionResponse is our normalized response format
type ChatCompletionResponse struct {
	ID      string   `json:"id"`
	Model   string   `json:"model"`
	Choices []Choice `json:"choices"`
	Usage   Usage    `json:"usage"`
}

type Choice struct {
	Index        int     `json:"index"`
	Message      Message `json:"message"`
	FinishReason string  `json:"finish_reason"`
}

type Usage struct {
	PromptTokens     int `json:"prompt_tokens"`
	CompletionTokens int `json:"completion_tokens"`
	TotalTokens      int `json:"total_tokens"`
}

// LLMProvider is the interface all provider adapters must implement
type LLMProvider interface {
	ChatCompletion(ctx context.Context, req ChatCompletionRequest) (*ChatCompletionResponse, error)
	ProviderType() enum.ProviderType
}
```

Now the OpenAI adapter:

```go
// openai.go
package provider

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"strings"

	"llm-gateway/domain/enum"
	"llm-gateway/domain/model"
)

type OpenAIAdapter struct {
	httpClient *http.Client
	provider   *model.Provider
}

func NewOpenAIAdapter(httpClient *http.Client, provider *model.Provider) *OpenAIAdapter {
	return &OpenAIAdapter{
		httpClient: httpClient,
		provider:   provider,
	}
}

func (a *OpenAIAdapter) ProviderType() enum.ProviderType {
	return a.provider.Type
}

func (a *OpenAIAdapter) ChatCompletion(ctx context.Context, req ChatCompletionRequest) (*ChatCompletionResponse, error) {
	// Normalize the model name for this provider
	modelName := a.normalizeModelName(req.Model)
	req.Model = modelName

	// Build the request
	body, err := json.Marshal(req)
	if err != nil {
		return nil, fmt.Errorf("failed to marshal request: %w", err)
	}

	baseURL := a.getBaseURL()
	httpReq, err := http.NewRequestWithContext(
		ctx,
		"POST",
		baseURL+"/chat/completions",
		bytes.NewReader(body),
	)
	if err != nil {
		return nil, fmt.Errorf("failed to create request: %w", err)
	}

	// Set headers
	httpReq.Header.Set("Content-Type", "application/json")
	httpReq.Header.Set("Authorization", "Bearer "+a.provider.APIKey)

	// Make the request
	resp, err := a.httpClient.Do(httpReq)
	if err != nil {
		return nil, fmt.Errorf("request failed: %w", err)
	}
	defer resp.Body.Close()

	// Read response
	respBody, err := io.ReadAll(resp.Body)
	if err != nil {
		return nil, fmt.Errorf("failed to read response: %w", err)
	}

	// Check for errors
	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("OpenAI API error (status %d): %s", resp.StatusCode, string(respBody))
	}

	// Parse response
	var result ChatCompletionResponse
	if err := json.Unmarshal(respBody, &result); err != nil {
		return nil, fmt.Errorf("failed to parse response: %w", err)
	}

	return &result, nil
}

func (a *OpenAIAdapter) getBaseURL() string {
	if a.provider.BaseURL != "" {
		return strings.TrimSuffix(a.provider.BaseURL, "/")
	}
	return "https://api.openai.com/v1"
}

func (a *OpenAIAdapter) normalizeModelName(model string) string {
	// OpenAI uses "gpt-3.5-turbo", Azure uses "gpt-35-turbo"
	// Normalize to OpenAI format
	return strings.ReplaceAll(model, "-35-", "-3.5-")
}
```

The Azure adapter is similar but handles their different authentication and URL patterns:

```go
// azure.go
package provider

import (
	"bytes"
	"context"
	"encoding/json"
	"fmt"
	"io"
	"net/http"
	"strings"

	"llm-gateway/domain/enum"
	"llm-gateway/domain/model"
)

const azureAPIVersion = "2024-08-01-preview"

type AzureOpenAIAdapter struct {
	httpClient *http.Client
	provider   *model.Provider
}

func NewAzureOpenAIAdapter(httpClient *http.Client, provider *model.Provider) *AzureOpenAIAdapter {
	return &AzureOpenAIAdapter{
		httpClient: httpClient,
		provider:   provider,
	}
}

func (a *AzureOpenAIAdapter) ProviderType() enum.ProviderType {
	return enum.AzureOpenAIProviderType
}

func (a *AzureOpenAIAdapter) ChatCompletion(ctx context.Context, req ChatCompletionRequest) (*ChatCompletionResponse, error) {
	// Azure uses deployment names, not model names
	deploymentName := a.normalizeModelName(req.Model)

	body, err := json.Marshal(req)
	if err != nil {
		return nil, fmt.Errorf("failed to marshal request: %w", err)
	}

	// Azure URL format: {endpoint}/openai/deployments/{deployment}/chat/completions?api-version={version}
	url := fmt.Sprintf(
		"%s/openai/deployments/%s/chat/completions?api-version=%s",
		strings.TrimSuffix(a.provider.BaseURL, "/"),
		deploymentName,
		azureAPIVersion,
	)

	httpReq, err := http.NewRequestWithContext(ctx, "POST", url, bytes.NewReader(body))
	if err != nil {
		return nil, fmt.Errorf("failed to create request: %w", err)
	}

	// Azure uses api-key header instead of Bearer token
	httpReq.Header.Set("Content-Type", "application/json")
	httpReq.Header.Set("api-key", a.provider.APIKey)

	resp, err := a.httpClient.Do(httpReq)
	if err != nil {
		return nil, fmt.Errorf("request failed: %w", err)
	}
	defer resp.Body.Close()

	respBody, err := io.ReadAll(resp.Body)
	if err != nil {
		return nil, fmt.Errorf("failed to read response: %w", err)
	}

	if resp.StatusCode != http.StatusOK {
		return nil, fmt.Errorf("Azure OpenAI API error (status %d): %s", resp.StatusCode, string(respBody))
	}

	var result ChatCompletionResponse
	if err := json.Unmarshal(respBody, &result); err != nil {
		return nil, fmt.Errorf("failed to parse response: %w", err)
	}

	return &result, nil
}

func (a *AzureOpenAIAdapter) normalizeModelName(model string) string {
	// Azure deployments typically use names like "gpt-35-turbo" (no dots)
	return strings.ReplaceAll(model, "-3.5-", "-35-")
}
```

### Step 5: The HTTP Handler
Now let's wire it all together:

```go
// chat_completion.go
package handler

import (
	"context"
	"fmt"
	"net/http"
	"time"

	"llm-gateway/domain/enum"
	"llm-gateway/domain/model"
	"llm-gateway/domain/query"
	"llm-gateway/provider"

	"github.com/gofiber/fiber/v2"
	"go.mongodb.org/mongo-driver/mongo"
)

type ChatCompletionHandler struct {
	mongoClient *mongo.Client
	httpClient  *http.Client
}

func NewChatCompletionHandler(mongoClient *mongo.Client, httpClient *http.Client) *ChatCompletionHandler {
	return &ChatCompletionHandler{
		mongoClient: mongoClient,
		httpClient:  httpClient,
	}
}

func (h *ChatCompletionHandler) Handle(c *fiber.Ctx) error {
	ctx := c.UserContext()

	// Parse the incoming request
	var req provider.ChatCompletionRequest
	if err := c.BodyParser(&req); err != nil {
		return c.Status(400).JSON(fiber.Map{
			"error": "Invalid request body",
		})
	}

	if req.Model == "" {
		return c.Status(400).JSON(fiber.Map{
			"error": "Model is required",
		})
	}

	// Find the best available provider
	selectedProvider, err := h.findBestProvider(ctx, req.Model)
	if err != nil {
		if err == query.ErrNoProviderAvailable {
			return c.Status(503).JSON(fiber.Map{
				"error": "No provider available for this model. Please try again later.",
			})
		}
		return c.Status(500).JSON(fiber.Map{
			"error": "Failed to select provider",
		})
	}

	// Create the appropriate adapter
	adapter := h.createAdapter(selectedProvider)

	// Make the request
	startTime := time.Now()
	resp, err := adapter.ChatCompletion(ctx, req)
	latency := time.Since(startTime)

	if err != nil {
		// Record the failure
		h.recordUsage(ctx, selectedProvider, req.Model, 0, false)
		return c.Status(502).JSON(fiber.Map{
			"error": "Provider request failed: " + err.Error(),
		})
	}

	// Record successful usage
	h.recordUsage(ctx, selectedProvider, req.Model, uint(resp.Usage.TotalTokens), true)

	// Add headers so the caller knows which provider was used
	c.Set("X-LLM-Provider", selectedProvider.Name)
	c.Set("X-LLM-Provider-Type", string(selectedProvider.Type))
	c.Set("X-LLM-Latency-Ms", fmt.Sprintf("%d", latency.Milliseconds()))

	return c.JSON(resp)
}

func (h *ChatCompletionHandler) findBestProvider(ctx context.Context, modelName string) (*model.Provider, error) {
	// Default to trying Azure first (usually cheaper), then OpenAI
	providerTypes := []enum.ProviderType{
		enum.AzureOpenAIProviderType,
		enum.OpenAIProviderType,
	}

	q := query.NewGetMostAvailableProvider(h.mongoClient, providerTypes, modelName)
	return q.Execute(ctx)
}

func (h *ChatCompletionHandler) createAdapter(p *model.Provider) provider.LLMProvider {
	switch p.Type {
	case enum.AzureOpenAIProviderType:
		return provider.NewAzureOpenAIAdapter(h.httpClient, p)
	case enum.OpenAIProviderType:
		return provider.NewOpenAIAdapter(h.httpClient, p)
	default:
		return provider.NewOpenAIAdapter(h.httpClient, p)
	}
}

func (h *ChatCompletionHandler) recordUsage(ctx context.Context, p *model.Provider, feature string, tokens uint, success bool) {
	usage := &model.Usage{
		ProviderID: p.ID,
		Feature:    feature,
		Tokens:     &tokens,
		Timestamp:  time.Now().UTC(),
	}

	db := h.mongoClient.Database("llm_gateway")
	_, _ = db.Collection("usage").InsertOne(ctx, usage)

	// Also emit metrics for observability
	metrics.RecordRequest(p.Name, feature, tokens, success)
}
```

### Step 6: The Main Entry Point

```go
// main.go
package main

import (
	"context"
	"log"
	"net"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"llm-gateway/handler"
	"llm-gateway/middleware"

	"github.com/gofiber/fiber/v2"
	"github.com/gofiber/fiber/v2/middleware/logger"
	"github.com/gofiber/fiber/v2/middleware/recover"
	"go.mongodb.org/mongo-driver/mongo"
	"go.mongodb.org/mongo-driver/mongo/options"
)

func main() {
	log.Println("Starting LLM Gateway...")

	// Initialize MongoDB
	mongoClient, err := initMongo()
	if err != nil {
		log.Fatalf("Failed to connect to MongoDB: %v", err)
	}
	defer mongoClient.Disconnect(context.Background())

	// Initialize HTTP client with connection pooling
	httpClient := initHTTPClient()

	// Initialize handlers
	chatHandler := handler.NewChatCompletionHandler(mongoClient, httpClient)

	// Create Fiber app
	app := fiber.New(fiber.Config{
		ServerHeader:          "LLM-Gateway",
		ReadTimeout:           30 * time.Second,
		WriteTimeout:          120 * time.Second, // LLM responses can be slow
		IdleTimeout:           120 * time.Second,
		DisableStartupMessage: false,
	})

	// Middleware
	app.Use(recover.New())
	app.Use(logger.New())
	app.Use(middleware.Authentication())

	// Health check
	app.Get("/health", func(c *fiber.Ctx) error {
		return c.JSON(fiber.Map{"status": "healthy"})
	})

	// OpenAI-compatible endpoints
	app.Post("/v1/chat/completions", chatHandler.Handle)
	app.Post("/openai/chat/completions", chatHandler.Handle) // Alias

	// Start server
	go func() {
		port := os.Getenv("PORT")
		if port == "" {
			port = "8080"
		}
		log.Printf("Server starting on port %s", port)
		if err := app.Listen(":" + port); err != nil {
			log.Fatalf("Server failed: %v", err)
		}
	}()

	// Graceful shutdown
	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit

	log.Println("Shutting down server...")
	app.ShutdownWithTimeout(10 * time.Second)
	log.Println("Server stopped")
}

func initMongo() (*mongo.Client, error) {
	ctx, cancel := context.WithTimeout(context.Background(), 10*time.Second)
	defer cancel()

	uri := os.Getenv("MONGODB_URI")
	if uri == "" {
		uri = "mongodb://localhost:27017"
	}

	client, err := mongo.Connect(ctx, options.Client().ApplyURI(uri))
	if err != nil {
		return nil, err
	}

	// Ping to verify connection
	if err := client.Ping(ctx, nil); err != nil {
		return nil, err
	}

	return client, nil
}

func initHTTPClient() *http.Client {
	transport := &http.Transport{
		DialContext: (&net.Dialer{
			Timeout:   30 * time.Second,
			KeepAlive: 30 * time.Second,
		}).DialContext,
		ForceAttemptHTTP2:     true,
		MaxIdleConns:          100,
		MaxIdleConnsPerHost:   100,
		IdleConnTimeout:       90 * time.Second,
		TLSHandshakeTimeout:  10 * time.Second,
		ExpectContinueTimeout: 1 * time.Second,
	}

	return &http.Client{
		Transport: transport,
		Timeout:   120 * time.Second, // LLM requests can be slow
	}
}
```

### Advanced Features

### Automatic Failover
When a provider fails, retry with another:

```go
func (h *ChatCompletionHandler) HandleWithRetry(c *fiber.Ctx) error {
	ctx := c.UserContext()

	var req provider.ChatCompletionRequest
	if err := c.BodyParser(&req); err != nil {
		return c.Status(400).JSON(fiber.Map{"error": "Invalid request"})
	}

	// Track which providers we've tried
	triedProviders := make(map[string]bool)
	maxRetries := 3

	for attempt := 0; attempt < maxRetries; attempt++ {
		provider, err := h.findBestProviderExcluding(ctx, req.Model, triedProviders)
		if err != nil {
			break // No more providers available
		}

		triedProviders[provider.ID] = true
		adapter := h.createAdapter(provider)

		resp, err := adapter.ChatCompletion(ctx, req)
		if err == nil {
			// Success!
			h.recordUsage(ctx, provider, req.Model, uint(resp.Usage.TotalTokens), true)
			c.Set("X-LLM-Provider", provider.Name)
			c.Set("X-LLM-Retry-Count", fmt.Sprintf("%d", attempt))
			return c.JSON(resp)
		}

		// Record failure and try next provider
		h.recordUsage(ctx, provider, req.Model, 0, false)
		log.Printf("Provider %s failed, trying next... (attempt %d/%d)",
			provider.Name, attempt+1, maxRetries)
	}

	return c.Status(503).JSON(fiber.Map{
		"error": "All providers failed",
	})
}
```

### Request Caching
For deterministic requests (temperature=0), cache the results:

```go
// cache.go
package middleware

import (
	"crypto/sha256"
	"encoding/hex"
	"encoding/json"
	"time"

	"github.com/gofiber/fiber/v2"
)

type CacheMiddleware struct {
	cache CacheService // Redis, Memcached, etc.
	ttl   time.Duration
}

func (m *CacheMiddleware) Handle(c *fiber.Ctx) error {
	// Only cache if temperature is 0 (deterministic)
	var req map[string]interface{}
	json.Unmarshal(c.Body(), &req)

	temp, ok := req["temperature"].(float64)
	if !ok || temp != 0 {
		return c.Next() // Don't cache non-deterministic requests
	}

	// Generate cache key from request body
	hash := sha256.Sum256(c.Body())
	cacheKey := "llm:" + hex.EncodeToString(hash[:])

	// Check cache
	if cached, found := m.cache.Get(cacheKey); found {
		c.Set("X-Cache", "HIT")
		return c.JSON(cached)
	}

	// Process request
	err := c.Next()
	if err != nil {
		return err
	}

	// Cache successful responses
	if c.Response().StatusCode() == 200 {
		var resp interface{}
		json.Unmarshal(c.Response().Body(), &resp)
		m.cache.Set(cacheKey, resp, m.ttl)
		c.Set("X-Cache", "MISS")
	}

	return nil
}
```

### Cost Tracking
Track costs per client for billing:

```go
// cost.go
package model

// Cost per 1M tokens (as of 2026)
var ModelCosts = map[string]struct {
	InputCost  float64
	OutputCost float64
}{
	"gpt-4":         {30.00, 60.00},
	"gpt-4-turbo":   {10.00, 30.00},
	"gpt-4o":        {5.00, 15.00},
	"gpt-4o-mini":   {0.15, 0.60},
	"gpt-3.5-turbo": {0.50, 1.50},
	"claude-3-opus": {15.00, 75.00},
}

func CalculateCost(model string, inputTokens, outputTokens int) float64 {
	costs, ok := ModelCosts[model]
	if !ok {
		return 0 // Unknown model
	}

	inputCost := (float64(inputTokens) / 1_000_000) * costs.InputCost
	outputCost := (float64(outputTokens) / 1_000_000) * costs.OutputCost

	return inputCost + outputCost
}
```

### Observability
You can't optimize what you can't measure. Add OpenTelemetry:

```go
// metrics.go
package metrics

import (
	"context"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/attribute"
	"go.opentelemetry.io/otel/metric"
)

var (
	meter          = otel.Meter("llm-gateway")
	requestCounter metric.Int64Counter
	tokenCounter   metric.Int64Counter
	latencyHist    metric.Float64Histogram
	errorCounter   metric.Int64Counter
)

func init() {
	requestCounter, _ = meter.Int64Counter("llm.requests.total",
		metric.WithDescription("Total LLM requests"))

	tokenCounter, _ = meter.Int64Counter("llm.tokens.total",
		metric.WithDescription("Total tokens processed"))

	latencyHist, _ = meter.Float64Histogram("llm.latency.seconds",
		metric.WithDescription("Request latency in seconds"))

	errorCounter, _ = meter.Int64Counter("llm.errors.total",
		metric.WithDescription("Total errors"))
}

func RecordRequest(ctx context.Context, provider, model, client string,
	tokens int, latencyMs int64, success bool) {

	attrs := []attribute.KeyValue{
		attribute.String("provider", provider),
		attribute.String("model", model),
		attribute.String("client", client),
		attribute.Bool("success", success),
	}

	requestCounter.Add(ctx, 1, metric.WithAttributes(attrs...))
	tokenCounter.Add(ctx, int64(tokens), metric.WithAttributes(attrs...))
	latencyHist.Record(ctx, float64(latencyMs)/1000, metric.WithAttributes(attrs...))

	if !success {
		errorCounter.Add(ctx, 1, metric.WithAttributes(attrs...))
	}
}
```

With these metrics, you can build dashboards showing:
- Requests per provider over time
- Token usage per client
- P50/P95/P99 latencies
- Error rates by provider
- Cost per client

### Seeding Your Providers

Finally, you need to configure your providers in MongoDB:

```javascript
// MongoDB seed script
db.providers.insertMany([
  {
    _id: "azure-east-us",
    name: "Azure OpenAI East US",
    type: "azure_openai",
    baseUrl: "https://your-resource.openai.azure.com",
    apiKey: "your-azure-api-key",
    features: [
      {
        name: "gpt-4",
        version: "turbo-2024-04-09",
        requestsPerMinuteLimit: 100,
        tokensPerMinuteLimit: 150000,
        weight: 10  // Prefer Azure (cheaper)
      },
      {
        name: "gpt-35-turbo",
        version: "0125",
        requestsPerMinuteLimit: 300,
        tokensPerMinuteLimit: 300000,
        weight: 10
      }
    ]
  },
  {
    _id: "openai-primary",
    name: "OpenAI Primary",
    type: "openai",
    baseUrl: "https://api.openai.com/v1",
    apiKey: "sk-your-openai-key",
    features: [
      {
        name: "gpt-4",
        requestsPerMinuteLimit: 60,
        tokensPerMinuteLimit: 100000,
        weight: 5  // Fallback
      },
      {
        name: "gpt-3.5-turbo",
        requestsPerMinuteLimit: 60,
        tokensPerMinuteLimit: 100000,
        weight: 5
      }
    ]
  }
]);
```

### Results
After deploying our LLM Gateway:
- **Zero downtime** guaranteed by automatic failover to Azure.
- **30% cost reduction** by routing appropriate requests to cheaper providers
- **2x throughput** by utilizing multiple providers' rate limits in parallel
- **Full visibility** into which clients are using which models and how much

The gateway processes ~50K+ requests per day with P95 latency overhead of just 15ms.

### Conclusion

Building an LLM Gateway isn't simply a defensive move against outages — it's a strategic decision to take ownership of your AI infrastructure.

By introducing a dedicated control plane between your application and LLM providers, you gain:
- **Resilience** — Automatic failover across providers and deployments, so individual incidents don't become customer-facing outages.
- **Cost optimisation** — Intelligent routing to the most cost-effective model that meets the task's requirements, rather than defaulting to the most expensive option.
- **Observability** — Clear, centralised visibility into every request — latency, failures, retries, and spend — instead of opaque provider-side behaviour.
- **Flexibility** — The ability to add new providers, adjust routing rules, or change model strategy without rewriting or redeploying application code.

The code shared in this post is production-tested and actively handling real-world traffic. It reflects the operational realities of running LLMs at scale, not a theoretical reference implementation.

Use it and adapt it to your environment. And stop letting provider constraints dictate the shape of your architecture.
