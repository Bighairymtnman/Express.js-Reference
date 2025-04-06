# Express.js Elements Quick Reference

A concise reference guide for Express.js API development patterns.

## Table of Contents

- [Application Setup](#application-setup)
- [Core Routing Patterns](#core-routing-patterns)
- [Router Object](#router-object)
- [Middleware Patterns](#middleware-patterns)
- [Request Object Properties](#request-object-properties)
- [Response Object Methods](#response-object-methods)
- [Error Handling Patterns](#error-handling-patterns)
- [Content Negotiation](#content-negotiation)
- [CORS Configuration](#cors-configuration)
- [Environment Configuration](#environment-configuration)
- [Express Application Settings](#express-application-settings)

## Application Setup

Basic Express application setup:

```javascript
// Import the Express module
const express = require("express");

// Create an Express application
const app = express();

// Define port to run server on
const PORT = process.env.PORT || 3000;

// Middleware for parsing JSON and URL-encoded data
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Start the server
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Core Routing Patterns

### Basic Routes

HTTP methods for RESTful API endpoints:

```javascript
// GET request - Retrieve resources
app.get("/users", (req, res) => {
  // Return a list of users
  res.json({ users: [] });
});

// POST request - Create a resource
app.post("/users", (req, res) => {
  // Create a new user
  res.status(201).json({ message: "User created" });
});

// PUT request - Update a resource
app.put("/users/:id", (req, res) => {
  // Update an existing user
  res.json({ message: `User ${req.params.id} updated` });
});

// DELETE request - Remove a resource
app.delete("/users/:id", (req, res) => {
  // Delete a user
  res.json({ message: `User ${req.params.id} deleted` });
});
```

### Route Parameters

Extracting values from URL paths:

```javascript
// Required parameters
app.get("/users/:id", (req, res) => {
  // The :id parameter is required
  const userId = req.params.id;
  res.json({ userId });
});

// Optional parameters (with ?)
app.get("/products/:category?", (req, res) => {
  // The :category parameter is optional
  const category = req.params.category || "all";
  res.json({ category });
});

// Multiple parameters
app.get("/shops/:shopId/products/:productId", (req, res) => {
  // Extract multiple parameters
  const { shopId, productId } = req.params;
  res.json({ shopId, productId });
});
```

### Query Parameters

Handling query string parameters:

```javascript
app.get("/products", (req, res) => {
  // Extract query parameters with defaults
  const { search, limit = 10, page = 1 } = req.query;

  // Use the parameters for filtering, pagination, etc.
  res.json({
    search,
    limit: parseInt(limit),
    page: parseInt(page),
  });
});
```

## Router Object

Modular routing with Express Router:

```javascript
// users.routes.js
const express = require("express");
const router = express.Router();

// Define routes on the router
router.get("/", (req, res) => {
  res.json({ users: [] });
});

router.get("/:id", (req, res) => {
  res.json({ userId: req.params.id });
});

// Export the router
module.exports = router;

// app.js
// Import the router
const usersRouter = require("./users.routes");

// Mount the router at a specific path
app.use("/users", usersRouter);
// Now routes are accessible at /users and /users/:id
```

## Middleware Patterns

### Built-in Middleware

Express includes several built-in middleware functions:

```javascript
// Parse JSON request bodies
app.use(express.json());

// Parse URL-encoded request bodies (form data)
app.use(express.urlencoded({ extended: true }));

// Serve static files from the 'public' directory
app.use(express.static("public"));
```

### Custom Middleware

Creating and using custom middleware functions:

```javascript
// Logger middleware - logs all requests
app.use((req, res, next) => {
  console.log(`${req.method} ${req.url} at ${new Date().toISOString()}`);
  // Call next() to pass control to the next middleware
  next();
});

// Authentication middleware - protects routes
const authenticate = (req, res, next) => {
  const token = req.headers.authorization;

  if (!token) {
    return res.status(401).json({ message: "Authentication required" });
  }

  // Verify token logic would go here
  // If valid, attach user to request object
  req.user = { id: "123", role: "admin" };

  next();
};

// Apply to a specific route
app.get("/protected", authenticate, (req, res) => {
  res.json({ data: "protected data", user: req.user });
});

// Apply to all routes under a path
app.use("/admin", authenticate);
```

### Error-handling Middleware

Special middleware for handling errors:

```javascript
// Regular middleware that might throw or pass an error
app.use((req, res, next) => {
  // This passes an error to the error handler
  if (!req.user) {
    const error = new Error("User not found");
    error.status = 404;
    next(error);
  }
  next();
});

// Error-handling middleware (must have 4 parameters)
app.use((err, req, res, next) => {
  // Set status code from error or default to 500
  res.status(err.status || 500);

  // Send error response
  res.json({
    error: {
      message: err.message,
      status: err.status,
    },
  });
});
```

## Request Object Properties

Common properties of the request object:

```javascript
app.get("/example", (req, res) => {
  // URL parameters from route definition
  const id = req.params.id;

  // Query string parameters (?key=value)
  const page = req.query.page;

  // Request body (for POST, PUT requests)
  const data = req.body;

  // Request headers (case-insensitive)
  const contentType = req.headers["content-type"];
  const authorization = req.headers.authorization;

  // Request path and method
  const path = req.path;
  const method = req.method;

  // Client's IP address
  const ip = req.ip;

  // Check if request is HTTPS
  const secure = req.secure;

  // Original URL including query string
  const fullUrl = req.originalUrl;

  // Protocol (http or https)
  const protocol = req.protocol;
});
```

## Response Object Methods

Common methods of the response object:

```javascript
app.get("/response-examples", (req, res) => {
  // Send JSON response
  res.json({ message: "Success", data: [] });

  // Send plain text or HTML
  res.send("Plain text response");

  // Set status code and chain methods
  res.status(201).json({ message: "Created" });

  // Set response headers
  res.set("Content-Type", "application/json");
  res.set({
    "Cache-Control": "no-store",
    Pragma: "no-cache",
  });

  // Redirect to another URL
  res.redirect("/new-location");
  res.redirect(301, "/permanent-location"); // With status code

  // Send a file
  res.sendFile("/path/to/file.pdf");

  // Send a file as download attachment
  res.download("/path/to/file.pdf", "report.pdf");

  // Set cookies
  res.cookie("name", "value", {
    maxAge: 900000,
    httpOnly: true,
  });

  // Clear a cookie
  res.clearCookie("name");

  // End response without data
  res.end();
});
```

## Error Handling Patterns

Patterns for handling errors in Express:

```javascript
// Try-catch in async handlers
app.get("/users/:id", async (req, res, next) => {
  try {
    // Code that might throw an error
    const user = await findUser(req.params.id);

    if (!user) {
      return res.status(404).json({ message: "User not found" });
    }

    res.json(user);
  } catch (err) {
    // Pass error to Express error handler
    next(err);
  }
});

// Async handler wrapper function
// Eliminates need for try-catch in every async route
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Using the wrapper
app.get(
  "/products/:id",
  asyncHandler(async (req, res) => {
    const product = await findProduct(req.params.id);

    if (!product) {
      return res.status(404).json({ message: "Product not found" });
    }

    res.json(product);
  })
);
```

## Content Negotiation

Responding with different formats based on client preferences:

```javascript
app.get("/data", (req, res) => {
  // Respond with different formats based on Accept header
  res.format({
    // If client accepts text/plain
    "text/plain": () => {
      res.send("Plain text data");
    },

    // If client accepts text/html
    "text/html": () => {
      res.send("<p>HTML data</p>");
    },

    // If client accepts application/json
    "application/json": () => {
      res.json({ message: "JSON data" });
    },

    // If no acceptable format is found
    default: () => {
      res.status(406).send("Not Acceptable");
    },
  });
});
```

## CORS Configuration

Cross-Origin Resource Sharing configuration:

```javascript
// Basic CORS middleware
app.use((req, res, next) => {
  // Allow requests from any origin
  res.header("Access-Control-Allow-Origin", "*");

  // Allow specific headers
  res.header(
    "Access-Control-Allow-Headers",
    "Origin, X-Requested-With, Content-Type, Accept, Authorization"
  );

  // Handle preflight requests
  if (req.method === "OPTIONS") {
    res.header("Access-Control-Allow-Methods", "GET, POST, PUT, DELETE, PATCH");
    return res.status(200).json({});
  }

  next();
});

// Using the cors package
const cors = require("cors");

// Allow all origins
app.use(cors());

// Custom CORS configuration
app.use(
  cors({
    // Allowed origins
    origin: ["https://example.com", "https://api.example.com"],

    // Allowed HTTP methods
    methods: ["GET", "POST", "PUT", "DELETE"],

    // Allowed request headers
    allowedHeaders: ["Content-Type", "Authorization"],

    // Allow credentials (cookies, authorization headers)
    credentials: true,

    // How long preflight requests can be cached (in seconds)
    maxAge: 86400,
  })
);
```

## Environment Configuration

Managing environment-specific configuration:

```javascript
// Load environment variables from .env file
require("dotenv").config();

// Access environment variables
const PORT = process.env.PORT || 3000;
const NODE_ENV = process.env.NODE_ENV || "development";
const DB_URI = process.env.DB_URI;

// Environment-specific behavior
if (NODE_ENV === "development") {
  // Development-only middleware
  const morgan = require("morgan");
  app.use(morgan("dev"));
}

if (NODE_ENV === "production") {
  // Production-only middleware
  const helmet = require("helmet");
  app.use(helmet());
}
```

## Express Application Settings

Configuring Express application behavior:

```javascript
// View engine setup for server-rendered applications
app.set("view engine", "ejs");
app.set("views", "./views");

// Trust proxy headers (useful behind load balancers)
app.set("trust proxy", true);

// Case sensitivity in routes
// With this enabled, /users and /Users are different routes
app.set("case sensitive routing", true);

// Strict routing (treats /foo and /foo/ as different)
app.set("strict routing", true);

// Disable X-Powered-By header for security
app.disable("x-powered-by");

// Pretty-print JSON responses in development
app.set("json spaces", 2);

// Maximum request body size
app.use(express.json({ limit: "10mb" }));
app.use(express.urlencoded({ limit: "10mb", extended: true }));
```
