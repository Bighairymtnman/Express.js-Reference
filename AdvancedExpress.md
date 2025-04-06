# Advanced Express.js

A concise reference guide for advanced Express.js concepts and patterns.

## Table of Contents

- [Authentication & Authorization](#authentication--authorization)
- [Template Engines](#template-engines)
- [Database Integration](#database-integration)
- [File Uploads](#file-uploads)
- [Validation](#validation)
- [Security Best Practices](#security-best-practices)
- [Performance Optimization](#performance-optimization)
- [Testing](#testing)
- [Deployment](#deployment)
- [Advanced Patterns](#advanced-patterns)
- [WebSockets](#websockets)
- [GraphQL with Express](#graphql-with-express)
- [Microservices](#microservices)

## Authentication & Authorization

### JWT Authentication

```javascript
const jwt = require("jsonwebtoken");
const SECRET_KEY = process.env.JWT_SECRET || "your-secret-key";

// Generate token
app.post("/login", (req, res) => {
  // Validate credentials
  const token = jwt.sign({ userId: 1, role: "admin" }, SECRET_KEY, {
    expiresIn: "1h",
  });
  res.json({ token });
});

// Verify token middleware
const authenticateJWT = (req, res, next) => {
  const token = req.headers.authorization?.split(" ")[1];

  if (!token) return res.status(401).json({ message: "Unauthorized" });

  jwt.verify(token, SECRET_KEY, (err, user) => {
    if (err) return res.status(403).json({ message: "Invalid token" });
    req.user = user;
    next();
  });
};

// Role-based authorization
const authorizeRole = (role) => (req, res, next) => {
  if (req.user.role !== role)
    return res.status(403).json({ message: "Forbidden" });
  next();
};

// Protected routes
app.get("/profile", authenticateJWT, (req, res) =>
  res.json({ user: req.user })
);
app.get("/admin", authenticateJWT, authorizeRole("admin"), (req, res) =>
  res.json({ message: "Admin area" })
);
```

### OAuth Integration

```javascript
const passport = require("passport");
const GoogleStrategy = require("passport-google-oauth20").Strategy;

passport.use(
  new GoogleStrategy(
    {
      clientID: process.env.GOOGLE_CLIENT_ID,
      clientSecret: process.env.GOOGLE_CLIENT_SECRET,
      callbackURL: "/auth/google/callback",
    },
    (accessToken, refreshToken, profile, done) => {
      // Find or create user
      return done(null, profile);
    }
  )
);

app.use(passport.initialize());

app.get(
  "/auth/google",
  passport.authenticate("google", { scope: ["profile", "email"] })
);

app.get(
  "/auth/google/callback",
  passport.authenticate("google", { failureRedirect: "/login" }),
  (req, res) => res.redirect("/dashboard")
);
```

## Template Engines

### EJS

```javascript
app.set("view engine", "ejs");
app.set("views", "./views");

app.get("/", (req, res) => {
  res.render("index", {
    title: "Express with EJS",
    items: ["Item 1", "Item 2", "Item 3"],
  });
});

// views/index.ejs
// <h1><%= title %></h1>
// <ul>
//   <% items.forEach(item => { %>
//     <li><%= item %></li>
//   <% }); %>
// </ul>
```

### Pug

```javascript
app.set("view engine", "pug");
app.set("views", "./views");

app.get("/", (req, res) => {
  res.render("index", {
    title: "Express with Pug",
    items: ["Item 1", "Item 2", "Item 3"],
  });
});

// views/index.pug
// h1= title
// ul
//   each item in items
//     li= item
```

## Database Integration

### MongoDB with Mongoose

```javascript
const mongoose = require("mongoose");

mongoose
  .connect("mongodb://localhost:27017/myapp")
  .then(() => console.log("Connected to MongoDB"))
  .catch((err) => console.error("MongoDB connection error:", err));

// Define schema and model
const userSchema = new mongoose.Schema({
  name: { type: String, required: true },
  email: { type: String, required: true, unique: true },
});

const User = mongoose.model("User", userSchema);

// CRUD operations
app.post("/users", async (req, res) => {
  try {
    const user = new User(req.body);
    await user.save();
    res.status(201).json(user);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

app.get("/users", async (req, res) => {
  try {
    const users = await User.find();
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});
```

### SQL with Sequelize

```javascript
const { Sequelize, DataTypes } = require("sequelize");

const sequelize = new Sequelize("database", "username", "password", {
  host: "localhost",
  dialect: "postgres",
});

// Define model
const User = sequelize.define("User", {
  name: { type: DataTypes.STRING, allowNull: false },
  email: { type: DataTypes.STRING, allowNull: false, unique: true },
});

// Sync model with database
sequelize.sync();

// CRUD operations
app.post("/users", async (req, res) => {
  try {
    const user = await User.create(req.body);
    res.status(201).json(user);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});

app.get("/users", async (req, res) => {
  try {
    const users = await User.findAll();
    res.json(users);
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});
```

## File Uploads

### Using Multer

```javascript
const multer = require("multer");

// Configure storage
const storage = multer.diskStorage({
  destination: (req, file, cb) => cb(null, "uploads/"),
  filename: (req, file, cb) => cb(null, `${Date.now()}-${file.originalname}`),
});

// File filter
const fileFilter = (req, file, cb) => {
  if (!file.originalname.match(/\.(jpg|jpeg|png|gif)$/)) {
    return cb(new Error("Only image files are allowed!"), false);
  }
  cb(null, true);
};

// Configure upload
const upload = multer({
  storage,
  limits: { fileSize: 1024 * 1024 * 5 }, // 5MB
  fileFilter,
});

// Single file upload
app.post("/upload", upload.single("image"), (req, res) => {
  if (!req.file) return res.status(400).json({ message: "No file uploaded" });

  res.json({
    message: "File uploaded successfully",
    file: req.file.filename,
  });
});

// Multiple files upload
app.post("/upload-multiple", upload.array("images", 5), (req, res) => {
  if (!req.files?.length)
    return res.status(400).json({ message: "No files uploaded" });

  res.json({
    message: "Files uploaded successfully",
    files: req.files.map((file) => file.filename),
  });
});
```

## Validation

### Using Express-Validator

```javascript
const { body, param, validationResult } = require("express-validator");

// Validation middleware
const validate = (validations) => {
  return async (req, res, next) => {
    await Promise.all(validations.map((validation) => validation.run(req)));
    const errors = validationResult(req);
    if (errors.isEmpty()) return next();
    res.status(400).json({ errors: errors.array() });
  };
};

// User creation validation
app.post(
  "/users",
  validate([
    body("name").trim().notEmpty().withMessage("Name is required"),
    body("email").isEmail().withMessage("Valid email required"),
    body("password")
      .isLength({ min: 6 })
      .withMessage("Password must be at least 6 characters"),
  ]),
  (req, res) => {
    // Validation passed
    res.status(201).json({ message: "User created" });
  }
);

// URL parameter validation
app.get(
  "/users/:id",
  validate([param("id").isInt().withMessage("User ID must be an integer")]),
  (req, res) => {
    res.json({ userId: req.params.id });
  }
);
```

## Security Best Practices

### Helmet for HTTP Headers

```javascript
const helmet = require("helmet");

// Basic usage
app.use(helmet());

// Custom configuration
app.use(
  helmet({
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'", "trusted-cdn.com"],
      },
    },
  })
);
```

### CORS Configuration

```javascript
const cors = require("cors");

// Basic CORS
app.use(cors());

// Configured CORS
app.use(
  cors({
    origin: ["https://example.com", "https://www.example.com"],
    methods: ["GET", "POST", "PUT", "DELETE"],
    allowedHeaders: ["Content-Type", "Authorization"],
    credentials: true,
  })
);

// Route-specific CORS
app.get("/api/public", cors(), (req, res) => {
  res.json({ message: "Public API" });
});
```

### Rate Limiting

```javascript
const rateLimit = require("express-rate-limit");

// API rate limiter
const apiLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // limit each IP to 100 requests per windowMs
  message: "Too many requests, please try again later.",
});

// Apply to all requests
app.use(apiLimiter);

// Apply to specific routes
app.use("/api/", apiLimiter);
```

## Performance Optimization

### Compression

```javascript
const compression = require("compression");

// Apply compression
app.use(compression());
```

### HTTP Caching

```javascript
// Static assets caching
app.use(
  "/static",
  express.static("public", {
    maxAge: "1d", // Cache for 1 day
  })
);

// API response caching
app.get("/api/data", (req, res) => {
  res.set("Cache-Control", "public, max-age=300"); // Cache for 5 minutes
  res.json({ data: "Some data" });
});
```

## Testing

### Unit Testing with Jest

```javascript
// users.js
const express = require("express");
const router = express.Router();

router.get("/", (req, res) => {
  res.json([{ id: 1, name: "John" }]);
});

module.exports = router;

// users.test.js
const request = require("supertest");
const express = require("express");
const usersRouter = require("./users");

const app = express();
app.use("/users", usersRouter);

describe("Users API", () => {
  test("GET /users should return users", async () => {
    const response = await request(app).get("/users");
    expect(response.status).toBe(200);
    expect(response.body).toHaveLength(1);
    expect(response.body[0]).toHaveProperty("name", "John");
  });
});
```

### Integration Testing

```javascript
const mongoose = require("mongoose");
const { MongoMemoryServer } = require("mongodb-memory-server");
const request = require("supertest");
const app = require("./app");

let mongoServer;

beforeAll(async () => {
  mongoServer = await MongoMemoryServer.create();
  await mongoose.connect(mongoServer.getUri());
});

afterAll(async () => {
  await mongoose.disconnect();
  await mongoServer.stop();
});

describe("API Integration", () => {
  test("should create and retrieve a user", async () => {
    // Create user
    const createResponse = await request(app)
      .post("/users")
      .send({ name: "Test User", email: "test@example.com" });

    expect(createResponse.status).toBe(201);

    // Get users
    const getResponse = await request(app).get("/users");
    expect(getResponse.status).toBe(200);
    expect(getResponse.body).toHaveLength(1);
    expect(getResponse.body[0].name).toBe("Test User");
  });
});
```

## Deployment

### Environment Configuration

```javascript
// config.js
require("dotenv").config();

module.exports = {
  port: process.env.PORT || 3000,
  nodeEnv: process.env.NODE_ENV || "development",
  mongoUri: process.env.MONGO_URI,
  jwtSecret: process.env.JWT_SECRET,
};
```

### Process Management with PM2

```bash
# Install PM2
npm install -g pm2

# Start application
pm2 start app.js --name "api"

# Cluster mode
pm2 start app.js -i max

# Ecosystem file
```

```javascript
// ecosystem.config.js
module.exports = {
  apps: [
    {
      name: "api",
      script: "./app.js",
      instances: "max",
      env_production: {
        NODE_ENV: "production",
        PORT: 8080,
      },
    },
  ],
};
```

## Advanced Patterns

### Middleware Composition

```javascript
// Authentication middleware
const authenticate = (req, res, next) => {
  if (!req.user) return res.status(401).json({ message: "Unauthorized" });
  next();
};

// Role check middleware
const checkRole = (role) => (req, res, next) => {
  if (req.user.role !== role)
    return res.status(403).json({ message: "Forbidden" });
  next();
};

// Combine middleware
const adminOnly = [authenticate, checkRole("admin")];

// Apply to route
app.get("/admin", adminOnly, (req, res) => {
  res.json({ message: "Admin area" });
});
```

### Error Handling

```javascript
// Custom error class
class AppError extends Error {
  constructor(message, statusCode) {
    super(message);
    this.statusCode = statusCode;
  }
}

// Async handler
const asyncHandler = (fn) => (req, res, next) => {
  Promise.resolve(fn(req, res, next)).catch(next);
};

// Route with async handler
app.get(
  "/users/:id",
  asyncHandler(async (req, res) => {
    const user = await findUser(req.params.id);
    if (!user) throw new AppError("User not found", 404);
    res.json(user);
  })
);

// Global error handler
app.use((err, req, res, next) => {
  const statusCode = err.statusCode || 500;
  res.status(statusCode).json({
    status: "error",
    message: err.message,
  });
});
```

## WebSockets

### Using Socket.io

```javascript
const express = require("express");
const http = require("http");
const socketIo = require("socket.io");

const app = express();
const server = http.createServer(app);
const io = socketIo(server);

// Socket.io connection handling
io.on("connection", (socket) => {
  console.log("New client connected", socket.id);

  // Join a room
  socket.on("join-room", (room) => {
    socket.join(room);
    socket.to(room).emit("user-joined", { socketId: socket.id });
  });

  // Handle chat messages
  socket.on("chat-message", (data) => {
    const { room, message, user } = data;
    io.to(room).emit("chat-message", {
      message,
      user,
      timestamp: new Date(),
    });
  });

  // Handle disconnection
  socket.on("disconnect", () => {
    console.log("Client disconnected", socket.id);
  });
});

server.listen(3000, () => {
  console.log("Server running on port 3000");
});
```

### Authentication with WebSockets

```javascript
const jwt = require("jsonwebtoken");
const SECRET_KEY = process.env.JWT_SECRET || "your-secret-key";

// Middleware to authenticate socket connections
io.use((socket, next) => {
  const token = socket.handshake.auth.token;

  if (!token) {
    return next(new Error("Authentication error"));
  }

  jwt.verify(token, SECRET_KEY, (err, decoded) => {
    if (err) return next(new Error("Invalid token"));

    // Attach user data to socket
    socket.user = decoded;
    next();
  });
});

io.on("connection", (socket) => {
  console.log(`User connected: ${socket.user.username}`);

  // Join user's private room
  socket.join(`user:${socket.user.id}`);

  // Send private message
  socket.on("private-message", ({ userId, message }) => {
    io.to(`user:${userId}`).emit("private-message", {
      from: socket.user.id,
      message,
    });
  });
});
```

### Real-time Notifications

```javascript
// Server-side
function notifyUser(userId, notification) {
  io.to(`user:${userId}`).emit("notification", notification);
}

// Example: Notify when a new comment is added
app.post("/posts/:postId/comments", authenticateJWT, async (req, res) => {
  try {
    const { postId } = req.params;
    const { content } = req.body;
    const userId = req.user.id;

    // Create comment
    const comment = await Comment.create({
      content,
      postId,
      userId,
    });

    // Get post author
    const post = await Post.findById(postId);

    // Notify post author about new comment
    if (post.userId !== userId) {
      notifyUser(post.userId, {
        type: "new_comment",
        message: `New comment on your post`,
        data: {
          postId,
          commentId: comment.id,
        },
      });
    }

    res.status(201).json(comment);
  } catch (error) {
    res.status(400).json({ message: error.message });
  }
});
```

## GraphQL with Express

### Basic Setup

```javascript
const express = require("express");
const { graphqlHTTP } = require("express-graphql");
const { buildSchema } = require("graphql");

// Define schema
const schema = buildSchema(`
  type User {
    id: ID!
    name: String!
    email: String!
  }
  
  type Query {
    user(id: ID!): User
    users: [User!]!
  }
  
  type Mutation {
    createUser(name: String!, email: String!): User!
  }
`);

// Sample data
const users = [{ id: "1", name: "John", email: "john@example.com" }];

// Resolvers
const root = {
  user: ({ id }) => users.find((user) => user.id === id),
  users: () => users,
  createUser: ({ name, email }) => {
    const id = String(users.length + 1);
    const user = { id, name, email };
    users.push(user);
    return user;
  },
};

// Set up GraphQL endpoint
const app = express();

app.use(
  "/graphql",
  graphqlHTTP({
    schema: schema,
    rootValue: root,
    graphiql: true, // Enable GraphiQL interface
  })
);

app.listen(4000, () => {
  console.log("GraphQL server running at http://localhost:4000/graphql");
});
```

### Integration with Database

```javascript
const express = require("express");
const { graphqlHTTP } = require("express-graphql");
const { buildSchema } = require("graphql");
const mongoose = require("mongoose");

// Connect to MongoDB
mongoose.connect("mongodb://localhost:27017/graphql-demo");

// Define User model
const User = mongoose.model("User", {
  name: String,
  email: String,
});

// Define Post model
const Post = mongoose.model("Post", {
  title: String,
  content: String,
  authorId: mongoose.Schema.Types.ObjectId,
});

// Define GraphQL schema
const schema = buildSchema(`
  type User {
    id: ID!
    name: String!
    email: String!
    posts: [Post!]!
  }
  
  type Post {
    id: ID!
    title: String!
    content: String!
    author: User!
  }
  
  type Query {
    user(id: ID!): User
    users: [User!]!
    post(id: ID!): Post
    posts: [Post!]!
  }
  
  type Mutation {
    createUser(name: String!, email: String!): User!
    createPost(title: String!, content: String!, authorId: ID!): Post!
  }
`);

// Resolvers
const root = {
  user: async ({ id }) => {
    const user = await User.findById(id);
    if (!user) return null;

    return {
      id: user._id,
      name: user.name,
      email: user.email,
      posts: async () => {
        const posts = await Post.find({ authorId: user._id });
        return posts.map((p) => ({
          id: p._id,
          title: p.title,
          content: p.content,
          author: () => root.user({ id: p.authorId }),
        }));
      },
    };
  },

  users: async () => {
    const users = await User.find();
    return users.map((u) => ({
      id: u._id,
      name: u.name,
      email: u.email,
      posts: async () => {
        const posts = await Post.find({ authorId: u._id });
        return posts.map((p) => ({
          id: p._id,
          title: p.title,
          content: p.content,
          author: () => root.user({ id: p.authorId }),
        }));
      },
    }));
  },

  post: async ({ id }) => {
    const post = await Post.findById(id);
    if (!post) return null;

    return {
      id: post._id,
      title: post.title,
      content: post.content,
      author: () => root.user({ id: post.authorId }),
    };
  },

  posts: async () => {
    const posts = await Post.find();
    return posts.map((p) => ({
      id: p._id,
      title: p.title,
      content: p.content,
      author: () => root.user({ id: p.authorId }),
    }));
  },

  createUser: async ({ name, email }) => {
    const user = new User({ name, email });
    await user.save();
    return {
      id: user._id,
      name: user.name,
      email: user.email,
      posts: () => [],
    };
  },

  createPost: async ({ title, content, authorId }) => {
    const post = new Post({ title, content, authorId });
    await post.save();
    return {
      id: post._id,
      title: post.title,
      content: post.content,
      author: () => root.user({ id: post.authorId }),
    };
  },
};

// Set up GraphQL endpoint
const app = express();

app.use(
  "/graphql",
  graphqlHTTP({
    schema: schema,
    rootValue: root,
    graphiql: true,
  })
);

app.listen(4000, () => {
  console.log("GraphQL server running at http://localhost:4000/graphql");
});
```

## Microservices

### Service Communication

```javascript
// user-service.js
const express = require("express");
const app = express();

app.use(express.json());

// User data store
const users = [{ id: 1, name: "John", email: "john@example.com" }];

// Get all users
app.get("/users", (req, res) => {
  res.json(users);
});

// Get user by ID
app.get("/users/:id", (req, res) => {
  const user = users.find((u) => u.id === parseInt(req.params.id));
  if (!user) return res.status(404).json({ message: "User not found" });
  res.json(user);
});

app.listen(3001, () => {
  console.log("User service running on port 3001");
});

// order-service.js
const express = require("express");
const axios = require("axios");
const app = express();

app.use(express.json());

// Order data store
const orders = [{ id: 1, userId: 1, items: ["item1", "item2"], total: 50 }];

// Get all orders
app.get("/orders", (req, res) => {
  res.json(orders);
});

// Get order with user details
app.get("/orders/:id/details", async (req, res) => {
  try {
    const order = orders.find((o) => o.id === parseInt(req.params.id));
    if (!order) return res.status(404).json({ message: "Order not found" });

    // Get user details from user service
    const userResponse = await axios.get(
      `http://localhost:3001/users/${order.userId}`
    );
    const user = userResponse.data;

    res.json({
      ...order,
      user,
    });
  } catch (error) {
    res.status(500).json({ message: "Error fetching order details" });
  }
});

app.listen(3002, () => {
  console.log("Order service running on port 3002");
});
```

### API Gateway

```javascript
const express = require("express");
const { createProxyMiddleware } = require("http-proxy-middleware");

const app = express();

// Authentication middleware
const authenticate = (req, res, next) => {
  const token = req.headers.authorization;
  if (!token) {
    return res.status(401).json({ message: "Unauthorized" });
  }
  // Validate token (simplified)
  next();
};

// Route to user service
app.use(
  "/api/users",
  authenticate,
  createProxyMiddleware({
    target: "http://localhost:3001",
    pathRewrite: {
      "^/api/users": "/users",
    },
    changeOrigin: true,
  })
);

// Route to order service
app.use(
  "/api/orders",
  authenticate,
  createProxyMiddleware({
    target: "http://localhost:3002",
    pathRewrite: {
      "^/api/orders": "/orders",
    },
    changeOrigin: true,
  })
);

// Health check endpoint
app.get("/health", (req, res) => {
  res.json({ status: "UP" });
});

app.listen(3000, () => {
  console.log("API Gateway running on port 3000");
});
```

### Service Discovery

```javascript
// npm install eureka-js-client

// user-service.js
const express = require("express");
const { Eureka } = require("eureka-js-client");

const app = express();

// Eureka client setup
const client = new Eureka({
  instance: {
    app: "user-service",
    hostName: "localhost",
    ipAddr: "127.0.0.1",
    port: {
      $: 3001,
      "@enabled": true,
    },
    vipAddress: "user-service",
    dataCenterInfo: {
      "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
      name: "MyOwn",
    },
  },
  eureka: {
    host: "localhost",
    port: 8761,
    servicePath: "/eureka/apps/",
  },
});

// Register with Eureka
client.start();

// API routes
app.get("/users", (req, res) => {
  res.json([{ id: 1, name: "John" }]);
});

app.listen(3001, () => {
  console.log("User service running on port 3001");
});

// order-service.js with service discovery
const express = require("express");
const axios = require("axios");
const { Eureka } = require("eureka-js-client");

const app = express();

// Eureka client setup
const client = new Eureka({
  instance: {
    app: "order-service",
    hostName: "localhost",
    ipAddr: "127.0.0.1",
    port: {
      $: 3002,
      "@enabled": true,
    },
    vipAddress: "order-service",
    dataCenterInfo: {
      "@class": "com.netflix.appinfo.InstanceInfo$DefaultDataCenterInfo",
      name: "MyOwn",
    },
  },
  eureka: {
    host: "localhost",
    port: 8761,
    servicePath: "/eureka/apps/",
  },
});

// Register with Eureka
client.start();

// Helper to get service URL
function getServiceUrl(serviceName) {
  const instances = client.getInstancesByAppId(serviceName);
  if (instances.length === 0) {
    throw new Error(`Service ${serviceName} not found`);
  }
  const instance = instances[0];
  return `http://${instance.hostName}:${instance.port.$}`;
}

// API routes
app.get("/orders/:id/details", async (req, res) => {
  try {
    // Get user service URL from Eureka
    const userServiceUrl = getServiceUrl("USER-SERVICE");

    // Call user service
    const userResponse = await axios.get(`${userServiceUrl}/users/1`);
    const user = userResponse.data;

    res.json({
      id: req.params.id,
      items: ["item1", "item2"],
      user,
    });
  } catch (error) {
    res.status(500).json({ message: error.message });
  }
});

app.listen(3002, () => {
  console.log("Order service running on port 3002");
});
```
