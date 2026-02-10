---
title: "Building Observability in AWS Lambda with OpenTelemetry: A Step-by-Step Guide"
date: 2025-08-28
description: "### Introduction  Serverless is amazing — you just drop your code into **AWS Lambda** and it scales without worrying about infrastructure. But there's"
author: "Bhawesh Kumar Singh"
categories: ["AWS", "Observability", "Go"]
medium_url: "https://medium.com/@bhaweshkumarsingh/building-observability-in-aws-lambda-with-opentelemetry-a-step-by-step-guide-08ad103b059a"
---

*Originally published on [Medium](https://medium.com/@bhaweshkumarsingh/building-observability-in-aws-lambda-with-opentelemetry-a-step-by-step-guide-08ad103b059a)*

### Introduction

Serverless is amazing — you just drop your code into **AWS Lambda** and it scales without worrying about infrastructure. But there's a catch: Lambdas are short-lived, spin up on demand, and disappear just as quickly. That makes it tricky to answer questions like:
- *Why is my function slow sometimes?*
- *Which request caused that error?*
- *How do I connect what happened in Lambda to the rest of my system?*

That's where **observability** comes in. Observability means being able to see *inside* your systems through three pillars:
- **Traces** → show the flow of a request across services
- **Metrics** → show the performance (latency, errors, throughput)
- **Logs** → tell you the detailed story of what happened

To make this work in a clean, vendor-neutral way, we'll use **OpenTelemetry (OTEL)** — an open-source standard for collecting and exporting telemetry data.

In this guide, we'll set up observability for AWS Lambda functions written in **Go**, and we'll cover both **API Gateway (HTTP)** and **SQS (message queue)** handlers. By the end, you'll have a reusable setup that ships your telemetry to an OTEL collector, where tools like **Jaeger, Prometheus, or Grafana** can visualize it.

Let's dive in 🚀

### What We Actually Need

Serverless functions introduce some unique challenges:
- They live only for the duration of a request. That means we need to flush telemetry data per invocation so nothing gets lost.
- Cold starts add latency, which we want to measure and monitor.
- Requests often hop between services (API Gateway → Lambda → SQS → another Lambda). We need trace IDs to follow the full journey.

So here's our toolkit:
- Tracing → follow requests end-to-end
- Metrics → track performance trends
- Logging → structured logs with trace IDs

We're using Go because it's lightweight and efficient for Lambda. The OTEL Go SDK is also quite mature.

Prerequisites:
- Go 1.24+
- AWS account with Lambda functions
- An OTEL collector (local or in AWS)
- Familiarity with Go modules + Lambda basics

Start a new module:

```bash
go mod init my-lambda-observability
```

### Setting Up the OpenTelemetry Provider

The heart of our setup is an OtelProvider. It initializes tracing, metrics, and logging once and makes them available everywhere.

Why bother? Because consistent initialization ensures *every piece of your code* (even third-party libraries) uses the same setup.

Create a telemetry package with this provider:

```go
package telemetry

import (
	"context"
	"time"

	"go.opentelemetry.io/otel"
	"go.opentelemetry.io/otel/exporters/otlp/otlplog/otlploghttp"
	"go.opentelemetry.io/otel/exporters/otlp/otlpmetric/otlpmetrichttp"
	"go.opentelemetry.io/otel/exporters/otlp/otlptrace/otlptracehttp"
	"go.opentelemetry.io/otel/sdk/log"
	"go.opentelemetry.io/otel/sdk/metric"
	"go.opentelemetry.io/otel/sdk/trace"
	"go.uber.org/zap"
	"go.uber.org/zap/zapcore"
)

type OtelProvider struct {
	tracerProvider *tracerProvider
	meterProvider  *meterProvider
	loggerProvider *loggerProvider
}

func NewProvider(ctx context.Context) (*OtelProvider, error) {
	traceExporter, err := otlptracehttp.New(ctx)
	if err != nil {
		return nil, err
	}

	metricExporter, err := otlpmetrichttp.New(ctx)
	if err != nil {
		return nil, err
	}

	metricReader := metric.NewPeriodicReader(metricExporter, metric.WithInterval(10*time.Second))

	logExporter, err := otlploghttp.New(ctx)
	if err != nil {
		return nil, err
	}

	logger, err := zap.NewProduction()
	if err != nil {
		return nil, err
	}

	return newOtelProvider(traceExporter, metricReader, logExporter, logger.Core()), nil
}

func newOtelProvider(traceExporter trace.SpanExporter, meterReader metric.Reader, logExporter log.Exporter, logCore zapcore.Core) *OtelProvider {
	return &OtelProvider{
		tracerProvider: newTracerProvider(traceExporter),
		meterProvider:  newMeterProvider(meterReader),
		loggerProvider: newLoggerProvider(logExporter, logCore),
	}
}
```

Configure your collector endpoint:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
```

Now traces, metrics, and logs will be exported via HTTP to your collector.

### Logging with Zap + OTEL

Logs are your detailed "play-by-play." We'll use Zap (fast, structured logging) and bridge it into OTEL.

Why? Because logs without context are useless. By injecting trace IDs into every log, you can filter logs for a specific request in Grafana, Loki, or ELK.

Here's a logger provider:

```go
type loggerProvider struct {
	provider *log.LoggerProvider
	logger   *zap.Logger
}

func newLoggerProvider(exporter log.Exporter, core zapcore.Core) *loggerProvider {
	provider := log.NewLoggerProvider(log.WithProcessor(log.NewBatchProcessor(exporter)))
	logger := zap.New(core)
	return &loggerProvider{provider: provider, logger: logger}
}

func (l *loggerProvider) loggerContext(ctx context.Context) context.Context {
	span := trace.SpanFromContext(ctx)
	fields := []zap.Field{zap.String("trace.id", span.SpanContext().TraceID().String())}
	childLogger := l.logger.With(fields...)
	return context.WithValue(ctx, "logger", childLogger)
}

func GetLogger(ctx context.Context) *zap.Logger {
	if logger, ok := ctx.Value("logger").(*zap.Logger); ok {
		return logger
	}
	fallback, _ := zap.NewProduction()
	return fallback
}
```

Tip: Set OTEL Logger to Zap core only to avoid OTEL forwarding its own logs

```go
otel.SetLogger(zapr.NewLogger(zap.New(core)))
```

Now, in your handler:

```go
GetLogger(ctx).Info("Processing request", zap.String("userId", "123"))
```

Every log line now carries trace context.

### Tracing for Distributed Requests

Traces show how a request flows through multiple services.

Define a tracer provider:

```go
type tracerProvider struct {
	tracerProvider *trace.TracerProvider
}

func newTracerProvider(exporter trace.SpanExporter) *tracerProvider {
	provider := trace.NewTracerProvider(trace.WithBatcher(exporter))
	otel.SetTracerProvider(provider)
	return &tracerProvider{tracerProvider: provider}
}
```

In your handler, attach attributes:

```go
span := trace.SpanFromContext(ctx)
span.SetAttributes(
	semconv.HTTPRequestMethodKey.String(req.HTTPMethod),
	semconv.URLPath(req.Path),
)
```

Handle propagation: Use *propagation.NewCompositeTextMapPropagator* for headers like traceparent.

This creates spans with semantic conventions (semconv) for standardization, ensuring compatibility with visualization tools.

### Metrics for Performance

Metrics let you monitor trends without drowning in trace data.

```go
type meterProvider struct {
	meterProvider *metric.MeterProvider
}

func newMeterProvider(reader metric.Reader) *meterProvider {
	provider := metric.NewMeterProvider(metric.WithReader(reader))
	otel.SetMeterProvider(provider)
	return &meterProvider{meterProvider: provider}
}
```

Create a histogram:

```go
meter := m.meterProvider.Meter("telemetry")
httpDuration, _ := meter.Float64Histogram(
	"http.server.request.duration",
	metric.WithUnit("s"),
	metric.WithDescription("Duration of HTTP requests"),
)
```

Record inside your handler:

```go
start := time.Now()
// ...handle request...
httpDuration.Record(ctx, time.Since(start).Seconds())
```

### Wrapping Lambda Handlers

Tie it all together with handler wrappers.

For API Gateway:

```go
func (o *OtelProvider) InstrumentAPIHandler(handler APIHandler) APIHandler {
	return func(ctx context.Context, req events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
		ctx = o.loggerProvider.loggerContext(ctx)
		start := time.Now()

		spanCtx, span := otel.Tracer("telemetry").Start(ctx, "APIGatewayHandler")
		defer span.End()

		resp, err := handler(spanCtx, req)

		// Record metrics
		o.meterProvider.meterProvider.Meter("telemetry").
			Histogram("http.server.request.duration").Record(spanCtx, time.Since(start).Seconds())

		defer forceFlush(ctx, 100*time.Millisecond, o.loggerProvider, o.meterProvider, o.tracerProvider)

		return resp, err
	}
}
```

In main.go:

```go
provider, _ := telemetry.NewProvider(context.Background())
lambda.Start(provider.InstrumentAPIHandler(yourHandler))
```

This ensures all logs, metrics, and traces are flushed before Lambda shuts down.

### Testing Your Setup

Before deploying, test locally:
- Use in-memory exporters for unit tests
- Assert that spans/metrics/logs are being created
- Simulate timeouts to ensure the flush doesn't block

Once deployed, check your OTEL collector → data should flow into Jaeger/Grafana.

### Best Practices

- Keep flush short: 100ms is usually safe
- Add error attributes: `span.RecordError(err)`
- Use sampling: to avoid trace overload in production
- Export to X-Ray: if you want AWS native integration

### Conclusion

You now have a full observability setup for AWS Lambda in Go: traces, metrics, and logs all wired up with OpenTelemetry.

With this in place, debugging shifts from *guesswork* to *clear insights*. You can follow requests across services, monitor latency spikes, and drill down into structured logs — all in one flow.

Start small, deploy this setup, and build up your dashboards in Grafana or Jaeger. Your future self (and your on-call teammates) will thank you.

### References

- [OpenTelemetry Docs](https://opentelemetry.io?utm_source=chatgpt.com)
- [AWS Lambda Go SDK](https://github.com/aws/aws-lambda-go?utm_source=chatgpt.com)
- Zap Logging
