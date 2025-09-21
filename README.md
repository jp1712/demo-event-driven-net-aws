# Event-Driven Architecture in .NET with AWS

A comprehensive example demonstrating real-world event-driven architecture patterns using .NET 8, AWS Lambda, SNS, and SQS. This repository accompanies the blog post "Event-Driven Architecture in .NET: Lessons Learned with AWS Lambda, SNS & SQS" and provides practical solutions to common production challenges.

## 🎯 What You'll Learn

This repository demonstrates 8 critical patterns every event-driven system needs:

1. **Handling Duplicate Events & Idempotency** - Prevent double processing with check-then-write patterns
2. **Event Ordering Under High Concurrency** - Use FIFO queues and MessageGroupId for strict ordering
3. **Schema Evolution & Backward Compatibility** - Version events safely without breaking consumers
4. **Dead-Letter Queues & Poison-Pill Events** - Handle failures gracefully with proper DLQ management
5. **Monitoring, Tracing & Observability** - Track events across distributed services with X-Ray
6. **Event Replay & Historical Data Recovery** - Safely reprocess events for bug fixes and analytics
7. **Handling Large Payloads with S3 Pointer** - Overcome SNS/SQS size limits using S3 storage
8. **Filtering with SNS Attributes** - Route only relevant events to reduce costs and noise

## 🏗️ Architecture Overview

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│   Orders    │───▶│  SNS Topic  │───▶│   Lambda    │
│   Service   │    │ order-events│    │ Processors  │
└─────────────┘    └─────────────┘    └─────────────┘
                           │                   │
                           ▼                   ▼
                   ┌─────────────┐    ┌─────────────┐
                   │ SQS Queues  │    │ DynamoDB /  │
                   │ (FIFO/Std)  │    │ S3 Storage  │
                   └─────────────┘    └─────────────┘
```

## 🚀 Quick Start

### Prerequisites

- .NET 8 SDK
- AWS CLI configured with appropriate permissions
- AWS account with access to SNS, SQS, Lambda, S3, and DynamoDB

### AWS Resources Setup

Before running the examples, create these AWS resources:

#### SNS Topics
```sh
# Standard topic for high-throughput events
aws sns create-topic --name order-events

# FIFO topic for ordered events
aws sns create-topic --name order-events.fifo --attributes FifoTopic=true,ContentBasedDeduplication=true
```

#### SQS Queues
```sh
# Standard queue
aws sqs create-queue --queue-name order-events-queue

# FIFO queue
aws sqs create-queue --queue-name order-events-queue.fifo --attributes FifoQueue=true,ContentBasedDeduplication=true

# Dead Letter Queue
aws sqs create-queue --queue-name order-events-dlq
```

#### S3 Bucket
```sh
# For large payload storage
aws s3 mb s3://event-payloads-[your-suffix]
```

#### DynamoDB Table
```sh
# For idempotency tracking
aws dynamodb create-table --table-name processed-events \
  --attribute-definitions AttributeName=EventId,AttributeType=S \
  --key-schema AttributeName=EventId,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST
```

### Running the Examples

1. **Clone and build:**
```sh
git clone https://github.com/your-org/event-driven-net-aws
cd DemoEventDrivenSolution
dotnet build
```

2. **Run specific examples:**
```sh
# Navigate to examples project
cd src/Examples

# Run idempotency example
dotnet run idempotency

# Run ordering example
dotnet run ordering

# Run schema evolution example
dotnet run schema

# Run dead letter queue example
dotnet run dlq

# Run tracing example
dotnet run tracing

# Run event replay example
dotnet run replay

# Run large payloads example
dotnet run largepayloads

# Run event filtering example
dotnet run filtering
```

3. **View available examples:**
```sh
dotnet run
```

## 📁 Project Structure

```
DemoEventDrivenSolution/
├── src/
│   ├── Shared/                          # Shared models and interfaces
│   │   ├── Models/
│   │   │   └── OrderEvents.cs           # Event models and versioning
│   │   ├── Interfaces/
│   │   │   └── IServices.cs             # Service interfaces
│   │   └── EventDriven.Shared.csproj
│   └── Examples/                        # Example implementations
│       ├── Idempotency/
│       │   └── IdempotencyHandler.cs    # Problem 1: Duplicate handling
│       ├── Ordering/
│       │   └── FifoQueueSender.cs       # Problem 2: Message ordering
│       ├── SchemaEvolution/
│       │   └── EventSchemaEvolution.cs  # Problem 3: Event versioning
│       ├── DeadLetterQueue/
│       │   └── DeadLetterQueueSetup.cs  # Problem 4: Failure handling
│       ├── Tracing/
│       │   └── TracingPublisher.cs      # Problem 5: Observability
│       ├── EventReplay/
│       │   └── EventReplayService.cs    # Problem 6: Historical processing
│       ├── LargePayloads/
│       │   └── LargePayloadHandler.cs   # Problem 7: S3 pointer pattern
│       ├── EventFiltering/
│       │   └── EventFilteringPublisher.cs # Problem 8: SNS filtering
│       ├── Program.cs                   # Main entry point
│       └── EventDriven.Examples.csproj
├── DemoEventDrivenSolution.sln
└── README.md
```

## 🛠️ Example Patterns

### 1. Idempotency Pattern
```csharp
public async Task HandleAsync(OrderCreatedEvent order)
{
    // Check if already processed
    if (await _idempotencyStore.ExistsAsync(order.OrderId))
        return; // Skip duplicate

    // Process the event
    await _paymentService.ChargeAsync(order);
    
    // Mark as processed
    await _idempotencyStore.MarkProcessedAsync(order.OrderId);
}
```

### 2. FIFO Ordering Pattern
```csharp
await _sqsClient.SendMessageAsync(new SendMessageRequest
{
    QueueUrl = FifoQueueUrl,
    MessageBody = JsonSerializer.Serialize(order),
    MessageGroupId = order.OrderId.ToString(), // Ensures per-order ordering
    MessageDeduplicationId = Guid.NewGuid().ToString()
});
```

### 3. S3 Pointer Pattern
```csharp
if (payloadSize > 200000) // SNS limit
{
    // Store in S3
    var s3Key = $"orders/{order.OrderId}.json";
    await _s3Client.PutObjectAsync(new PutObjectRequest
    {
        BucketName = "event-payloads",
        Key = s3Key,
        ContentBody = payloadJson
    });
    
    // Publish pointer
    var pointer = new S3PayloadPointer(order.OrderId, "LargeOrder", "event-payloads", s3Key);
    await _snsClient.PublishAsync(CreatePointerMessage(pointer));
}
```

## 📊 Problem-Solution Mapping

| Example | Blog Section | Description | Key Pattern |
|---------|-------------|-------------|-------------|
| `IdempotencyHandler.cs` | 1. Duplicate Event Processing | Prevent double processing | Check-then-write |
| `FifoQueueSender.cs` | 2. Event Ordering | Guarantee message sequence | MessageGroupId |
| `EventSchemaEvolution.cs` | 3. Schema Evolution | Backward compatibility | Event versioning |
| `DeadLetterQueueSetup.cs` | 4. Dead Letter Queues | Handle poison pills | DLQ management |
| `TracingPublisher.cs` | 5. Monitoring & Tracing | End-to-end visibility | Trace ID propagation |
| `EventReplayService.cs` | 6. Event Replay | Historical reprocessing | Archive and republish |
| `LargePayloadHandler.cs` | 7. Large Payloads | Handle size limits | S3 pointer pattern |
| `EventFilteringPublisher.cs` | 8. Event Filtering | Selective routing | SNS message attributes |

## 🧪 Testing

Each example includes:
- ✅ **Mock implementations** for local testing
- ✅ **Console output** showing pattern execution
- ✅ **Error handling** demonstrations
- ✅ **AWS service interaction** examples

To test without AWS resources, the examples will show the pattern logic with mock data.

## 📚 Key Takeaways

| Pattern | Key Benefit | When to Use | Production Impact |
|---------|-------------|-------------|-------------------|
| Idempotency | Prevents duplicate processing | Always - non-negotiable | Prevents financial loss |
| FIFO Ordering | Guarantees event sequence | When order matters | Ensures data consistency |
| Schema Evolution | Backward compatibility | When events change | Prevents breaking changes |
| Dead Letter Queues | Failure isolation | Always - for resilience | Improves system reliability |
| Distributed Tracing | End-to-end visibility | Debugging distributed systems | Reduces MTTR |
| Event Replay | Historical reprocessing | Bug fixes, analytics | Enables safe recovery |
| S3 Pointer Pattern | Large payload handling | When events exceed 256KB | Overcomes size limits |
| SNS Filtering | Selective routing | Reduce noise and costs | Optimizes resource usage |

## 🎯 Production Considerations

### Cost Optimization
- Monitor SNS fan-out and Lambda invocations
- Use SNS filtering to reduce unnecessary Lambda executions
- Implement appropriate DLQ retention periods

### Security
- Enable KMS encryption for sensitive data
- Use IAM roles with least privilege
- Implement VPC endpoints for private communication

### Monitoring & Alerting
- Set up CloudWatch alarms for DLQ depth
- Monitor Lambda error rates and duration
- Track message processing latency

### Performance
- Use long polling and appropriate batch sizes
- Configure Lambda concurrency limits
- Optimize payload sizes

## 🔧 Configuration

Update these configuration values in the respective example files:

```csharp
// Update with your AWS resources
private const string TopicArn = "arn:aws:sns:us-east-1:YOUR-ACCOUNT:orders";
private const string QueueUrl = "https://sqs.us-east-1.amazonaws.com/YOUR-ACCOUNT/orders";
private const string BucketName = "your-event-payloads-bucket";
```

## 📖 Related Blog Post

This repository accompanies the detailed blog post: **"Event-Driven Architecture in .NET: Lessons Learned with AWS Lambda, SNS & SQS"**

The blog covers:
- Real-world challenges in production
- Step-by-step AWS resource setup
- Detailed explanations of each pattern
- Performance and cost considerations
- Lessons learned from production incidents

## 🏃‍♂️ Getting Started Quickly

If you want to see the patterns in action immediately:

```sh
# 1. Clone the repo
git clone https://github.com/your-org/event-driven-net-aws
cd DemoEventDrivenSolution

# 2. Build the solution
dotnet build

# 3. Run the idempotency example (works without AWS)
cd src/Examples
dotnet run idempotency

# 4. See all available examples
dotnet run
```

## 🤝 Contributing

Contributions are welcome! Please:
1. Fork the repository
2. Create a feature branch (`git checkout -b feature/amazing-pattern`)
3. Add tests for new patterns
4. Commit your changes (`git commit -m 'Add amazing pattern'`)
5. Push to the branch (`git push origin feature/amazing-pattern`)
6. Open a Pull Request

### Development Setup
- Ensure .NET 8 SDK is installed
- Run `dotnet restore` to install dependencies
- Follow existing code patterns and documentation style

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🆘 Support & Community

- 📚 **Documentation**: Check the inline code comments and XML documentation
- 🐛 **Bug Reports**: [Open an issue](https://github.com/your-org/event-driven-net-aws/issues) with reproduction steps
- 💡 **Questions**: [Start a discussion](https://github.com/your-org/event-driven-net-aws/discussions) for architectural questions
- 💬 **Community**: Share your experiences in the blog post comments

## 🎉 Acknowledgments

- AWS Documentation for comprehensive service guides
- .NET Community for architectural patterns and best practices
- Contributors who helped refine these patterns through real-world usage

---

⭐ **Star this repo** if it helped you build better event-driven systems!

🔄 **Share** with your team to spread event-driven architecture best practices!

---

*Built with ❤️ for the .NET and AWS community*
