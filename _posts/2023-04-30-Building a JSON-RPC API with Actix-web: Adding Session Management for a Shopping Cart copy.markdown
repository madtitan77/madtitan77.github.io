---
layout: post
title:  "Building a JSON-RPC API with Actix-web: Adding Session Management for a Shopping Cart"
date:   2023-04-30 01:14:00 +0200
categories: jekyll update
---
# Building a JSON-RPC API with Actix-web: Adding Session Management for a Shopping Cart

Welcome back, fellow Rustaceans! In our previous article, we explored how to create a simple JSON-RPC endpoint using Actix-web. Today, we’re taking the next step: making our application useful by adding session management. Specifically, we’ll build a shopping cart application where users can add items to their cart, and the cart data will persist across requests using Actix-web’s session management capabilities.

By the end of this article, you’ll understand:
1. How sessions work in Actix-web.
2. How to add and access data in sessions.
3. The constructs needed to build a session-based API in Actix-web.
4. A step-by-step example of creating a session-based shopping cart endpoint.

Let’s dive in!

---

## Table of Contents
1. **Understanding Sessions in Actix-web**
2. **Adding Data to Sessions**
   - Constraints and Traits
   - Thread Safety: Mutex, Arc, and Async
3. **Accessing Session Data**
4. **Actix-web Constructs for Session Management**
   - Routes
   - Handlers
   - Middleware
5. **Step-by-Step Example: Building a Shopping Cart**
   - Setting Up `main.rs`
   - Defining Routes in `routes.rs`
   - Creating Models in `models.rs`
   - Implementing Handlers in `handlers.rs`
6. **Common Issues and Troubleshooting**
7. **Conclusion**

---

## 1. Understanding Sessions in Actix-web

Sessions are a way to store user-specific data across multiple HTTP requests. In Actix-web, sessions are managed using middleware, specifically the `actix-session` crate. This middleware allows you to store data in a server-side session and associate it with a unique session ID, which is typically stored in a cookie on the client side.

### Key Concepts:
- **Session Middleware**: This is responsible for creating and managing sessions. It uses a session store (e.g., in-memory, Redis) to persist session data.
- **Session Object**: This is a key-value store where you can add, retrieve, and remove data.
- **Session ID**: A unique identifier for each session, usually stored in a cookie.

Actix-web’s session management is flexible and supports various backends for storing session data, such as in-memory storage, Redis, or database-backed storage.

---

## 2. Adding Data to Sessions

### Constraints and Traits
To store data in a session, the data must implement the `Serialize` and `Deserialize` traits from the `serde` crate. This is because session data is serialized before being stored and deserialized when retrieved.

For example, if you want to store a shopping cart in the session, the cart data structure must derive `Serialize` and `Deserialize`:

```rust
#[derive(Serialize, Deserialize, Clone)]
struct ShoppingCart {
    items: Vec<CartItem>,
}

#[derive(Serialize, Deserialize, Clone)]
struct CartItem {
    product_id: u32,
    quantity: u32,
}
```

### Thread Safety: Mutex, Arc, and Async
Actix-web is an asynchronous framework, so you need to ensure that session data is accessed safely across threads. If you’re storing mutable data in the session, you’ll need to wrap it in a `Mutex` or `RwLock` to prevent data races.

For example, if you want to store a mutable shopping cart:

```rust
use std::sync::{Arc, Mutex};

#[derive(Serialize, Deserialize, Clone)]
struct ShoppingCart {
    items: Vec<CartItem>,
}

type SharedCart = Arc<Mutex<ShoppingCart>>;
```

However, Actix-web’s session management already handles concurrency for you, so you don’t need to worry about `Mutex` or `Arc` for most use cases. The session object itself is thread-safe.

---

## 3. Accessing Session Data

Once data is added to the session, you can access it in subsequent requests using the `Session` object. The `Session` object provides methods like `get`, `insert`, and `remove` to manipulate session data.

For example, to retrieve the shopping cart from the session:

```rust
async fn get_cart(session: Session) -> Option<ShoppingCart> {
    session.get::<ShoppingCart>("cart").unwrap()
}
```

To add or update data in the session:

```rust
async fn add_to_cart(session: Session, item: CartItem) {
    let mut cart: ShoppingCart = session.get("cart").unwrap().unwrap_or_default();
    cart.items.push(item);
    session.insert("cart", cart).unwrap();
}
```

---

## 4. Actix-web Constructs for Session Management

To build a session-based API, you’ll need to use the following constructs:

### Routes
Routes define the endpoints of your API. In Actix-web, routes are configured using the `App` or `Scope` objects.

Example:
```rust
use actix_web::{web, App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    HttpServer::new(|| {
        App::new()
            .route("/cart", web::get().to(get_cart))
            .route("/cart/add", web::post().to(add_to_cart))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### Handlers
Handlers are functions that process incoming requests and return responses. They can access the session object to read or write data.

Example:
```rust
use actix_web::{web, HttpResponse};
use actix_session::Session;

async fn get_cart(session: Session) -> HttpResponse {
    if let Some(cart) = session.get::<ShoppingCart>("cart").unwrap() {
        HttpResponse::Ok().json(cart)
    } else {
        HttpResponse::NotFound().body("Cart not found")
    }
}
```

### Middleware
Middleware is used to add functionality to your application, such as session management. The `actix-session` crate provides the `SessionMiddleware` for this purpose.

Example:
```rust
use actix_session::{SessionMiddleware, storage::RedisSessionStore};
use actix_web::{web, App, HttpServer};

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let store = RedisSessionStore::new("redis://127.0.0.1:6379").await.unwrap();

    HttpServer::new(move || {
        App::new()
            .wrap(SessionMiddleware::new(store.clone(), &[0; 32]))
            .route("/cart", web::get().to(get_cart))
            .route("/cart/add", web::post().to(add_to_cart))
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

---

## 5. Step-by-Step Example: Building a Shopping Cart

Let’s build a shopping cart application step by step.

### Step 1: Setting Up `main.rs`
First, set up the main application with session middleware.

```rust
use actix_session::{SessionMiddleware, storage::RedisSessionStore};
use actix_web::{web, App, HttpServer};

mod routes;
mod handlers;
mod models;

#[actix_web::main]
async fn main() -> std::io::Result<()> {
    let store = RedisSessionStore::new("redis://127.0.0.1:6379").await.unwrap();

    HttpServer::new(move || {
        App::new()
            .wrap(SessionMiddleware::new(store.clone(), &[0; 32]))
            .configure(routes::init_routes)
    })
    .bind("127.0.0.1:8080")?
    .run()
    .await
}
```

### Step 2: Defining Routes in `routes.rs`
Define the routes for the shopping cart API.

```rust
use actix_web::web;
use crate::handlers::{get_cart, add_to_cart};

pub fn init_routes(cfg: &mut web::ServiceConfig) {
    cfg.service(
        web::scope("/api")
            .route("/cart", web::get().to(get_cart))
            .route("/cart/add", web::post().to(add_to_cart)),
    );
}
```

### Step 3: Creating Models in `models.rs`
Define the data structures for the shopping cart.

```rust
use serde::{Serialize, Deserialize};

#[derive(Serialize, Deserialize, Clone)]
pub struct ShoppingCart {
    pub items: Vec<CartItem>,
}

#[derive(Serialize, Deserialize, Clone)]
pub struct CartItem {
    pub product_id: u32,
    pub quantity: u32,
}
```

### Step 4: Implementing Handlers in `handlers.rs`
Implement the handlers for retrieving and updating the shopping cart.

```rust
use actix_web::{web, HttpResponse};
use actix_session::Session;
use crate::models::{ShoppingCart, CartItem};

pub async fn get_cart(session: Session) -> HttpResponse {
    if let Some(cart) = session.get::<ShoppingCart>("cart").unwrap() {
        HttpResponse::Ok().json(cart)
    } else {
        HttpResponse::NotFound().body("Cart not found")
    }
}

pub async fn add_to_cart(session: Session, item: web::Json<CartItem>) -> HttpResponse {
    let mut cart: ShoppingCart = session.get("cart").unwrap().unwrap_or_default();
    cart.items.push(item.into_inner());
    session.insert("cart", cart).unwrap();
    HttpResponse::Ok().body("Item added to cart")
}
```

---

## 6. Common Issues and Troubleshooting

### Issue 1: Session Data Not Persisting
Ensure that the session middleware is correctly configured and that the session store (e.g., Redis) is running.

### Issue 2: Serialization Errors
Make sure all data stored in the session implements `Serialize` and `Deserialize`.

### Issue 3: Concurrent Access
If you’re modifying session data in multiple handlers, ensure that the data is accessed safely. Use `Mutex` or `RwLock` if necessary.

---

## 7. Conclusion

Congratulations! You’ve built a session-based shopping cart API using Actix-web. You’ve learned how sessions work, how to add and access session data, and how to structure your application for scalability and maintainability.

In the next article, we’ll explore how to secure our API using authentication and authorization. Until then, happy coding!

---

Feel free to ask questions or share your thoughts in the comments. If you found this article helpful, don’t forget to share it with your fellow developers. Rust on! 🚀