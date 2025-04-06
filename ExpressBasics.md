# Express.js Basics

A concise reference guide to Express.js fundamentals.

## Table of Contents

- [Introduction](#introduction)
- [Installation & Setup](#installation--setup)
- [Core Concepts](#core-concepts)
  - [Middleware](#middleware)
  - [Routing](#routing)
  - [Router](#router)
  - [Request & Response](#request--response)
- [Environment Configuration](#environment-configuration)
- [Error Handling](#error-handling)
- [Simple API Example](#simple-api-example)

## Introduction

Express.js is a minimal and flexible Node.js web application framework that provides a robust set of features for web and mobile applications.

### Project Structure

A well-organized Express.js project structure:

```
project-root/
├── node_modules/
├── public/            # Static assets
├── routes/            # Route handlers
├── controllers/       # Business logic
├── models/            # Data models
├── middleware/        # Custom middleware
├── config/            # Configuration files
├── utils/             # Utility functions
├── app.js             # Main application file
├── package.json
└── .env               # Environment variables
```

## Installation & Setup

```bash
# Create a new project
mkdir my-express-app && cd my-express-app
npm init -y

# Install Express
npm install express
```

Basic app setup:

```javascript
const express = require("express");
const app = express();
const PORT = process.env.PORT || 3000;

// Parse JSON bodies
app.use(express.json());

// Basic route
app.get("/", (req, res) => {
  res.send("Hello World!");
});

// Start server
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Core Concepts

### Middleware

Functions that have access to the request, response objects, and the next middleware function:

```javascript
// Custom middleware
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url} at ${new Date()}`);
  next(); // Pass control to next middleware
});

// Route-specific middleware
const authenticate = (req, res, next) => {
  if (req.headers.authorization) {
    next();
  } else {
    res.status(401).send("Unauthorized");
  }
};

app.get("/protected", authenticate, (req, res) => {
  res.send("Protected route");
});
```

Common built-in middleware:

```javascript
// Parse JSON bodies
app.use(express.json());

// Parse URL-encoded bodies
app.use(express.urlencoded({ extended: true }));

// Serve static files
app.use(express.static("public"));
```

### Routing

Handling different HTTP methods and URL patterns:

```javascript
// Basic routes
app.get("/users", (req, res) => {
  res.json([{ name: "John" }, { name: "Jane" }]);
});

app.post("/users", (req, res) => {
  res.status(201).json({ message: "User created" });
});

app.put("/users/:id", (req, res) => {
  res.json({ message: `Updated user ${req.params.id}` });
});

app.delete("/users/:id", (req, res) => {
  res.json({ message: `Deleted user ${req.params.id}` });
});
```

Route parameters:

```javascript
app.get("/users/:id", (req, res) => {
  res.json({ userId: req.params.id });
});

app.get("/products/:category/:id", (req, res) => {
  const { category, id } = req.params;
  res.json({ category, productId: id });
});
```

Query parameters:

```javascript
// URL: /search?q=express&limit=10
app.get("/search", (req, res) => {
  const { q, limit } = req.query;
  res.json({ searchTerm: q, limit });
});
```

### Router

Express Router for modular route handling:

```javascript
// routes/users.js
const express = require("express");
const router = express.Router();

router.get("/", (req, res) => {
  res.json([{ name: "John" }, { name: "Jane" }]);
});

router.get("/:id", (req, res) => {
  res.json({ userId: req.params.id });
});

module.exports = router;

// app.js
const usersRouter = require("./routes/users");
app.use("/users", usersRouter);
```

### Request & Response

Common request properties:

```javascript
app.get("/example", (req, res) => {
  const method = req.method; // GET, POST, etc.
  const path = req.path; // /example
  const params = req.params; // Route parameters
  const query = req.query; // Query parameters
  const body = req.body; // Request body
  const headers = req.headers; // Request headers
});
```

Common response methods:

```javascript
app.get("/response-examples", (req, res) => {
  // Send JSON
  res.json({ message: "Success" });

  // Send text/HTML
  res.send("Hello World");

  // Set status code
  res.status(201).json({ message: "Created" });

  // Redirect
  res.redirect("/new-location");

  // Send file
  res.sendFile("/path/to/file.pdf");
});
```

## Environment Configuration

Managing environment-specific settings:

```javascript
// npm install dotenv
require("dotenv").config();

const app = express();
const PORT = process.env.PORT || 3000;
const NODE_ENV = process.env.NODE_ENV || "development";

// Environment-specific middleware
if (NODE_ENV === "development") {
  // Only use logger in development
  const morgan = require("morgan");
  app.use(morgan("dev"));
}

if (NODE_ENV === "production") {
  // Add security headers in production
  const helmet = require("helmet");
  app.use(helmet());
}

// Access environment variables
const dbConnection = process.env.DATABASE_URL;
const apiKey = process.env.API_KEY;

// Example .env file
// PORT=3000
// NODE_ENV=development
// DATABASE_URL=mongodb://localhost:27017/myapp
// API_KEY=your-secret-key
```

## Error Handling

Basic error handling:

```javascript
// Route with error handling
app.get("/users/:id", (req, res) => {
  try {
    // Code that might throw an error
    if (req.params.id === "999") {
      throw new Error("User not found");
    }
    res.json({ id: req.params.id, name: "John" });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Global error handler
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: "Something went wrong!" });
});
```

## Simple API Example

Complete example of a simple REST API:

```javascript
const express = require("express");
const app = express();
const PORT = 3000;

// Middleware
app.use(express.json());

// In-memory data store
const users = [
  { id: 1, name: "John Doe", email: "john@example.com" },
  { id: 2, name: "Jane Smith", email: "jane@example.com" },
];

// Routes
app.get("/api/users", (req, res) => {
  res.json(users);
});

app.get("/api/users/:id", (req, res) => {
  const user = users.find((u) => u.id === parseInt(req.params.id));

  if (!user) {
    return res.status(404).json({ message: "User not found" });
  }

  res.json(user);
});

app.post("/api/users", (req, res) => {
  const { name, email } = req.body;

  if (!name || !email) {
    return res.status(400).json({ message: "Name and email are required" });
  }

  const newUser = {
    id: users.length + 1,
    name,
    email,
  };

  users.push(newUser);
  res.status(201).json(newUser);
});

app.put("/api/users/:id", (req, res) => {
  const user = users.find((u) => u.id === parseInt(req.params.id));

  if (!user) {
    return res.status(404).json({ message: "User not found" });
  }

  const { name, email } = req.body;

  if (name) user.name = name;
  if (email) user.email = email;

  res.json(user);
});

app.delete("/api/users/:id", (req, res) => {
  const index = users.findIndex((u) => u.id === parseInt(req.params.id));

  if (index === -1) {
    return res.status(404).json({ message: "User not found" });
  }

  const deletedUser = users.splice(index, 1)[0];
  res.json(deletedUser);
});

// Start server
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```
