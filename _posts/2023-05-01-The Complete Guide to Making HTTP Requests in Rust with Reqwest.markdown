---
layout: post
title:  "The Complete Guide to Making HTTP Requests in Rust with Reqwest"
date:   2023-05-01 20:13:33 +0200
categories: jekyll update
---
# **The Complete Guide to Making HTTP Requests in Rust with Reqwest**

## **Introduction: Talking to APIs with Rust**

If you've ever used `curl` to fetch data from an API, you already know the basics of HTTP requests. But when you're building applications in Rust, you need a more structured and reliable way to interact with web services. That's where **`reqwest`** comes in—a powerful, ergonomic HTTP client for Rust that makes API interactions smooth and efficient.

In this guide, we'll cover:
1. **Sending simple HTTP requests** (GET, POST, etc.)  
2. **Parsing JSON responses** (using `serde_json`)  
3. **Constructing and deconstructing requests** (headers, query parameters, body)  
4. **Debugging requests** (logging raw HTTP traffic like `curl -v`)  
5. **Alternatives to `reqwest`** (`hyper`, `ureq`, `isahc`)  

By the end, you'll be able to confidently interact with any RESTful or JSON-RPC API in Rust.

---

## **1. Setting Up Reqwest**

First, add `reqwest` and `tokio` (for async runtime) to your `Cargo.toml`:

```toml
[dependencies]
reqwest = { version = "0.11", features = ["json"] }
tokio = { version = "1.0", features = ["full"] }
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
```

We enable the `json` feature to simplify JSON handling.

---

## **2. Making a Simple GET Request**

The simplest way to fetch data from an API is a **GET** request.

```rust
use reqwest;

#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let response = reqwest::get("https://jsonplaceholder.typicode.com/posts/1").await?;
    
    if response.status().is_success() {
        let body = response.text().await?;
        println!("Response: {}", body);
    } else {
        println!("Request failed with status: {}", response.status());
    }
    
    Ok(())
}
```

### **Breaking It Down:**
- `reqwest::get(url)` sends a GET request.
- `.await` pauses execution until the request completes.
- `response.text()` reads the response body as a string.

---

## **3. Parsing JSON Responses**

Most APIs return JSON. We can parse it into Rust structs using `serde_json`.

### **Define a Struct for the Response**
```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct Post {
    userId: u32,
    id: u32,
    title: String,
    body: String,
}
```

### **Deserialize the JSON**
```rust
let post: Post = response.json().await?;
println!("Post Title: {}", post.title);
```

Now, instead of raw JSON, we have a structured `Post` object.

---

## **4. Sending POST Requests with JSON Body**

APIs often require sending data (e.g., creating a resource).

```rust
#[derive(Serialize)]
struct NewPost {
    title: String,
    body: String,
    userId: u32,
}

let new_post = NewPost {
    title: "Hello, Rust!".into(),
    body: "Reqwest is awesome.".into(),
    userId: 1,
};

let client = reqwest::Client::new();
let response = client
    .post("https://jsonplaceholder.typicode.com/posts")
    .json(&new_post)
    .send()
    .await?;

println!("Created Post: {:?}", response.json::<Post>().await?);
```

### **Key Components:**
- `Client::new()` reuses connections (better performance).  
- `.json(&new_post)` serializes the struct into JSON.  
- `.send()` fires the request.  

---

## **5. Adding Headers and Query Parameters**

### **Headers (e.g., API Key)**
```rust
let response = client
    .get("https://api.example.com/data")
    .header("Authorization", "Bearer YOUR_API_KEY")
    .send()
    .await?;
```

### **Query Parameters**
```rust
let params = [("page", "2"), ("limit", "10")];
let response = client
    .get("https://api.example.com/items")
    .query(&params)
    .send()
    .await?;
```

---

## **6. Debugging: Logging Requests Like `curl -v`**

To debug, we need to see the **raw HTTP traffic** (headers, body, etc.).

### **Enable Reqwest Debugging**
```toml
[dependencies]
reqwest = { version = "0.11", features = ["json", "debug"] }
env_logger = "0.9"
```

Then, in `main()`:
```rust
use env_logger;

#[tokio::main]
async fn main() {
    env_logger::init(); // Enable logging
    
    let response = reqwest::get("https://jsonplaceholder.typicode.com/posts/1")
        .await
        .unwrap();
    
    println!("Response: {:?}", response);
}
```

Run with:
```sh
RUST_LOG=reqwest=debug cargo run
```

This will log **all HTTP traffic**, similar to `curl -v`.

---

## **7. Alternatives to Reqwest**

While `reqwest` is the most popular, other HTTP clients exist:

| Library | Pros | Cons |
|---------|------|------|
| **`hyper`** | Extremely fast, low-level | Complex API |
| **`ureq`** | Simple, blocking (no async) | Not for high concurrency |
| **`isahc`** | cURL-like, easy to use | Less popular |

Example with `ureq` (synchronous):
```rust
let response = ureq::get("https://jsonplaceholder.typicode.com/posts/1")
    .call()?;
println!("{}", response.into_string()?);
```

---

## **Conclusion**

We've covered:

✅ **Basic GET/POST requests**  
✅ **JSON serialization/deserialization**  
✅ **Headers and query parameters**  
✅ **Debugging with logging**  
✅ **Alternative HTTP clients**  

With `reqwest`, Rust becomes a powerful tool for interacting with APIs—whether you're building a CLI tool, a web scraper, or a microservice.  

Now go forth and **fetch all the things!** 🚀  

---

### **Further Reading**
- [Reqwest Documentation](https://docs.rs/reqwest/latest/reqwest/)  
- [Serde (JSON Handling)](https://serde.rs/)  
- [Hyper (Low-Level HTTP)](https://hyper.rs/)  

Happy coding! 🦀