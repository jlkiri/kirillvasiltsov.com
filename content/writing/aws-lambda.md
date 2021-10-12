---
title: "aws"
date: 2021-09-28
language: en
---

Ever since AWS released [Rust runtime for AWS lambda](https://aws.amazon.com/blogs/opensource/rust-runtime-for-aws-lambda/) I've been wanting to try it out. Now is the time and I am going to walk you through every step required to deploy a lambda written in Rust to AWS. I assume you have Rust toolchain, Docker and Node.js installed in your environment.

To avoid building another boring hello-world handler, we will build a _n a n o s e r v i c e_ that does a simple binary search and returns the result. Pretty useless but still fun and just enough for a walkthrough.

## Start a fresh Rust project

Let's begin with a fresh Rust project.

```
cargo new bisearch
cd bisearch
```

Here's the core of our API: a binary search function. There's nothing new here so feel free to just copypaste it and move forward.

```rust
fn binary_search(array: &[i32], num: i32) -> Option<usize> {
    let mut lo = 0;
    let mut hi = array.len() - 1;
    while lo <= hi {
        let mid = lo + (hi - lo) / 2;
        match array[mid] {
            n if n > num => {
                hi = mid;
            }
            n if n < num => {
                lo = mid;
            }
            _ => return Some(mid),
        }
    }
    None
}
```

## Cargo.toml: Download required dependencies

These are the minimal dependencies we'll need.

```toml
[dependencies]
lambda_runtime = "0.4.1"
lambda_http = "0.4.1"
serde = "1.0.130"
serde_json = "1.0.68"
tokio = "1.12.0"
```

`lambda_runtime` is the runtime for our functions. This is required because Rust is not (yet) in the list of default runtimes at the time of writing. It possible, however to BYOR (bring your own runtime) to AWS and that's what we're doing here. `lambda_http` is a helper library that gives us type definitions for the request and context of the lambda. `serde` and `serde_json` are needed to (de)serialize requests and responses. Finally, `tokio` is an async runtime. Our handler is so simple we don't need async but `lambda_runtime` requires it so we have no choice but to play along. If you are unfamiliar with it, think of it as a library that runs Rust futures. We won't need to worry much about async apart from defining our functions as `async`.

## main.rs: Entrypoint

Alright, we have a binary search function but how do we turn it into a request handler? We need to simply wrap our actual handler in a `lambda_http::handler`. This creates a lambda that can be run by the lambda runtime we just installed. Literally two lines of code to hook everything up.

```rust
use lambda_runtime::{self, Error};
use lambda_http::handler;

#[tokio::main]
async fn main() -> Result<(), Error> {
    let func = handler(find_index); // We define this function below.
    lambda_runtime::run(func).await?;
    Ok(())
}
```

## find_index: Handler function

This is the most interesting part. Our function `find_index` is going to look like this:

```rust
use lambda_http::{
    IntoResponse, Request, RequestExt, Response,
};

async fn find_index(req: Request, _c: Context) -> Result<impl IntoResponse, Error> {
    // ...
}
```

`IntoResponse` is a convenient trait implemented for some standard types like `()`, `String` or `&str`. It is also implemented for `Value` which is what you get by constructing a JSON object using `serde_json` library as we will see below. So easy.

Before we write out next line, we need to decide on the shape of input to our API. Since we need at least an array of numbers and the number to look for, it makes sense to expect a JSON object:

```json
{
  "array": [1, 2, 3, 4, 5],
  "num": 3
}
```

Let's go on and define it as a `struct`:

```rust
#[derive(Debug, Deserialize)]
struct SearchRequest {
    array: Vec<i32>,
    num: i32,
}
```

The `Deserialize` part is important. This tells `serde` _how_ to deserialize text from the body to our type. Now that we have this in place, we can deserialize the raw body to `SearchRequest`.

```rust
let payload = req.payload::<SearchRequest>()?; // Deserialize request.
let data = payload.ok_or("No body!")?; // Check if body exists.
```

## CDK project

We are ready to create a CDK project which is responsible for deploying our lambda to the cloud. Create a `lambda` folder (or choose whatever name you want) and execute the following command in it:

```
cdk init app --language=typescript
```

This will generate all the files we'll need. Open `lambda\lib\lambda-stack.ts` which should look like this:

```ts
import * as cdk from "@aws-cdk/core";

export class LambdaStack extends cdk.Stack {
  constructor(scope: cdk.Construct, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // The code that defines your stack goes here
  }
}
```

Let's check that everything is OK by running `cdk synth` which does a dry run and shows you the CloudFormation code it would generate. There is not much we can do without installing _constructs_ - the basic building blocks of AWS CDK apps which, so let's do it first.

```
npm install @aws-cdk/aws-lambda @aws-cdk/aws-apigatewayv2-integrations @aws-cdk/aws-apigatewayv2  @aws-cdk/aws-apigatewayv2
```

```ts
import * as apigw from "@aws-cdk/aws-apigatewayv2";
import * as intg from "@aws-cdk/aws-apigatewayv2-integrations";
import * as lambda from "@aws-cdk/aws-lambda";
import * as cdk from "@aws-cdk/core";
```

Now we can actually _define_ a lambda function inside the `constructor` above:

```ts
const glitchHandler = new lambda.Function(this, "GlitchHandler", {
  code: lambda.Code.fromAsset("../artifacts"),
  handler: "unrelated",
  runtime: lambda.Runtime.PROVIDED_AL2,
});
```

`code` is where our binary lies. `handler`, normally, is the name of the actual function to call but it seems to be irrelevant when using custom runtimes, so just choose any string you want. Finally, `runtime` is `PROVIDED_AL2` which simply means we bring our own runtime (which we earlier installed as a Rust dependency) that will work on Amazon Linux 2. Just a lambda is not enough, however. Lambdas are not publicly accessible from outside of the cloud by default and we need to use API Gateway to connect the function to the outside world. To do this, add the following to your CDK code:

```ts
const glitchApi = new apigw.HttpApi(this, "GlitchAPI", {
  description: "Image glitching API",
  defaultIntegration: new intg.LambdaProxyIntegration({
    handler: glitchHandler,
  }),
});
```

The code is pretty self-explanatory. But there is one thing we forgot. CORS. Right, now is the best time to think about CORS so we don't have to discover that CORS is the reason our lambda doesn't work later.
