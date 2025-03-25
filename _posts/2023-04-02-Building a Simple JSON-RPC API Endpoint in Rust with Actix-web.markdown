---
layout: post
title:  "Building a Simple JSON-RPC API Endpoint in Rust with Actix-web"
date:   2023-04-02 19:12:00 +0200
categories: jekyll update
---


# Building a Simple JSON-RPC API Endpoint in Rust with Actix-web

## Introduction

Imagine you're hosting a dinner party, and you want to ensure that every guest gets exactly what they ordered. You could take each order individually, but that would be time-consuming and chaotic. Instead, you decide to create a menu where guests can write down their orders, and you process them all at once. This is essentially what an API endpoint does—it accepts requests, processes them, and returns the appropriate responses.

In this article, we'll walk through the process of creating a simple JSON-RPC API endpoint in Rust using the `actix-web` framework. We'll cover everything from setting up the project to handling sessions, routing, and data models. By the end of this article, you'll have a solid understanding of how to build a robust API endpoint in Rust.

## Prerequisites

Before we dive in, make sure you have the following installed:

- Rust (install from [rustup.rs](https://rustup.rs/))
- Cargo (comes with Rust)

## Setting Up the Project

First, let's create a new Rust project:

```bash
cargo new rust_json_rpc_api
cd rust_json_rpc_api
```

Next, add the necessary dependencies to your `Cargo.toml` file:

```toml
[dependencies]
actix-web = "4.0"
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

Here, we're using `actix-web` for the web framework, `serde` for serialization and deserialization, and `serde_json` for handling JSON data.

## Understanding the Main Components

### 1. Sessions in Actix-web

In `actix-web`, sessions are used to store user-specific data across multiple requests. Think of a session as a temporary storage locker for each guest at your dinner party. Each locker has a unique key, and only the guest with the key can access their locker.

To use sessions in `actix-web`, you need to configure a session middleware. This middleware will handle the creation and management of sessions for you.

### 2. Adding Data to Sessions

Data stored in sessions must be serializable and deserializable. In Rust, this means the data must implement the `Serialize` and `Deserialize` traits from the `serde` crate.

Additionally, since sessions are shared across multiple threads, the data must be thread-safe. This is typically achieved using `Arc<Mutex<T>>`, where `Arc` (Atomic Reference Counted) allows multiple ownership, and `Mutex` ensures mutual exclusion.

### 3. Accessing Data from Sessions

Once data is added to a session, it can be accessed using the session's `get` method. This method returns an `Option<T>`, where `T` is the type of the data stored in the session.

### 4. Constructs in Actix-web

To create an API endpoint in `actix-web`, you'll need to define the following constructs:

- **Routes**: Define the URL paths and the HTTP methods (GET, POST, etc.) that your API will handle.
- **Handlers**: Functions that process the incoming requests and return responses.
- **Models**: Data structures that represent the data being sent and received by your API.
- **Main**: The entry point of your application where you configure and run the server.

### 5. Step-by-Step Example

Let's walk through the process of creating a simple JSON-RPC API endpoint step by step.

#### Step 1: Define the Models

Create a `models.rs` file in your `src` directory:

```rust
// src/models.rs
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
pub struct JsonRpcRequest {
    pub jsonrpc: String,
    pub method: String,
    pub params: serde_json::Value,
    pub id: Option<i32>,
}

#[derive(Serialize, Deserialize, Debug)]
pub struct JsonRpcResponse {
    pub jsonrpc: String,
    pub result: serde_json::Value,
    pub id: Option<i32>,
}
```

Here, we define two structs: `JsonRpcRequest` and `JsonRpcResponse`. These structs represent the JSON-RPC request and response formats.

#### Step 2: Define the Handlers

Create a `handlers.rs` file in your `src` directory:

```rust
// src/handlers.rs
use actix_web::{web, HttpResponse};
use serde_json::json;
use crate::models::{JsonRpcRequest, JsonRpcResponse};

pub async fn handle_json_rpc(request: web::Json<JsonRpcRequest>) -> HttpResponse {
    let method = &request.method;
    let params = &request.params;

    // Process the request based on the method
    let result = match method.as_str() {
        "echo" => params.clone(),
        _ => json!({"error": "Method not found"}),
    };

    // Create a JSON-RPC response
    let response = JsonRpcResponse {
        jsonrpc: "2.0".to_string(),
        result,
        id: request.id,
    };

    HttpResponse::Ok().json(response)
}
```

In this handler, we process the incoming JSON-RPC request and return a response. If the method is "echo", we simply return the parameters. Otherwise, we return an error.

#### Step 3: Define the Routes

Create a `routes.rs` file in your `src` directory:

```rust
// src/routes.rs
use actix_web::web;
use crate::handlers;

pub fn config(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::resource("/jsonrpc")
            .route(web::post().to(handlers::handle_json_rpc)),
    );
}
```

Here, we define a single route `/jsonrpc` that accepts POST requests and routes them to the `handle_json_rpc` handler.

#### Step 4: Configure the Main Application

Finally, update your `main.rs` file to configure and run the server:

```rust
// src/main.rs
mod models;
mod handlers;
mod routes;

use actix_web::{App, HttpServer};
use routes::config;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .configure(config)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

In this file, we define the entry point of our application. We create an `HttpServer` and configure it with our routes.

### Step 5: Running the Application

To run the application, use the following command:

```bash
cargo run
```

Your API will be available at `http://127.0.0.1:8080/jsonrpc`.

### Step 6: Testing the API

You can test the API using `curl` or any HTTP client like Postman. Here's an example using `curl`:

```bash
curl -X POST http://127.0.0.1:8080/jsonrpc -H "Content-Type: application/json" -d '{
    "jsonrpc": "2.0",
    "method": "echo",
    "params": {"message": "Hello, world!"},
    "id": 1
}'
```

You should receive a response like this:

```json
{
    "jsonrpc": "2.0",
    "result": {"message": "Hello, world!"},
    "id": 1
}
```

## Conclusion

In this article, we've walked through the process of creating a simple JSON-RPC API endpoint in Rust using the `actix-web` framework. We've covered the main components, including sessions, data models, handlers, and routes, and provided a step-by-step example to help you get started.

By understanding these concepts, you'll be well-equipped to build more complex APIs in Rust. Whether you're building a small microservice or a large-scale web application, Rust's performance and safety guarantees make it an excellent choice for API development.

So, the next time you're hosting a dinner party (or building an API), remember to keep things organized, efficient, and, most importantly, enjoyable. Happy coding!