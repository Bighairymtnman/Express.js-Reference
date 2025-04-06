# Collect Tracker Express.js Example

This document analyzes a real-world Express.js application from my CollectTracker project, explaining how Express concepts are implemented in practice. CollectTracker is a collection tracking application for desktop developed by Bighairymtnman.

**Project Repository:** [Bighairymtnman/CollectTracker](https://github.com/Bighairymtnman/CollectTracker)

## Table of Contents

- [Server Setup and Configuration](#server-setup-and-configuration)
- [Route Handling](#route-handling)
- [Database Integration](#database-integration)
- [Middleware Usage](#middleware-usage)

## Server Setup and Configuration

The main server file (`server/index.js`) demonstrates how to set up an Express application with necessary middleware and route configurations:

```javascript
const express = require("express");
const cors = require("cors");
const dotenv = require("dotenv");
const db = require("./db.config");
const collectionsRouter = require("./routes/collections");
const categoriesRouter = require("./routes/categories");
const multer = require("multer");
const storage = multer.memoryStorage();
const upload = multer({ storage: storage });

// Load environment variables
dotenv.config();

// Initialize express
const app = express();

// Middleware
app.use(cors());
app.use(express.json({ limit: "50mb" }));
app.use(express.urlencoded({ limit: "50mb", extended: true }));

// Routes
app.use("/api/collections", collectionsRouter);
app.use("/api", categoriesRouter);

// Basic test route
app.get("/test", (req, res) => {
  res.json({ message: "Server is running!" });
});

// Database test route
app.get("/db-test", async (req, res) => {
  try {
    const result = await db.get("SELECT 1");
    res.json({ message: "Database connected successfully!", result });
  } catch (error) {
    res
      .status(500)
      .json({ message: "Database connection failed", error: error.message });
  }
});

// Set port and start server
const PORT = process.env.PORT || 5000;
app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

### Key Express.js Concepts Demonstrated:

1. **Application Initialization**:

   - `const app = express();` creates a new Express application instance.

2. **Middleware Configuration**:

   - `app.use(cors())` enables Cross-Origin Resource Sharing for all routes.
   - `app.use(express.json({limit: '50mb'}))` parses incoming JSON requests with an increased payload size limit.
   - `app.use(express.urlencoded({limit: '50mb', extended: true}))` parses URL-encoded data with extended syntax support.

3. **Route Mounting**:

   - `app.use('/api/collections', collectionsRouter)` mounts the collections router at the '/api/collections' path.
   - `app.use('/api', categoriesRouter)` mounts the categories router at the '/api' path.
   - This demonstrates Express's modular routing capabilities, keeping route handlers organized in separate files.

4. **Simple Route Handlers**:

   - The `/test` route provides a basic health check endpoint.
   - The `/db-test` route demonstrates an async route handler with error handling for database connectivity testing.

5. **Server Initialization**:

   - `app.listen(PORT, callback)` starts the Express server on the specified port.
   - Uses environment variables with a fallback value for configuration.

6. **File Upload Preparation**:
   - Configures `multer` with memory storage for handling file uploads, though not directly used in this file.

This server setup follows Express.js best practices by:

- Separating concerns (routing, middleware, configuration)
- Using environment variables for configuration
- Implementing basic error handling
- Creating test endpoints for verification
- Setting up appropriate middleware for API functionality

## Route Handling

The `server/routes/categories.js` file demonstrates Express.js route handling for category management in the CollectTracker application:

```javascript
const express = require("express");
const router = express.Router();
const db = require("../db.config");

// Update a category
router.put(
  "/collections/:collectionId/categories/:categoryId",
  async (req, res) => {
    try {
      const result = await db.run(
        "UPDATE categories SET name = ? WHERE id = ? AND collection_id = ?",
        [req.body.name, req.params.categoryId, req.params.collectionId]
      );

      if (result.changes === 0) {
        return res.status(404).json({ error: "Category not found" });
      }

      const updatedCategory = await db.get(
        "SELECT * FROM categories WHERE id = ?",
        [req.params.categoryId]
      );

      res.json(updatedCategory);
    } catch (error) {
      console.error("Error updating category:", error);
      res.status(500).json({ error: error.message });
    }
  }
);

// Get all categories for a collection
router.get("/collections/:id/categories", async (req, res) => {
  try {
    const categories = await db.all(
      "SELECT * FROM categories WHERE collection_id = ?",
      [req.params.id]
    );
    res.json(categories);
  } catch (error) {
    console.error("Error fetching categories:", error);
    res.status(500).json({ error: error.message });
  }
});

// Create a new category
router.post("/collections/:id/categories", async (req, res) => {
  try {
    const result = await db.run(
      "INSERT INTO categories (collection_id, name) VALUES (?, ?)",
      [req.params.id, req.body.name]
    );
    const newCategory = await db.get("SELECT * FROM categories WHERE id = ?", [
      result.id,
    ]);
    res.json(newCategory);
  } catch (error) {
    console.error("Error creating category:", error);
    res.status(500).json({ error: error.message });
  }
});

// Additional routes omitted for brevity
// Full code available at: https://github.com/Bighairymtnman/CollectTracker/blob/main/server/routes/categories.js
```

### Key Express.js Concepts Demonstrated:

1. **Express Router**:

   - `const router = express.Router()` creates a modular, mountable route handler.
   - This router is imported and mounted in the main app file at the '/api' path.

2. **RESTful API Design**:

   - The file implements a RESTful API for category management with appropriate HTTP methods:
     - `GET` for retrieving categories
     - `POST` for creating categories
     - `PUT` for updating categories
     - `DELETE` for removing categories

3. **Route Parameters**:

   - Uses route parameters like `:collectionId` and `:categoryId` to capture dynamic values from URLs.
   - These parameters are accessed via `req.params` object.

4. **Request Body Handling**:

   - Accesses request body data via `req.body` (parsed by the `express.json()` middleware configured in the main app).

5. **Async/Await Pattern**:

   - All route handlers use async/await for clean, readable asynchronous database operations.
   - This avoids callback hell and makes error handling more straightforward.

6. **Error Handling**:

   - Each route handler has try/catch blocks to handle errors.
   - Appropriate HTTP status codes are returned for different error conditions:
     - 404 for resources not found
     - 500 for server errors
   - Error details are logged to the console and returned to the client.

7. **Response Formatting**:

   - Consistent JSON response format across all endpoints.
   - Success responses return the relevant data.
   - Error responses include an error message.

8. **Resource Relationships**:
   - Handles relationships between collections, items, and categories.
   - Implements routes for managing many-to-many relationships (items to categories).

This route file demonstrates how Express.js enables the creation of clean, organized API endpoints with proper error handling and RESTful design principles.

## Advanced Route Handling

The `server/routes/collections.js` file demonstrates more complex Express.js route handling, including file uploads and relationship management:

```javascript
const express = require("express");
const router = express.Router();
const db = require("../db.config");
const multer = require("multer");
const storage = multer.memoryStorage();
const upload = multer({ storage: storage });

// Create a new collection
router.post("/", async (req, res) => {
  try {
    const { name, type, description } = req.body;
    const result = await db.run(
      "INSERT INTO collections (name, type, description) VALUES (?, ?, ?)",
      [name, type, description]
    );
    res.status(201).json({ id: result.id, name, type, description });
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Get all collections with item count
router.get("/", async (req, res) => {
  try {
    const collections = await db.all(`
            SELECT
                c.*,
                COUNT(i.id) as item_count
            FROM collections c
            LEFT JOIN items i ON c.id = i.collection_id
            GROUP BY c.id
        `);
    res.json(collections);
  } catch (error) {
    res.status(500).json({ error: error.message });
  }
});

// Create new item in collection with image support
router.post(
  "/:collectionId/items",
  upload.single("coverImage"),
  async (req, res) => {
    try {
      const { collectionId } = req.params;
      const { title, type, description, data } = req.body;

      console.log("Incoming item data:", { title, type, description, data });
      const result = await db.run(
        "INSERT INTO items (collection_id, title, type, description, data) VALUES (?, ?, ?, ?, ?)",
        [collectionId, title, type, description, JSON.stringify(data)]
      );
      if (req.file) {
        await db.run(
          "INSERT INTO images (item_id, image_data, is_cover) VALUES (?, ?, 1)",
          [result.id, req.file.buffer]
        );
      }

      const newItem = await db.get("SELECT * FROM items WHERE id = ?", [
        result.id,
      ]);

      res.status(201).json(newItem);
    } catch (error) {
      console.error("Error creating item:", error);
      res.status(500).json({ error: error.message });
    }
  }
);

// Get item image
router.get("/items/:itemId/image", async (req, res) => {
  try {
    const image = await db.get(
      "SELECT image_data FROM images WHERE item_id = ? AND is_cover = 1",
      [req.params.itemId]
    );

    if (image) {
      res.setHeader("Content-Type", "image/jpeg");
      res.send(image.image_data);
    } else {
      res.status(404).json({ error: "Image not found" });
    }
  } catch (error) {
    res.status(500).json({ error: "Failed to fetch image" });
  }
});

// Additional routes omitted for brevity
// Full code available at: https://github.com/Bighairymtnman/CollectTracker/blob/main/server/routes/collections.js
```

### Key Express.js Concepts Demonstrated:

1. **File Upload Handling**:

   - Uses `multer` middleware for handling file uploads with memory storage.
   - `upload.single('coverImage')` processes a single file from the 'coverImage' field.
   - Stores binary image data directly in the database.

2. **Complex Response Formatting**:

   - The GET collections endpoint includes a JOIN to count related items.
   - The GET items endpoint formats complex nested data with categories.

3. **Binary Data Handling**:

   - Serves binary image data with appropriate Content-Type headers.
   - Stores and retrieves binary data from the database.

4. **Middleware Chaining**:

   - Routes like `POST /:collectionId/items` chain multiple middleware functions:
     - First the route parameter parsing
     - Then the file upload middleware
     - Finally the route handler

5. **Relationship Management**:

   - Handles parent-child relationships (collections to items).
   - Manages many-to-many relationships through junction tables.
   - Cascading deletes when removing parent resources.

6. **Transaction-like Operations**:

   - Several routes perform multiple database operations that should succeed or fail together.
   - For example, when creating an item with an image, both the item and image records must be created.

7. **Content Negotiation**:

   - Routes return different content types based on the endpoint:
     - JSON for API data
     - Binary image data for image endpoints

8. **Complex Query Parameters**:

   - Uses route parameters to identify resources in hierarchical relationships.
   - For example, `/:collectionId/items/:itemId` identifies a specific item within a collection.

9. **Data Transformation**:
   - Transforms data between storage and API formats:
     - Parses and stringifies JSON data
     - Formats nested relationship data

This route file demonstrates how Express.js can handle complex API requirements including file uploads, relationship management, and different response types. The modular router approach keeps the code organized despite the complexity of the operations.

## Database Integration

The `server/utils/initDb.js` file demonstrates how database initialization is handled in the Express.js application:

```javascript
const db = require("../db.config");

async function initializeDatabase() {
  try {
    // Collections table
    await db.run(`CREATE TABLE IF NOT EXISTS collections (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            name TEXT NOT NULL,
            type TEXT NOT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            description TEXT
        )`);

    // Categories table
    await db.run(`CREATE TABLE IF NOT EXISTS categories (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            collection_id INTEGER NOT NULL,
            name TEXT NOT NULL,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (collection_id) REFERENCES collections (id)
        )`);

    // Items table
    await db.run(`CREATE TABLE IF NOT EXISTS items (
            id INTEGER PRIMARY KEY AUTOINCREMENT,
            collection_id INTEGER,
            title TEXT NOT NULL,
            type TEXT NOT NULL,
            data TEXT,
            description TEXT,
            created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
            FOREIGN KEY (collection_id) REFERENCES collections (id)
        )`);

    // Additional tables omitted for brevity
    // Full code available at: https://github.com/Bighairymtnman/CollectTracker

    console.log("Database initialized successfully");
  } catch (error) {
    console.error("Error initializing database:", error);
  }
}

initializeDatabase();
```

### Key Express.js and Database Integration Concepts:

1. **Database Initialization**:

   - The file contains a self-executing function that initializes the database schema.
   - This approach ensures the database is properly set up before the Express application starts handling requests.

2. **Schema Definition**:

   - Creates tables with appropriate relationships using SQL DDL statements.
   - Uses `CREATE TABLE IF NOT EXISTS` to make the initialization idempotent (safe to run multiple times).

3. **Relationship Modeling**:

   - Defines foreign key constraints to maintain data integrity.
   - Models one-to-many relationships (collections to items) and many-to-many relationships (items to categories).

4. **Error Handling**:

   - Wraps database operations in try/catch blocks to handle initialization errors gracefully.
   - Logs errors to the console for debugging.

5. **Database Abstraction**:

   - Uses a database configuration module (`db.config.js`) that abstracts the underlying database operations.
   - This abstraction allows the application to potentially switch database engines with minimal changes.

6. **Asynchronous Operations**:

   - Uses async/await for clean handling of asynchronous database operations.
   - Sequential table creation ensures dependencies are satisfied in the correct order.

7. **Integration with Express**:
   - While not directly part of the Express request handling, this initialization is crucial for the Express routes to function correctly.
   - The database must be initialized before the Express server starts accepting requests.

This file demonstrates a clean approach to database initialization in an Express.js application. By separating the database setup from the route handling code, it follows the principle of separation of concerns, making the codebase more maintainable.

## Complete Application Architecture

Analyzing the CollectTracker Express.js application reveals a well-structured architecture following modern best practices:

### 1. Layered Architecture

The application follows a clear separation of concerns:

- **Routes Layer**: Handles HTTP requests and responses (`routes/collections.js`, `routes/categories.js`)
- **Database Layer**: Manages data persistence (`db.config.js`, `utils/initDb.js`)
- **Application Layer**: Configures and initializes the Express application (`server/index.js`)

### 2. Modular Design

- **Router Modules**: Each resource type has its own router module
- **Database Initialization**: Separated into its own utility module
- **Configuration**: Environment variables for flexible deployment

### 3. RESTful API Design

- **Resource-Based URLs**: `/collections`, `/collections/:id/items`, etc.
- **Appropriate HTTP Methods**: GET, POST, PUT, DELETE for CRUD operations
- **Proper Status Codes**: 201 for creation, 404 for not found, 500 for server errors

### 4. Error Handling

- **Consistent Error Responses**: JSON-formatted error messages
- **Try/Catch Blocks**: Around async operations
- **Appropriate Status Codes**: For different error conditions

### 5. Middleware Usage

- **Body Parsing**: For JSON and form data
- **CORS**: For cross-origin requests
- **File Upload**: Using multer for handling multipart/form-data

### 6. Database Operations

- **Parameterized Queries**: To prevent SQL injection
- **Transaction-like Operations**: For maintaining data integrity
- **Relationship Management**: Handling one-to-many and many-to-many relationships

### Express.js Best Practices Demonstrated

1. **Modular Routing**: Using express.Router() for cleaner code organization
2. **Middleware Chaining**: Applying multiple middleware to routes as needed
3. **Async/Await**: For clean handling of asynchronous operations
4. **Error Handling**: Consistent error handling patterns
5. **Environment Configuration**: Using dotenv for environment variables
6. **Separation of Concerns**: Clear separation between routes, database, and application setup

This architecture makes the application maintainable, scalable, and follows industry best practices for Express.js development.
