---
title: "Building Durable Payment Workflows with Temporal in Go: My Lessons from Engineering a UK Fintech's"
date: 2026-01-29
description: "### Building Durable Payment Workflows with Temporal in Go: My Lessons from Engineering a UK Fintech's Production System  How we built a production pa"
author: "Bhawesh Kumar Singh"
image: "images/blog/temporal-payment-workflows.png"
categories: ["Go", "Fintech", "Architecture"]
medium_url: "https://medium.com/@bhaweshkumarsingh/building-durable-payment-workflows-with-temporal-in-go-my-lessons-from-engineering-a-uk-fintechs-f28b647d8155"
---

*Originally published on [Medium](https://medium.com/@bhaweshkumarsingh/building-durable-payment-workflows-with-temporal-in-go-my-lessons-from-engineering-a-uk-fintechs-f28b647d8155)*

### Building Durable Payment Workflows with Temporal in Go: My Lessons from Engineering a UK Fintech's Production System

How we built a production payment system that handles SEPA, SWIFT, and domestic transfers with 7-day workflow durability, legacy integration, and zero-downtime deployments.

### The Project

I was **consulting** on a team building payment feature for UK fintech company.

Our mission: Build a modern, cloud-native payment system compatible with decades-old legacy payment infrastructure.

> The catch? Payments are unforgiving. A transfer that fails halfway through is worse than one that never started. And our legacy banking system communicated through IBM MQ with messages encoded in formats older than most of our team members.

This is the story of how we built it.

### The Problem We Faced

On day one, the existing payment flow looked something like this:

```text
User clicks "Send £500"
    → API receives request
    → Validate account
    → Check fraud
    → Debit account
    → Send to core banking
    → Wait for confirmation
    → Credit recipient
    → Send notification
```

Simple enough on paper. But in production:
- **Step 4 fails**: Did the debit happen? Do we retry? Do we reverse it?
- **Step 5 times out**: The message went to IBM MQ but we never got a response. Is the payment processing? Did it fail silently?
- **Step 6 takes 3 days**: International SWIFT payments can take days. How do we track state across service restarts and deployments?

Our first attempt used a state machine with PostgreSQL. It worked — until it didn't. We had payments stuck in PROCESSING for weeks, no visibility into what went wrong, and engineers manually fixing database records at 2 AM.

We needed something better.

### Why Temporal

After evaluating several options (Cadence, Step Functions, hand-rolled sagas), we chose [Temporal](https://temporal.io). The selling points:
- **Durable execution**: Workflows survive process crashes, deployments, even cluster failures
- **Native Go SDK**: Our backend was Go, so this was a natural fit
- **Signal-based coordination**: Perfect for waiting on async responses from legacy systems
- **Built-in visibility**: The Temporal UI shows exactly where every workflow is stuck
- **Versioning**: Deploy new workflow code without breaking in-flight payments

Here's the architecture we ended up with:
![](https://cdn-images-1.medium.com/max/1024/1*xx0DQQX-wF9mtfpK531qRQ.png)![](https://cdn-images-1.medium.com/max/1024/1*nw4O5RR_T_EoShfomTVUVg.png)
### The Payment Workflow

Let me walk you through how we implemented a SEPA payment. In the UK, SEPA (Single Euro Payments Area) handles Euro-denominated transfers to European banks — common for businesses with EU suppliers.

The workflow definition looked like:

```go
package workflow

import (
	"fmt"
	"time"

	"go.temporal.io/sdk/temporal"
	"go.temporal.io/sdk/workflow"
)

// SEPAPaymentInput contains all data needed to process a SEPA payment
type SEPAPaymentInput struct {
	PaymentID       string
	CustomerID      string
	SourceAccountID string
	DestinationIBAN string
	DestinationBIC  string
	Amount          int64  // Amount in minor units (cents)
	Currency        string // EUR for SEPA
	Reference       string
	BeneficiaryName string
}

// SEPAPaymentOutput is the result of a completed payment
type SEPAPaymentOutput struct {
	PaymentID      string
	Status         string
	TransactionRef string
	CompletedAt    time.Time
}

// SEPAPayment orchestrates the complete payment lifecycle
func SEPAPayment(ctx workflow.Context, input SEPAPaymentInput) (*SEPAPaymentOutput, error) {
	logger := workflow.GetLogger(ctx)
	logger.Info("Starting SEPA payment workflow", "paymentID", input.PaymentID)

	// Configure activity options with sensible defaults
	activityOptions := workflow.ActivityOptions{
		StartToCloseTimeout: 10 * time.Minute,
		RetryPolicy: &temporal.RetryPolicy{
			InitialInterval:    time.Second,
			BackoffCoefficient: 2.0,
			MaximumInterval:    time.Minute,
			MaximumAttempts:    5,
		},
	}
	ctx = workflow.WithActivityOptions(ctx, activityOptions)

	// Step 1: Validate the payment request
	var validationResult ValidationResult
	err := workflow.ExecuteActivity(ctx, ValidatePayment, input).Get(ctx, &validationResult)
	if err != nil {
		return nil, fmt.Errorf("validation failed: %w", err)
	}

	// Step 2: Check if this payment requires approval
	if err := handleApprovalIfRequired(ctx, input); err != nil {
		return nil, err
	}

	// Step 3: Security checks - detect suspicious patterns
	blocked, err := checkSecurityAndBlock(ctx, input)
	if err != nil {
		return nil, err
	}
	if blocked {
		// Wait for manual review - workflow pauses here
		if err := waitForUnblock(ctx); err != nil {
			return nil, err
		}
	}

	// Step 4: Fraud evaluation (single attempt)
	fraudCtx := workflow.WithActivityOptions(ctx, workflow.ActivityOptions{
		StartToCloseTimeout: 30 * time.Second,
		RetryPolicy: &temporal.RetryPolicy{
			MaximumAttempts: 1, // No retries for fraud checks
		},
	})

	var fraudResult FraudCheckResult
	err = workflow.ExecuteActivity(fraudCtx, CheckFraud, input).Get(ctx, &fraudResult)
	if err != nil {
		return nil, fmt.Errorf("fraud check failed: %w", err)
	}

	if fraudResult.Rejected {
		return nil, temporal.NewNonRetryableApplicationError(
			"Payment rejected by fraud system",
			"FRAUD_REJECTED",
			nil,
		)
	}

	// Step 5: Check if user cancelled while we were processing
	if cancelled, _ := isPaymentCancelled(ctx, input.PaymentID); cancelled {
		return nil, temporal.NewNonRetryableApplicationError(
			"Payment cancelled by user",
			"USER_CANCELLED",
			nil,
		)
	}

	// Step 6: Mark payee as trusted for future payments
	_ = workflow.ExecuteActivity(ctx, MarkPayeeAsTrusted, input).Get(ctx, nil)

	// Step 7: Publish to legacy banking system via IBM MQ
	var mqResult MQPublishResult
	err = workflow.ExecuteActivity(ctx, PublishToLegacyBanking, input).Get(ctx, &mqResult)
	if err != nil {
		return nil, fmt.Errorf("failed to publish to legacy system: %w", err)
	}

	// Step 8: Wait for confirmation from legacy system (this is the magic)
	paymentResult, err := waitForLegacyConfirmation(ctx, input.PaymentID)
	if err != nil {
		return nil, err
	}

	logger.Info("SEPA payment completed successfully",
		"paymentID", input.PaymentID,
		"transactionRef", paymentResult.TransactionRef)

	return paymentResult, nil
}
```

### Waiting for Legacy Responses

The most interesting part of our implementation is Step 8. The legacy banking system processes payments asynchronously — it might take seconds, hours, or even days for international transfers. Traditional architectures would poll a database or use callbacks with complex state tracking.

With Temporal, we simply wait:

```go
// waitForLegacyConfirmation blocks the workflow until we receive
// a signal from the legacy system (via mq-listener)
func waitForLegacyConfirmation(ctx workflow.Context, paymentID string) (*SEPAPaymentOutput, error) {
	logger := workflow.GetLogger(ctx)

	// Setup signal channels
	paymentSignalChan := workflow.GetSignalChannel(ctx, SignalPaymentResult)
	transactionSignalChan := workflow.GetSignalChannel(ctx, SignalTransactionResult)

	var paymentResult PaymentSignal
	var transactionResult TransactionSignal
	paymentReceived := false
	transactionReceived := false

	// We might receive payment confirmation and transaction confirmation separately
	for !paymentReceived || !transactionReceived {
		selector := workflow.NewSelector(ctx)

		// Handle payment status signal
		selector.AddReceive(paymentSignalChan, func(ch workflow.ReceiveChannel, more bool) {
			ch.Receive(ctx, &paymentResult)
			logger.Info("Received payment signal", "status", paymentResult.Status)
			paymentReceived = true
		})

		// Handle transaction confirmation signal
		selector.AddReceive(transactionSignalChan, func(ch workflow.ReceiveChannel, more bool) {
			ch.Receive(ctx, &transactionResult)
			logger.Info("Received transaction signal", "ref", transactionResult.Reference)
			transactionReceived = true
		})

		// Add timeout - don't wait forever
		timer := workflow.NewTimer(ctx, 7*24*time.Hour) // 7 days max for SWIFT
		selector.AddFuture(timer, func(f workflow.Future) {
			logger.Warn("Payment confirmation timeout", "paymentID", paymentID)
		})

		selector.Select(ctx)

		// Check for errors in received signals
		if paymentReceived && paymentResult.Status == "FAILED" {
			return nil, temporal.NewNonRetryableApplicationError(
				paymentResult.ErrorMessage,
				"LEGACY_REJECTED",
				nil,
			)
		}
	}

	return &SEPAPaymentOutput{
		PaymentID:      paymentID,
		Status:         "COMPLETED",
		TransactionRef: transactionResult.Reference,
		CompletedAt:    workflow.Now(ctx),
	}, nil
}
```

Below is a summary of some temporal patterns used:
- We have used **workflow.GetLogger(ctx)**, which retrieves a deterministic logger tied to the workflow context that is replay safe. This logger is used for recording events like signal receipts, timeouts, or other important workflow milestones. ([link to workflow replays](https://docs.temporal.io/develop/go/observability)).
Ideally, this should only be used in **long-running workflow code** for logging to maintain determinism and integrate with Temporal's observability tools. It **should not** be used in non-workflow code, such as activities or plain Go functions, where standard loggers like log or zap suffice.
- We also use **workflow.GetSignalChannel(ctx, signalName)**, which creates a receive channel for external signals such as SignalPaymentResult or SignalTransactionResult. This allows workflows to listen for asynchronous updates from external systems without polling, with the channel state persisted across workflow suspensions ([link](https://docs.temporal.io/develop/go/message-passing)).
Signal channels are perfect for workflows that need to react to unpredictable events, like approvals or confirmations, ensuring durability even if the workflow pauses.
- For handling multiple asynchronous events, we use **workflow.NewSelector(ctx)** along with AddReceive, AddFuture, and Select. This lets us multiplex events like signals and timers, executing callbacks when any event fires, and provides deterministic concurrency similar to Go's native select but safe for workflows ([link](https://docs.temporal.io/develop/go/selectors)).
Selectors are ideal for complex orchestration, such as waiting for multiple signals or activities concurrently without busy loops or non-deterministic behavior. They are unnecessary for simple sequential logic, activities, or when using Future.Get suffices. Word of caution: Overusing selectors can make workflows harder to debug.
- To handle timeouts or bounded waits, we use **workflow.NewTimer(ctx, duration)**, which schedules a durable timer that persists even through outages. For example, a 7-day timer prevents the workflow from waiting indefinitely for signals ([link](https://docs.temporal.io/develop/go/timers)).
Timers are essential for workflow reliability, particularly for retries, scheduling, or waiting on external dependencies. Use these only for long er delays
- For controlled workflow failures, we use **temporal.NewNonRetryableApplicationError(message, type, details)**. This creates a custom error for non-retryable failures, such as a legacy system rejection, allowing the workflow to fail gracefully without triggering automatic retries ([link](https://docs.temporal.io/develop/go/error-handling)).
This should only be used in workflows (not activities) for business logic errors that should not retry, like invalid data or rejections, and allows attaching debugging details. For transient issues like network failures, retryable errors should be used instead.
- Finally, to handle consistent timestamps in workflows, we use **workflow.Now(ctx)**, which returns the current time from Temporal's perspective. This is deterministic and avoids inconsistencies caused by local clocks, making it ideal for recording events like CompletedAt or other workflow milestones ([link](https://docs.temporal.io/develop/go/core-application)).

### The Legacy Bridge: Connecting Two Worlds

The legacy banking system spoke a different language — literally. Messages were encoded in a proprietary mainframe format and transmitted via IBM MQ. We built a bridge service to handle the translation.
![](https://cdn-images-1.medium.com/max/858/1*WOCATBpkklwZzlgAkerakQ.png)

### Transforming Messages for the Legacy System

The legacy system expected messages in a specific tab-delimited format with fixed-width fields. Here's how we transformed our domain objects:

```go
package transform

import (
	"fmt"
	"strings"
	"time"
)

// LegacyPaymentMessage represents the format expected by the legacy system
type LegacyPaymentMessage struct {
	OLBReference     string // Unique reference we generate
	AccountBranch    string // 6 digits
	AccountBasic     string // 8 digits
	AccountSuffix    string // 3 digits
	PayeeAccountIBAN string
	PayeeBIC         string
	PayeeName        string // 35 chars max
	Amount           string // In minor units, no decimals
	Currency         string
	PaymentDate      string // YYYYMMDD
	Reference1       string // 35 chars max
	Reference2       string // 35 chars max
	Reference3       string // 35 chars max
	IsSEPA           string // Y/N
	ChargeType       string // SHA/OUR/BEN
}

// ToLegacyFormat converts a payment to the legacy tab-delimited format
func ToLegacyFormat(payment SEPAPaymentInput, olbRef string) []byte {
	msg := LegacyPaymentMessage{
		OLBReference:     olbRef,
		AccountBranch:    extractBranch(payment.SourceAccountID),
		AccountBasic:     extractBasic(payment.SourceAccountID),
		AccountSuffix:    extractSuffix(payment.SourceAccountID),
		PayeeAccountIBAN: payment.DestinationIBAN,
		PayeeBIC:         payment.DestinationBIC,
		PayeeName:        truncate(payment.BeneficiaryName, 35),
		Amount:           fmt.Sprintf("%d", payment.Amount),
		Currency:         payment.Currency,
		PaymentDate:      time.Now().Format("20060102"),
		Reference1:       splitReference(payment.Reference, 0),
		Reference2:       splitReference(payment.Reference, 1),
		Reference3:       splitReference(payment.Reference, 2),
		IsSEPA:           "Y",
		ChargeType:       "SHA", // Shared charges for SEPA
	}
	return msg.Encode()
}

// Encode converts the message to tab-delimited legacy format
func (m LegacyPaymentMessage) Encode() []byte {
	fields := []string{
		m.OLBReference,
		m.AccountBranch,
		m.AccountBasic,
		m.AccountSuffix,
		m.PayeeAccountIBAN,
		m.PayeeBIC,
		padRight(m.PayeeName, 35),
		m.Amount,
		m.Currency,
		m.PaymentDate,
		padRight(m.Reference1, 35),
		padRight(m.Reference2, 35),
		padRight(m.Reference3, 35),
		m.IsSEPA,
		m.ChargeType,
	}
	return []byte(strings.Join(fields, "\t"))
}

// splitReference breaks a long reference into 35-char chunks
func splitReference(ref string, index int) string {
	const chunkSize = 35
	start := index * chunkSize
	if start >= len(ref) {
		return ""
	}
	end := start + chunkSize
	if end > len(ref) {
		end = len(ref)
	}
	return ref[start:end]
}
```

### The MQ Publisher Service
The publisher was called to forward messages to IBM MQ:

```go
package publisher

import (
	"context"
	"fmt"
	"time"

	"github.com/ibm-messaging/mq-golang/v5/ibmmq"
)

type MQPublisher struct {
	queueManager *ibmmq.MQQueueManager
	queue        ibmmq.MQObject
	config       Config
}

type Config struct {
	Host         string
	Port         int
	QueueManager string
	Channel      string
	Queue        string
	User         string
	Password     string
}

func NewMQPublisher(cfg Config) (*MQPublisher, error) {
	cd := ibmmq.NewMQCD()
	cd.ChannelName = cfg.Channel
	cd.ConnectionName = fmt.Sprintf("%s(%d)", cfg.Host, cfg.Port)

	csp := ibmmq.NewMQCSP()
	csp.AuthenticationType = ibmmq.MQCSP_AUTH_USER_ID_AND_PWD
	csp.UserId = cfg.User
	csp.Password = cfg.Password

	cno := ibmmq.NewMQCNO()
	cno.ClientConn = cd
	cno.SecurityParms = csp
	cno.Options = ibmmq.MQCNO_CLIENT_BINDING

	qMgr, err := ibmmq.Connx(cfg.QueueManager, cno)
	if err != nil {
		return nil, fmt.Errorf("failed to connect to queue manager: %w", err)
	}

	od := ibmmq.NewMQOD()
	od.ObjectType = ibmmq.MQOT_Q
	od.ObjectName = cfg.Queue

	openOptions := ibmmq.MQOO_OUTPUT | ibmmq.MQOO_FAIL_IF_QUIESCING

	queue, err := qMgr.Open(od, openOptions)
	if err != nil {
		qMgr.Disc()
		return nil, fmt.Errorf("failed to open queue: %w", err)
	}

	return &MQPublisher{
		queueManager: &qMgr,
		queue:        queue,
		config:       cfg,
	}, nil
}

// Publish sends a message to the legacy system with retry logic
func (p *MQPublisher) Publish(ctx context.Context, message []byte) error {
	return p.publishWithRetry(ctx, message, 3)
}

func (p *MQPublisher) publishWithRetry(ctx context.Context, message []byte, maxRetries int) error {
	var lastErr error

	for attempt := 0; attempt < maxRetries; attempt++ {
		if attempt > 0 {
			select {
			case <-ctx.Done():
				return ctx.Err()
			case <-time.After(5 * time.Second):
			}
		}

		err := p.put(message)
		if err == nil {
			return nil
		}
		lastErr = err

		// Check if it's a connection error that requires reconnect
		if mqErr, ok := err.(*ibmmq.MQReturn); ok {
			if mqErr.MQRC == ibmmq.MQRC_CONNECTION_BROKEN {
				if reconnectErr := p.reconnect(); reconnectErr != nil {
					return fmt.Errorf("reconnect failed: %w", reconnectErr)
				}
				continue
			}
		}
		return err
	}

	return fmt.Errorf("failed after %d attempts: %w", maxRetries, lastErr)
}

func (p *MQPublisher) put(message []byte) error {
	md := ibmmq.NewMQMD()
	md.Format = ibmmq.MQFMT_STRING

	pmo := ibmmq.NewMQPMO()
	pmo.Options = ibmmq.MQPMO_NO_SYNCPOINT | ibmmq.MQPMO_NEW_MSG_ID

	return p.queue.Put(md, pmo, message)
}
```

### The MQ Listener: Signaling Workflows
The listener was the other half of the bridge. It consumed messages from IBM MQ and signaled the appropriate Temporal workflows:

```go
package listener

import (
	"context"
	"fmt"
	"time"

	"github.com/ibm-messaging/mq-golang/v5/ibmmq"
	"go.temporal.io/sdk/client"
)

type MQListener struct {
	queueManager   *ibmmq.MQQueueManager
	queue          ibmmq.MQObject
	temporalClient client.Client
	parser         *MessageParser
}

// Start begins listening for messages from the legacy system
func (l *MQListener) Start(ctx context.Context) error {
	logger := getLogger(ctx)
	logger.Info("Starting MQ listener")

	for {
		select {
		case <-ctx.Done():
			logger.Info("Shutting down MQ listener")
			return nil
		default:
			if err := l.processNextMessage(ctx); err != nil {
				logger.Error("Error processing message", "error", err)
				// Continue listening - don't crash on single message failure
			}
		}
	}
}

func (l *MQListener) processNextMessage(ctx context.Context) error {
	md := ibmmq.NewMQMD()
	gmo := ibmmq.NewMQGMO()
	gmo.Options = ibmmq.MQGMO_WAIT | ibmmq.MQGMO_FAIL_IF_QUIESCING
	gmo.WaitInterval = 10 * 1000 // 10 seconds

	buffer := make([]byte, 32768)
	datalen, err := l.queue.Get(md, gmo, buffer)
	if err != nil {
		if mqErr, ok := err.(*ibmmq.MQReturn); ok {
			if mqErr.MQRC == ibmmq.MQRC_NO_MSG_AVAILABLE {
				return nil // No message, this is normal
			}
		}
		return fmt.Errorf("failed to get message: %w", err)
	}

	message := buffer[:datalen]
	parsed, err := l.parser.Parse(message)
	if err != nil {
		return fmt.Errorf("failed to parse message: %w", err)
	}

	return l.signalWorkflow(ctx, parsed)
}

func (l *MQListener) signalWorkflow(ctx context.Context, msg ParsedMessage) error {
	var signalName string
	var signalData interface{}

	switch msg.Type {
	case MessageTypePaymentResult:
		signalName = "SignalPaymentResult"
		signalData = PaymentSignal{
			Status:       msg.Status,
			ErrorCode:    msg.ErrorCode,
			ErrorMessage: msg.ErrorMessage,
		}
	case MessageTypeTransactionConfirmation:
		signalName = "SignalTransactionResult"
		signalData = TransactionSignal{
			Reference:   msg.TransactionRef,
			CompletedAt: msg.Timestamp,
		}
	default:
		return fmt.Errorf("unknown message type: %s", msg.Type)
	}

	return l.signalWithRetry(ctx, msg.WorkflowID, signalName, signalData)
}

func (l *MQListener) signalWithRetry(ctx context.Context, workflowID, signalName string, data interface{}) error {
	maxRetries := 3

	for attempt := 0; attempt < maxRetries; attempt++ {
		err := l.temporalClient.SignalWorkflow(ctx, workflowID, "", signalName, data)
		if err == nil {
			return nil
		}

		// Check if workflow is already completed - that's OK
		if isWorkflowNotFoundError(err) {
			getLogger(ctx).Warn("Workflow not found, may already be completed",
				"workflowID", workflowID)
			return nil
		}

		if attempt < maxRetries-1 {
			time.Sleep(time.Second * time.Duration(attempt+1))
		}
	}

	return fmt.Errorf("failed to signal workflow after %d attempts", maxRetries)
}
```

### Handling Bulk Payments with the Saga Pattern
One of our more complex requirements was bulk payments — businesses uploading CSV files with hundreds of payments to process. We couldn't let a single failed payment bring down the entire batch.

### The Challenge

```text
CSV with 500 payments uploaded
    → Payment 1: Success
    → Payment 2: Success
    → ...
    → Payment 247: FAILS (insufficient funds)
    → What happens to payments 1-246?
    → What about 248-500?
```

### The Solution: Saga with Compensation

```go
package workflow

// BulkPaymentInput represents a batch of payments
type BulkPaymentInput struct {
	BatchID    string
	Payments   []SinglePaymentLine
	UploadedBy string
	UploadedAt time.Time
}

type SinglePaymentLine struct {
	LineNumber  int
	Reference   string
	PaymentData SEPAPaymentInput
}

type BulkPaymentResult struct {
	BatchID     string
	TotalLines  int
	Successful  int
	Failed      int
	FailedLines []FailedLine
}

type FailedLine struct {
	LineNumber int
	Reference  string
	Error      string
}

// BulkPaymentWorkflow processes a batch of payments with compensation
func BulkPaymentWorkflow(ctx workflow.Context, input BulkPaymentInput) (*BulkPaymentResult, error) {
	logger := workflow.GetLogger(ctx)
	logger.Info("Starting bulk payment workflow",
		"batchID", input.BatchID,
		"totalPayments", len(input.Payments))

	activityOptions := workflow.ActivityOptions{
		StartToCloseTimeout: 5 * time.Minute,
		RetryPolicy: &temporal.RetryPolicy{
			InitialInterval:    time.Second,
			BackoffCoefficient: 2.0,
			MaximumInterval:    100 * time.Second,
			MaximumAttempts:    5,
		},
	}
	ctx = workflow.WithActivityOptions(ctx, activityOptions)

	result := &BulkPaymentResult{
		BatchID:     input.BatchID,
		TotalLines:  len(input.Payments),
		FailedLines: []FailedLine{},
	}

	// Process each payment individually
	for _, line := range input.Payments {
		err := processPaymentLine(ctx, input.BatchID, line)
		if err != nil {
			logger.Warn("Payment line failed",
				"lineNumber", line.LineNumber,
				"error", err)

			// Compensate: Mark line as failed so it can be retried later
			compensateErr := compensateFailedLine(ctx, input.BatchID, line)
			if compensateErr != nil {
				logger.Error("Compensation failed",
					"lineNumber", line.LineNumber,
					"error", compensateErr)
			}

			result.Failed++
			result.FailedLines = append(result.FailedLines, FailedLine{
				LineNumber: line.LineNumber,
				Reference:  line.Reference,
				Error:      err.Error(),
			})

			// Continue processing remaining lines - don't fail the batch
			continue
		}

		result.Successful++
	}

	logger.Info("Bulk payment workflow completed",
		"batchID", input.BatchID,
		"successful", result.Successful,
		"failed", result.Failed)

	return result, nil
}

func processPaymentLine(ctx workflow.Context, batchID string, line SinglePaymentLine) error {
	// Update line status to PROCESSING
	err := workflow.ExecuteActivity(ctx, UpdateLineStatus, UpdateLineStatusInput{
		BatchID:    batchID,
		LineNumber: line.LineNumber,
		Status:     "PROCESSING",
	}).Get(ctx, nil)
	if err != nil {
		return fmt.Errorf("failed to update line status: %w", err)
	}

	// Execute the actual payment
	var paymentResult PaymentLineResult
	err = workflow.ExecuteActivity(ctx, ExecutePaymentLine, line.PaymentData).Get(ctx, &paymentResult)
	if err != nil {
		return err
	}

	// Update line status to COMPLETED
	return workflow.ExecuteActivity(ctx, UpdateLineStatus, UpdateLineStatusInput{
		BatchID:       batchID,
		LineNumber:    line.LineNumber,
		Status:        "COMPLETED",
		TransactionID: paymentResult.TransactionID,
	}).Get(ctx, nil)
}

// compensateFailedLine reverts the line status so it can be retried
func compensateFailedLine(ctx workflow.Context, batchID string, line SinglePaymentLine) error {
	return workflow.ExecuteActivity(ctx, RevertLineStatus, RevertLineStatusInput{
		BatchID:    batchID,
		LineNumber: line.LineNumber,
		Status:     "FAILED",
	}).Get(ctx, nil)
}
```

**Compensation doesn't mean rollback**. For payments, we can't magically un-send money. Instead, compensation means:
- Recording that the line failed
- Preserving enough state to retry or investigate later
- Not letting one failure cascade to others

### Workflow Versioning: Deploying Without Fear

One of the scariest parts of our initial design discussions was: "How do we deploy new code when we have payments running for days?"

Temporal's versioning solved this elegantly:

```go
func SEPAPayment(ctx workflow.Context, input SEPAPaymentInput) (*SEPAPaymentOutput, error) {
	// ... skipped steps for brevity ...

	// We added activity tracking in a later release
	// Use versioning to handle in-flight workflows
	activityTrackingVersion := workflow.GetVersion(ctx,
		"activity-tracking-v1",  // Unique identifier for this change
		workflow.DefaultVersion, // Version 0: old code path
		1,                       // Version 1: new code path
	)

	if activityTrackingVersion >= 1 {
		// New path: Send status updates for real-time UI
		_ = workflow.ExecuteActivity(ctx, PublishStatusUpdate, StatusUpdate{
			PaymentID: input.PaymentID,
			Status:    "PROCESSING",
			Step:      "FRAUD_CHECK",
		}).Get(ctx, nil)
	}

	// ... continue with fraud check ...

	// Another versioned change: we switched from signal-based approvals
	// to direct API calls
	approvalVersion := workflow.GetVersion(ctx,
		"approval-api-v2",
		workflow.DefaultVersion,
		2,
	)

	if approvalVersion >= 2 {
		// V2: Direct approval check
		err := workflow.ExecuteActivity(ctx, CheckApprovalDirect, input).Get(ctx, nil)
		if err != nil {
			return nil, err
		}
	} else if approvalVersion == 1 {
		// V1: Signal-based approval (deprecated)
		err := waitForApprovalSignal(ctx, input.PaymentID)
		if err != nil {
			return nil, err
		}
	}

	// ... rest of workflow ...
}
```

How it works
![](https://cdn-images-1.medium.com/max/1024/1*Dad610moqfSAODnDNhyQjA.png)

The version is persisted in the workflow history, so even after code changes, old workflows execute exactly as they did originally.

### Error Handling: Knowing When NOT to Retry

Not all errors are created equal. Retrying "network timeout" makes sense. Retrying "insufficient funds" is pointless. We categorized errors carefully:

```go
package activities

import (
	"context"
	"errors"
	"fmt"

	"go.temporal.io/sdk/temporal"
)

// Custom error types for business logic failures
var (
	ErrInsufficientFunds  = errors.New("insufficient funds")
	ErrAccountFrozen      = errors.New("account frozen")
	ErrInvalidBeneficiary = errors.New("invalid beneficiary")
	ErrDuplicatePayment   = errors.New("duplicate payment detected")
	ErrFraudRejected      = errors.New("rejected by fraud system")
	ErrUserCancelled      = errors.New("cancelled by user")
)

// ValidatePayment checks if a payment can proceed
func ValidatePayment(ctx context.Context, input SEPAPaymentInput) (ValidationResult, error) {
	// Check account balance
	balance, err := accountsClient.GetBalance(ctx, input.SourceAccountID)
	if err != nil {
		// Network error - Temporal should retry
		return ValidationResult{}, fmt.Errorf("failed to get balance: %w", err)
	}

	if balance < input.Amount {
		// Business error - don't retry, it won't help
		return ValidationResult{}, temporal.NewNonRetryableApplicationError(
			fmt.Sprintf("Insufficient funds: available %d, required %d", balance, input.Amount),
			"INSUFFICIENT_FUNDS",
			ErrInsufficientFunds,
		)
	}

	// Check account status
	account, err := accountsClient.GetAccount(ctx, input.SourceAccountID)
	if err != nil {
		return ValidationResult{}, fmt.Errorf("failed to get account: %w", err)
	}

	if account.Status == "FROZEN" {
		return ValidationResult{}, temporal.NewNonRetryableApplicationError(
			"Account is frozen",
			"ACCOUNT_FROZEN",
			ErrAccountFrozen,
		)
	}

	// Validate IBAN format
	if !isValidIBAN(input.DestinationIBAN) {
		return ValidationResult{}, temporal.NewNonRetryableApplicationError(
			fmt.Sprintf("Invalid IBAN format: %s", input.DestinationIBAN),
			"INVALID_IBAN",
			ErrInvalidBeneficiary,
		)
	}

	return ValidationResult{Valid: true}, nil
}

// CheckFraud evaluates the payment for fraud risk
func CheckFraud(ctx context.Context, input SEPAPaymentInput) (FraudCheckResult, error) {
	result, err := fraudService.Evaluate(ctx, FraudRequest{
		CustomerID: input.CustomerID,
		Amount:     input.Amount,
		Recipient:  input.DestinationIBAN,
	})
	if err != nil {
		// Fraud service down - this is configured with MaxAttempts: 1
		// so we'll fail rather than proceed without fraud check
		return FraudCheckResult{}, fmt.Errorf("fraud service unavailable: %w", err)
	}

	if result.Decision == "REJECT" {
		// Fraud rejection is a result, not an error
		return FraudCheckResult{
			Rejected: true,
			Reason:   result.Reason,
		}, nil
	}

	return FraudCheckResult{
		Rejected: false,
		Score:    result.RiskScore,
	}, nil
}
```

Here is how the prod setup looked like:

```go
package config

import (
	"time"

	"go.temporal.io/sdk/temporal"
	"go.temporal.io/sdk/worker"
)

const (
	PaymentTaskQueue = "payments-task-queue"

	// SWIFT payments can take days to settle
	WorkflowRunTimeout  = 7 * 24 * time.Hour
	WorkflowTaskTimeout = 10 * time.Minute

	DefaultActivityTimeout = 10 * time.Minute
	FraudCheckTimeout      = 30 * time.Second
	LegacyPublishTimeout   = 5 * time.Minute
)

// WorkerOptions returns production worker configuration
func WorkerOptions() worker.Options {
	return worker.Options{
		MaxConcurrentWorkflowTaskExecutionSize: 50,
		MaxConcurrentActivityExecutionSize:     100,
		MaxConcurrentWorkflowTaskPollers:       4,
		MaxConcurrentActivityTaskPollers:       8,
		StickyScheduleToStartTimeout:           5 * time.Second,
	}
}

// DefaultRetryPolicy for most activities
func DefaultRetryPolicy() *temporal.RetryPolicy {
	return &temporal.RetryPolicy{
		InitialInterval:    time.Second,
		BackoffCoefficient: 2.0,
		MaximumInterval:    time.Minute,
		MaximumAttempts:    5,
	}
}

// NoRetryPolicy for activities that shouldn't retry
func NoRetryPolicy() *temporal.RetryPolicy {
	return &temporal.RetryPolicy{
		MaximumAttempts: 1,
	}
}
```

Worker Setup:

```go
package main

import (
	"log"
	"os"
	"os/signal"
	"syscall"

	"go.temporal.io/sdk/client"
	"go.temporal.io/sdk/worker"
)

func main() {
	c, err := client.Dial(client.Options{
		HostPort:  os.Getenv("TEMPORAL_HOST"),
		Namespace: os.Getenv("TEMPORAL_NAMESPACE"),
	})
	if err != nil {
		log.Fatalf("Failed to create Temporal client: %v", err)
	}
	defer c.Close()

	w := worker.New(c, config.PaymentTaskQueue, config.WorkerOptions())

	// Register workflows
	w.RegisterWorkflow(workflow.SEPAPayment)
	w.RegisterWorkflow(workflow.SWIFTPayment)
	w.RegisterWorkflow(workflow.DomesticPayment)
	w.RegisterWorkflow(workflow.BulkPaymentWorkflow)
	w.RegisterWorkflow(workflow.StandingOrderWorkflow)

	// Register activities
	registerActivities(w)

	go func() {
		if err := w.Run(worker.InterruptCh()); err != nil {
			log.Fatalf("Worker failed: %v", err)
		}
	}()

	quit := make(chan os.Signal, 1)
	signal.Notify(quit, syscall.SIGINT, syscall.SIGTERM)
	<-quit
	log.Println("Shutting down worker...")
	w.Stop()
}

func registerActivities(w worker.Worker) {
	accountAct := activities.NewAccountActivities(accountsClient)
	w.RegisterActivity(accountAct.GetAccount)
	w.RegisterActivity(accountAct.GetBalance)
	w.RegisterActivity(accountAct.DebitAccount)

	fraudAct := activities.NewFraudActivities(fraudClient)
	w.RegisterActivity(fraudAct.CheckFraud)

	mqAct := activities.NewMQActivities(mqPublisher)
	w.RegisterActivity(mqAct.PublishToLegacyBanking)

	validationAct := activities.NewValidationActivities(validationService)
	w.RegisterActivity(validationAct.ValidatePayment)

	statusAct := activities.NewStatusActivities(statusService)
	w.RegisterActivity(statusAct.UpdatePaymentStatus)
	w.RegisterActivity(statusAct.PublishStatusUpdate)
}
```

### Summary of Lessons Learned
After running this in production, here's a summary of what we learned:

#### 1. Design Workflows Around Business Time, Not Computer Time

Our first instinct was to make workflows as short as possible. But payments don't work that way — a SWIFT transfer might take 3 business days. We learned to embrace long-running workflows:

```go
// DON'T: Arbitrary timeout that doesn't match business reality
WorkflowRunTimeout: 1 * time.Hour

// DO: Match the business process
WorkflowRunTimeout: 7 * 24 * time.Hour // International payments can take days
```

#### 2. Signals Are Not Guaranteed Delivery
We initially assumed signals always reached their workflows. They don't — if the workflow completes or fails before the signal arrives, it's lost. We added idempotency checks:

```go
err := temporalClient.SignalWorkflow(ctx, workflowID, "", signalName, data)
if err != nil {
	if isWorkflowNotFoundError(err) {
		// Workflow already completed - log and move on
		logger.Info("Workflow already completed, signal dropped",
			"workflowID", workflowID)
		return nil
	}
	return err
}
```

#### 3. Version Early, Version Often
We added our first version check only after we had a bug in production. Now we add version checks proactively, for prominent changes:

```go
v := workflow.GetVersion(ctx, "change-identifier", workflow.DefaultVersion, 1)
if v >= 1 {
	// New behavior
} else {
	// Old behavior (for in-flight workflows)
}
```

#### 4. Non-Retryable Errors Need Clear Communication
When we return a non-retryable error, the workflow fails permanently. Users need to understand why:

```go
// DON'T: Vague error
return temporal.NewNonRetryableApplicationError("failed", "ERROR", nil)

// DO: Clear, actionable error
return temporal.NewNonRetryableApplicationError(
	fmt.Sprintf("Payment rejected: Account %s has insufficient funds. "+
		"Available: £%.2f, Required: £%.2f",
		accountID, float64(balance)/100, float64(amount)/100),
	"INSUFFICIENT_FUNDS",
	ErrInsufficientFunds,
)
```

Building durable payment workflows with Temporal transformed how we approached distributed systems. The patterns we developed — signal-based coordination, saga compensation, workflow versioning, and the legacy bridge — are now foundational to our architecture.

The key insight isn't about Temporal specifically. It's about matching your technical architecture to your business reality. Payments are inherently long-running, cross multiple systems, and need to survive failures. Traditional request-response architectures fight this reality. Workflow orchestration embraces it.

If you're building payment systems — or any long-running business process — I'd encourage you to explore workflow orchestration. The investment in understanding these patterns pays dividends in reliability, visibility, and developer sanity.
