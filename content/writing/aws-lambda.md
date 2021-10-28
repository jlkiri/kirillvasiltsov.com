---
title: "aws"
date: 2021-09-28
language: en
---

Ever since AWS released [Rust runtime for AWS lambda](https://aws.amazon.com/blogs/opensource/rust-runtime-for-aws-lambda/) I've been wanting to try it out. In this article I am going to walk you through every step required to write and deploy a lambda written in Rust to AWS. 

To avoid making this article too big I assume you are familiar with basic Rust, Docker and Node. Also make sure you have Rust toolchain, Docker and Node.js installed in your environment.

To avoid building yet another boring hello-world-like handler, we will build a _n a n o s e r v i c e_ that takes an image and returns a glitched version of it (which you can use as a profile picture etc. but that is up to you). Pretty useless in isolation but still fun and just enough for a good walkthrough.

## Start a fresh Rust project

Let's begin with a fresh Rust project.

```
cargo new glitch
cd glitch
```

Let's build the core of our API: a glitch function. Actually, two glitch functions. I must warn you that I'm not a professional glitch artist and that there is a lot of depth to glitch art, but the two simple tricks below will suffice. One trick is to just take a byte of the image you want to glitch and replace it with some random byte. Another trick is to take an arbitrary *sequence* of bytes and sort it. Rust does not come with a random number generator so we need to install it first:

```toml
[dependencies]
rand = "0.8.4"
```

And here's the byte-replacing glitch function. We put it in `src/lib.rs`.

```rust
use rand::{self, Rng};

pub fn glitch_replace(image: &mut [u8]) {
    let mut rng = rand::thread_rng();
    let size = image.len() - 1;
    let rand_idx: usize = rng.gen_range(0..=size);
    image[rand_idx] = rng.gen_range(0..=255);
}
```

Nothing extraordinary here, we just take a reference to our image as a mutable slice of bytes and replace one. Next is the sort glitch:

```rust
const CHUNK_LEN: usize = 19;

pub fn glitch_sort(image: &mut [u8]) {
    let mut rng = rand::thread_rng();
    let size = image.len() - 1;
    let split_idx: usize = rng.gen_range(0..=size - CHUNK_LEN);
    let (_left, right) = image.split_at_mut(split_idx);
    let (glitched, _rest) = right.split_at_mut(CHUNK_LEN);
    glitched.sort();
}
```

Again there is nothing complicated here. Note a very convenient `split_at_mut` method that easily lets us select the chunk we want to sort. `CHUNK_LEN` is a variable in the sense that you can try different values and expected different glitch outcomes. I randomly chose 19.

Finally, for more noticeable effect we apply these two functions multiple times as steps of one big glitch job.

```rust
pub fn glitch(image: &mut [u8]) {
    glitch_replace(image);
    glitch_sort(image);
    glitch_replace(image);
    glitch_sort(image);
    glitch_replace(image);
    glitch_sort(image);
    glitch_sort(image);
}
```

Next we move on to building a lambda.

## Cargo.toml: Download required dependencies

These are the minimal dependencies we'll need.

```toml
[dependencies]
lambda_http = "0.4.1"
lambda_runtime = "0.4.1"
tokio = "1.12.0"
rand = "0.8.4"
jemallocator = "0.3.2"
```

`lambda_runtime` is the runtime for our functions. This is required because Rust is not (yet) in the list of default runtimes at the time of writing. It possible, however to BYOR (bring your own runtime) to AWS and that's what we're doing here. `lambda_http` is a helper library that gives us type definitions for the request and context of the lambda. `tokio` is an async runtime. Our handler is so simple that we don't need `async` but `lambda_runtime` requires it so we have no choice but to play along. Just in case you are unfamiliar with it think of it as a library that runs Rust futures. We won't need to worry much about async apart from defining our functions as `async`. Finally, there is `jemallocator`. We will get to it later.

## main.rs: Handler

Alright, we have a glitch function but how do use it in our request handler? Let us define `apply_glitch` handler that takes the request, extracts image bytes from the body and copies the glitched version into the response.

```rust
use lambda_http::handler;
use lambda_http::Body;
use lambda_http::{IntoResponse, Request};

async fn apply_glitch(mut req: Request, _c: Context) -> Result<impl IntoResponse, Error> {
    let payload = req.body_mut();
    match payload {
        Body::Binary(image) => {
            glitch(image);
            Ok(image.to_owned())
        }
        // Ideally you want to handle Text and Empty cases too.
        // We use a special macro unimplemented!() that prevents the compiler from failing without all cases handled.
        _ => unimplemented!(),
    }
}
```

Note the useful `IntoResponse` trait that allows us to just return things like `String`s and `Vec<u8>`s without thinking much about response headers.

## main.rs: Main

Next, we need to simply wrap our actual handler in a `lambda_http::handler`. This creates an actual lambda that can be run by the lambda runtime we installed. Literally two lines of code to hook everything up.

```rust
use lambda_runtime::{self, Error};
use lambda_http::handler;
use jemallocator;

#[global_allocator]
static ALLOC: jemallocator::Jemalloc = jemallocator::Jemalloc;

#[tokio::main]
async fn main() -> Result<(), Error> {
    let func = handler(apply_glitch);
    lambda_runtime::run(func).await?;
    Ok(())
}
```

Don't forget the `#[tokio::main]` bit. This is an [attribute macro](https://doc.rust-lang.org/reference/procedural-macros.html) from `tokio` that does some magic under the hood to make our `main` function async. The `#[global_allocator]` part is also needed to make the lambda work but we will get to it later.

## Deploying to AWS

There are multiple ways to deploy this to AWS. One of them is using the AWS console. I find the console confusing for many tasks, even simple ones, so I am very excited that there exists another way: CDK. It is a Node.js library that allows us to define the required AWS resources declaratively with real code. It comes with TypeScript type definitions so in a lot of cases we don't even need to look into the documentation.

## CDK project

The only downside of CDK is that it requires a couple things in our local environment: [`aws` CLI](https://aws.amazon.com/cli/) and Node.js. Make sure the CLI is configured with your credentials. Next, install CDK:

```
npm install -g aws-cdk
cdk --version
```

CDK requires that some resources exist prior to any deployments like buckets that your CDK output (which is basically a CloudFormation stack) and other artifacts like Lambda functions are uploaded to. This is done with `bootstrap` command.

```
cdk bootstrap aws://ACCOUNT-NUMBER/REGION
```

Now we are ready to create a new CDK project which is responsible for deploying our lambda to the cloud. Create a `lambda` folder (or choose whatever name you want) in the root of your Rust project and execute the following command in it:

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

Let's check that everything is OK by running `cdk synth` which does a dry run and shows you the CloudFormation code it would generate. Right now, there is not much we can do without installing additional _constructs_ - the basic building blocks of AWS CDK apps, so let's do it first.

```
npm install @aws-cdk/aws-lambda @aws-cdk/aws-apigatewayv2-integrations @aws-cdk/aws-apigatewayv2 @aws-cdk/aws-apigatewayv2
```

Import these in your `lambda/lib/lambda-stack.ts`:

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

`code` is where our binary lies (we will get to it soon). `handler`, normally, is the name of the actual function to call but it seems to be irrelevant when using custom runtimes, so just choose any string you want. Finally, `runtime` is `PROVIDED_AL2` which simply means we bring our own runtime (which we earlier installed as a Rust dependency) that will work on Amazon Linux 2. Just a lambda is not enough, however. Lambdas are not publicly accessible from outside of the cloud by default and we need to use API Gateway to connect the function to the outside world. To do this, add the following to your CDK code:

```ts
const glitchApi = new apigw.HttpApi(this, "GlitchAPI", {
  description: "Image glitching API",
  defaultIntegration: new intg.LambdaProxyIntegration({
    handler: glitchHandler,
  }),
});
```

This code is pretty self-explanatory. It creates an HTTP API Gateway that will trigger our lambda, `glitchHandler`, which we defined above, on incoming requests. Note how CDK makes it easy to refer to other resources: by using actual references within code.

## Building a binary

We're almost ready but we need to make sure that CDK can see and upload our lambda binary. Normally Rust puts the build output inside `target/` folder and gives it the same name as your package name:

```toml
[package]
name = "glitch"
```

One weird thing about AWS Rust lambdas is that the binary needs to be named `bootstrap`. To do this, we need to add some settings to `Cargo.toml`:

```toml
[package]
autobins = false

[[bin]]
name = "bootstrap"
path = "src/main.rs"
```

This takes care of the name. Next, we could also change the output folder to `artifacts` so CDK can see it and `cargo build` the project directly but let's imagine that you want to work on this project in different environments. The `bootstrap` binary actually MUST be built with `x86_64-unknown-linux-gnu` target. This is not possible on e.g. Windows, so let's use Docker!

If you've ever used Docker with Rust you probably know that compiling can be painfully slow. This is because there is no `cargo` option to [build only dependencies](https://github.com/rust-lang/cargo/issues/2644) at the moment of writing.
Luckily, there is a very good project [cargo-chef](https://crates.io/crates/cargo-chef) that provides a workaround. Here's how we use it in our Dockerfile (mostly copy-paste from the project's README):

```Dockerfile
FROM lukemathwalker/cargo-chef:latest-rust-1.53.0 AS chef
WORKDIR /app

FROM chef AS planner
COPY . .
RUN cargo chef prepare --recipe-path recipe.json

FROM chef AS builder 
COPY --from=planner /app/recipe.json recipe.json
# Build dependencies - this is the caching Docker layer!
RUN cargo chef cook --release --recipe-path recipe.json
COPY . .
RUN cargo build --release

FROM scratch AS export
COPY --from=builder /app/target/release/bootstrap /
```

With this, if we run:

```
docker build -o artifacts .
```

Docker will build a `x86_64-unknown-linux-gnu` binary and put it inside `artifacts` folder. Finally, CDK has all it needs to successfully deploy our lambda! So let's do it (you need to be inside the `lambda` folder):

```
cdk deploy
```

Ideally we want to know the URL of our API Gateway immediately and there is a nice way to make CDK output this info by writing a couple more lines:

```ts
new cdk.CfnOutput(this, "glitchApi", {
      value: glitchApi.url!,
    });
```

Now if we add the `--outputs-file` option to the `cdk` command like this:

```
cdk deploy --outputs-file cdk-outputs.json
```

we will see a `lambda/cdk-outputs.json` file that has the URL inside:

```json
{
  "LambdaStack": {
    "glitchApi": "https://your-gateway-api-url.amazonaws.com/"
  }
}
```

## Glitch!

That was a lot of work but now we can finally call our glitch API. Prepare an image file you want to glitch and do this:

```
curl -X POST https://your-gateway-api-url.amazonaws.com --data-binary "@pic.jpg" -o glitched.jpg
```

You should see a `glitched.jpg` file that is glitched and hopefully looks aesthetically pleasing! Now that everything is working, you can play with the settings like the number and order of glitches, the size of the chunk that is sorted etc. If you know other simple ways to achieve nice-looking glitches, feel free to [tell me on Twitter](https://twitter.com/virtualkirill)!

Wait...what about `jemallocator`? Oh yes, I promised to explain this as well. So, it seems that for quite a long time AWS lambdas needed to be built for `x86_64-unknown-linux-musl` target. This was a pain because it needed a `musl` toolchain which is not available by default. However, it looks like now you [CAN](https://umccr.org/blog/aws-bioinformatics-rust/) use `x86_64-unknown-linux-gnu` but with a caveat: you need to use `jemallocator`. This is literally just one install and one more line to your code. The default allocator Rust uses on Unix platforms is `malloc`. I do not know if this limitation will disappear in the future.
