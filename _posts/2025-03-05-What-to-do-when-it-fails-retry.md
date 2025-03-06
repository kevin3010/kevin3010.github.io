---
layout: post
title:  "What to do when it fails? Retry"
date:   2025-03-05
categories: design-patterns
---
# Retry Mechanisms and Distributed Systems

Recently I came across a facinating piece of code that used design pattern and generics to implement an important helper function-Retry. In distributed systems, failures are inevitable. Networks falter, services time out, and dependencies become temporarily unavailable. And intrestingly the solution to a lot of these problem is just to retry. Many failures are simply **recoverable** — they resolve themselves given time. 


## Designing an Effective Retry Mechanism

A well-designed retry utility should:

1. **Be reusable** — wrap around any code that may need retries
2. **Support selective failure handling** — retry only specific types of failures 
3. **Allow configurable retry strategies** — different scenarios require different approaches

## Implementation with the Strategy Pattern


### The RetryHandler

```typescript
class RetryHandler {
    async executeWithRetry<R>(
        action: () => R, 
        retryOnExceptions: Array<ErrorConstructor>, 
        strategy: Strategy
    ): Promise<R> {
        // Delegate the retry logic to the strategy
        return strategy.executeWithRetry(action, retryOnExceptions);
    }
}
```

This handler acts as a facade, accepting:
- An `action` function to execute
- An array of exception types to retry on
- A retry strategy

### The Strategy Interface

```typescript
interface Strategy {
  executeWithRetry<R>(
    action: () => R, 
    retryOnExceptions: Array<ErrorConstructor>
  ): Promise<R>;
}
```

### Exponential Backoff Strategy

One of the most common retry approaches is exponential backoff, where the delay between retries increases exponentially:

```typescript
class ExponentialBackoffStrategy implements Strategy {
    constructor(
        public maxRetries: number, 
        public initialDelay: number, 
        public multiplier: number
    ) {}

    // Method to handle retry logic
    async executeWithRetry<R>(
        action: () => R, 
        retryOnExceptions: Array<ErrorConstructor>
    ): Promise<R> {
        let attempts = 0;
        let delay = this.initialDelay;

        while (attempts < this.maxRetries) {
            try {
                // Attempt to execute the logic 
                return action();
            } catch (e) {
                if (e instanceof Error) {

                    // If failure occurs check if it the one that should be retried
                    const shouldRetry = retryOnExceptions.some(ex => e instanceof ex);
                    if (!shouldRetry) {
                        throw e;
                    }
                    
                    attempts++;
                    if (attempts >= this.maxRetries) {
                        throw e;
                    }

                    // Suspend the while loop for some time and then try again
                    await this.sleep(delay);
                    delay *= this.multiplier;
                }
            }
        }

        throw new Error("Retry attempts exhausted");
    }

    // Helper function to simulate delay
    private sleep(ms: number): Promise<void> {
        return new Promise(resolve => setTimeout(resolve, ms));
    }
}
```

The strategy:
1. Attempts to execute the action
2. Catches any errors and checks if they're retryable
3. Calculates an increasing delay between attempts
4. Retries until successful or max attempts reached

## Usage Example

Here's how to use our retry mechanism:

```typescript
// Custom error class
class RuntimeError extends Error {
    constructor(message: string) {
        super(message);
        this.name = 'RuntimeError';
    }
}

async function main() {
    const retryHandler = new RetryHandler();
    const strategy = new ExponentialBackoffStrategy(3, 1000, 2);
    const exceptionsToRetry = [RuntimeError];

    try {
        const result = await retryHandler.executeWithRetry(() => {
            console.log("Attempting operation...");
            if (Math.random() < 0.7) { // Simulate failure
                throw new RuntimeError("Operation failed");
            }
            return "Success";
        }, exceptionsToRetry, strategy);

        console.log("Operation result: " + result);
    } catch (e) {
        console.log("Operation failed after retries: " + 
            (e instanceof Error ? e.message : "Unknown error"));
    }
}

main();
```

In this example:
- I create a handler and an exponential backoff strategy
- Specify that only `RuntimeError` should trigger retries
- Our action deliberately fails 70% of the time
- Retry up to 3 times with increasing delays

## Possible Extensions

### 1. Jitter

Adding randomness to retry delays helps prevent "thundering herd" problems where many clients retry simultaneously:

```typescript
// Add jitter by randomizing the delay slightly
delay = delay * (0.5 + Math.random());
```

### 2. Circuit Breaker Pattern

Combine retries with a circuit breaker to fail fast when a service is completely down:

```typescript
if (circuitBreaker.isOpen()) {
    throw new Error("Circuit breaker is open, service unavailable");
}
```

### 3. Retry Budgets

Limit the total number of retries across your system to prevent cascading failures:

```typescript
if (!retryBudget.canRetry()) {
    throw new Error("Retry budget exhausted");
}
```

## References 

https://github.com/spring-projects/spring-retry