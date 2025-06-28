---
title: "Rust and C Walk Into a Lambda: Custom Runtimes for the Performance-Hungry DevOps Engineer"
description: "Because AWS Lambda isn't just a service - it's a platform waiting to be mastered!"
category: "AWS Lambda"
image: "/custom-runtime.png"
date: "January 12, 2025"
author: "Yogesh Patil"
---


When it comes to AWS Lambda, developers often stick to the well-trodden paths of Node.js, Python, or Java. But as a DevOps engineer, you know those paths can get a little crowded. Sometimes, you need a trailblazing solution — a custom runtime.

Today, we're diving into the fascinating world of AWS Lambda custom runtimes using Rust and C, two low-level performance powerhouses. Why these languages? Because sometimes, we need precision, efficiency, and a little "look what I just optimized" flex.

And yes, we'll also answer the age-old battle cry: "Why not just use EC2?"

## Why Custom Runtimes (and Not EC2)?

Imagine a conversation between Dev and DevOps:

**Dev:** "Can't we just deploy this on EC2?"  
**DevOps:** "Sure, if you want to spend your weekends patching instances and scaling servers."

Lambda custom runtimes are like the work smarter, not harder mantra of serverless computing. Here's why they shine:

### 1. No Infrastructure Hassles
With EC2, you're managing instances. With containers, you're managing orchestration. With Lambda, you're managing… well, nothing. AWS does all the heavy lifting — patching, scaling, and uptime are handled for you.

### 2. Support for Legacy Code
Have old C-based libraries that you don't want to rewrite? Custom runtimes let you use them directly without the overhead of running a VM or container. Perfect for tasks like daily processing jobs or one-off operations.

### 3. Event-Driven Simplicity
Custom runtimes let you trigger legacy or specialized code when needed without always-on infrastructure. Ideal for workflows that process occasional large files or periodic reports.

### 4. Tight Execution Time and Cost Control
In the Lambda world, you pay for execution time and memory usage. Optimized custom runtimes ensure you only pay for what you use. They're a great fit for performance-critical tasks where every millisecond and megabyte counts.

*(These are my thoughts; yours may differ. Feel free to share your perspective!)*

## Why Rust and C?

If DevOps were a movie, Rust would be the brilliant hacker who secures the system, and C would be the old-school action hero who gets things done with brute force.

### Rust: The Memory-Safe Maverick
- **Concurrency without Fear:** Rust prevents race conditions like a pro.
- **Blazing Fast:** Designed for high-performance, low-latency workloads.

### C: The Veteran Performer
- **No-nonsense Control:** You decide exactly how memory is allocated and released.
- **Proven Track Record:** If there's an optimization challenge, C has been solving it since the '70s.

### The Dream Team
Rust handles your application logic, while C powers those heavy computations. Together, they deliver speed, efficiency, and that satisfying feeling of squeezing every last drop out of your CPU.

## Use Cases for Custom Runtimes with Rust and C

Still wondering when to go all-in on custom runtimes? Here's a quick cheat sheet:

- **Real-Time Data Processing:** Processing high-frequency sensor or financial data? Rust's concurrency model and C's speed make it seamless.
- **Edge Computing:** Deploying Lambda at the edge? Rust and C reduce resource usage, making them ideal for IoT and edge workloads.
- **Legacy Code Execution:** Have critical C libraries you can't rewrite? Custom runtimes let you run them as-is, triggered by Lambda events.
- **Performance-Driven APIs:** Need to handle millions of requests per second? Rust and C keep your Lambda snappy and your costs low.
- **Heavy-Duty Math:** Building scientific models or crunching AI data? Optimize those calculations like a pro.

## Building a Custom Runtime with Rust and C: Step-by-Step Guide

### Folder Structure Before You Begin

Here's the folder structure we'll create for this project. Follow this layout to avoid confusion as you progress through the steps:

```
rust_runtime/
├── bootstrap                # Entry point for AWS Lambda
├── src/
│   ├── main.rs          # Rust application logic
│   ├── complax_math.c   # C code for custom functionality
└── Cargo.toml           # Rust project configuration
```

### Prerequisites Installation (Step 0)

First, let's install all the tools we need:

1. **Install Rust:**
```bash
# For Windows, download from https://rustup.rs/
# For Linux/Mac:
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source $HOME/.cargo/env
```

2. **Install C Compiler:**
- Windows: Install MinGW-w64 from http://mingw-w64.org/
- Linux: `sudo apt-get install build-essential`
- Mac: `xcode-select --install`

3. **Install AWS CLI:**
- Download from: https://aws.amazon.com/cli/
- Verify installation: `aws --version`

4. **Install Docker:**
- Download from: https://www.docker.com/products/docker-desktop
- Verify installation: `docker --version`

### Project Setup (Step 1)

1. **Create Project Directory:**
```bash
cargo new rust_lambda
cd rust_lambda
```

2. **Create Project Structure:**
```bash
mkdir src
touch src/main.rs
touch src/complex_math.c
touch build.rs
```

3. **Set Up Dependencies:** Create/update `Cargo.toml`:
```toml
[package]
name = "rust_lambda"
version = "0.1.0"
edition = "2021"

[dependencies]
lambda_runtime = "0.7"
tokio = { version = "1", features = ["macros"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"

[build-dependencies]
cc = "1.0"

[profile.release]
opt-level = 3
lto = true
codegen-units = 1
```

### Adding Code (Step 2)

1. **Create C Implementation (`complex_math.c`):**
```c
// complex_math.c
#include <math.h>

// Fibonacci calculation in C
unsigned long long fibonacci_c(int n) {
    if (n <= 1) return n;
    
    unsigned long long prev = 0;
    unsigned long long curr = 1;
    
    for (int i = 2; i <= n; i++) {
        unsigned long long next = prev + curr;
        prev = curr;
        curr = next;
    }
    return curr;
}

// Matrix multiplication in C
void matrix_multiply_c(const double* a, const double* b, double* result, int size) {
    for (int i = 0; i < size; i++) {
        for (int j = 0; j < size; j++) {
            double sum = 0.0;
            for (int k = 0; k < size; k++) {
                sum += a[i * size + k] * b[k * size + j];
            }
            result[i * size + j] = sum;
        }
    }
}

// Prime number check
int is_prime_c(int num) {
    if (num <= 1) return 0;
    if (num <= 3) return 1;
    if (num % 2 == 0 || num % 3 == 0) return 0;

    for (int i = 5; i * i <= num; i += 6) {
        if (num % i == 0 || num % (i + 2) == 0) return 0;
    }
    return 1;
}
```

2. **Create Rust Implementation (`src/main.rs`):**
```rust
use lambda_runtime::{service_fn, Error, LambdaEvent};
use serde::{Deserialize, Serialize};
use serde_json::{json, Value};
use std::time::Instant;

// FFI declarations for C functions
extern "C" {
    fn fibonacci_c(n: i32) -> u64;
    fn matrix_multiply_c(a: *const f64, b: *const f64, result: *mut f64, size: i32);
    fn is_prime_c(num: i32) -> i32;
}

#[derive(Deserialize)]
struct Request {
    operation: String,
    parameters: Value,
}

#[derive(Serialize)]
struct Response {
    result: Value,
    execution_time_ms: f64,
    operation: String,
}

async fn main_handler(event: LambdaEvent<Request>) -> Result<Value, Error> {
    let start_time = Instant::now();
    let (request, _context) = event.into_parts();
    
    let result = match request.operation.as_str() {
        "fibonacci" => {
            let n: i32 = request.parameters.get("n")
                .and_then(|v| v.as_i64())
                .map(|v| v as i32)
                .unwrap_or(10);
            
            unsafe {
                let fib = fibonacci_c(n);
                json!({ "fibonacci": fib })
            }
        },
        "matrix_multiply" => {
            let size = 3;
            let matrix_a = vec![1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0];
            let matrix_b = vec![9.0, 8.0, 7.0, 6.0, 5.0, 4.0, 3.0, 2.0, 1.0];
            let mut result = vec![0.0; size * size];
            
            unsafe {
                matrix_multiply_c(
                    matrix_a.as_ptr(),
                    matrix_b.as_ptr(),
                    result.as_mut_ptr(),
                    size as i32
                );
            }
            
            json!({
                "matrix_result": result.chunks(size)
                    .map(|chunk| chunk.to_vec())
                    .collect::<Vec<_>>()
            })
        },
        "prime_check" => {
            let numbers: Vec<i32> = request.parameters.get("numbers")
                .and_then(|v| v.as_array())
                .map(|arr| arr.iter()
                    .filter_map(|v| v.as_i64().map(|n| n as i32))
                    .collect())
                .unwrap_or_else(|| vec![2, 3, 5, 7, 11, 13, 17, 19]);
                
            let prime_results: Vec<(i32, bool)> = numbers.iter()
                .map(|&n| {
                    let is_prime = unsafe { is_prime_c(n) == 1 };
                    (n, is_prime)
                })
                .collect();
                
            json!({ "prime_results": prime_results })
        },
        _ => json!({ "error": "Unsupported operation" })
    };
    
    let response = Response {
        result,
        execution_time_ms: start_time.elapsed().as_secs_f64() * 1000.0,
        operation: request.operation,
    };
    
    Ok(json!(response))
}

#[tokio::main]
async fn main() -> Result<(), Error> {
    let func = service_fn(main_handler);
    lambda_runtime::run(func).await?;
    Ok(())
}
```

3. **Create Build Script (`build.rs`):**
```rust
fn main() {
    cc::Build::new()
        .file("src/complex_math.c")
        .compile("complex_math");
}
```

### Building and Packaging for Lambda (Step 3)

1. **Build the Project:**
```bash
sudo apt-get install musl-tools
rustup target add x86_64-unknown-linux-musl
cd src
# you are in rust_example/src folder

gcc -c complex_math.c -o complex_math.o
cargo build --release --target x86_64-unknown-linux-musl
```

2. **Package Everything:**
```bash
cp target/x86_64-unknown-linux-musl/release/rust_runtime bootstrap

# Make bootstrap executable
chmod +x bootstrap

# Create deployment package
zip lambda.zip bootstrap
```

### Deploying to AWS (Step 4)

1. **Create IAM Role:**
```bash
# Create trust policy
echo '{
  "Version": "2012-10-17",
  "Statement": [{
    "Action": "sts:AssumeRole",
    "Effect": "Allow",
    "Principal": {
      "Service": "lambda.amazonaws.com"
    }
  }]
}' > trust-policy.json

# Create role
aws iam create-role \
    --role-name rust-lambda-role \
    --assume-role-policy-document file://trust-policy.json

# Attach basic execution policy
aws iam attach-role-policy \
    --role-name rust-lambda-role \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
```

2. **Deploy Function:**
```bash
# Upload to S3 (optional)
aws s3 cp lambda.zip s3://your-bucket/

# Create function
aws lambda create-function \
    --function-name rust-c-arithmetic \
    --runtime provided.al2 \
    --role <ROLE-ARN> \
    --handler bootstrap \
    --zip-file fileb://lambda.zip \
    --memory-size 128 \
    --timeout 30
```

### Testing the Function (Step 5)

1. **Test Fibonacci:**
```bash
aws lambda invoke \
    --function-name rust-c-arithmetic \
    --payload '{"operation":"fibonacci","parameters":{"n":10}}' \
    response.json
```

2. **Test Matrix Multiplication:**
```bash
aws lambda invoke \
    --function-name rust-c-arithmetic \
    --payload '{"operation":"matrix_multiply","parameters":{}}' \
    response.json
```

3. **Test Prime Check:**
```bash
aws lambda invoke \
    --function-name rust-c-arithmetic \
    --payload '{"operation":"prime_check","parameters":{"numbers":[2,3,5,7,11]}}' \
    response.json
```

## Troubleshooting Tips

### Build Issues:
- Ensure all dependencies are installed
- Check C compiler is in PATH
- Verify Rust toolchain is up to date: `rustup update`

### Deployment Issues:
- Check IAM role permissions
- Verify ZIP contains all required files
- Check Lambda logs in CloudWatch

### Runtime Issues:
- Monitor memory usage in CloudWatch
- Check timeout settings
- Verify input payload format

## Challenges and Debugging Tips

### 1. Runtime API Nuances
Ensure your bootstrap script handles the runtime API correctly. Debugging logs can help identify issues.

### 2. Cold Starts
Use Rust's optimized binaries and strip unused symbols to reduce startup time.

### 3. Memory Management
Be cautious with C's manual memory allocation to avoid leaks.

## The Dev vs. DevOps Takeaway

In a world where developers dream of simplicity and DevOps engineers strive for efficiency, Lambda custom runtimes are the ultimate middle ground. They let you build cutting-edge solutions without managing infrastructure. And with Rust and C in the mix, you get the perfect combination of safety and speed.

So next time someone says, "Why not just use EC2?" you'll have the perfect response: "Because I like my weekends, and I like my code optimized."

Go ahead — build, deploy, and let the performance speak for itself.

## Resources for Further Learning

- [AWS Lambda Runtime Interface](https://docs.aws.amazon.com/lambda/latest/dg/runtimes-api.html)
- [Rust and AWS Lambda](https://github.com/awslabs/aws-lambda-rust-runtime)

---

**Disclaimer:** No production environments were harmed during the writing of this post, though some dev environments may have been mildly inconvenienced.

I'd love to hear your thoughts! Have you tried custom runtimes? Are you team Rust, team C, or team "I'll stick with my JavaScript, thank you very much"? Feel free to share your ideas or suggestions with me!

## Final Words

So, there you have it! A fully functional, tested, and locally optimized Rust-based Lambda function. Now, excuse me while I go brag about this build to my DevOps team.