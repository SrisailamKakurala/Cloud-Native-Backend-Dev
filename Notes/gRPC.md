# gRPC Complete Guide: From Zero to Hero (JavaScript/Node.js)

## Table of Contents
1. [Introduction to gRPC](#introduction)
2. [Core Concepts](#core-concepts)
3. [Protocol Buffers](#protocol-buffers)
4. [Setting Up Development Environment](#setup)
5. [Your First gRPC Service](#first-service)
6. [Service Types](#service-types)
7. [Advanced Features](#advanced-features)
8. [Best Practices](#best-practices)
9. [Real-World Examples](#real-world-examples)
10. [Troubleshooting](#troubleshooting)

---

## 1. Introduction to gRPC {#introduction}

### What is gRPC?
gRPC (Google Remote Procedure Call) is a high-performance, open-source framework for building distributed systems. It allows applications to communicate with each other as if they were calling local functions.

### Why gRPC?
- **Performance**: 7-10x faster than REST
- **Type Safety**: Strongly typed contracts
- **Code Generation**: Auto-generates client/server code
- **Streaming**: Supports real-time bidirectional streaming
- **Language Agnostic**: Works with 10+ programming languages
- **HTTP/2**: Built on HTTP/2 for multiplexing and compression

### gRPC vs REST Comparison

| Feature | gRPC | REST |
|---------|------|------|
| Protocol | HTTP/2 | HTTP/1.1 |
| Data Format | Protocol Buffers (binary) | JSON (text) |
| Performance | High | Moderate |
| Browser Support | Limited (needs grpc-web) | Full |
| Streaming | Yes | No |
| Code Generation | Yes | No |

---

## 2. Core Concepts {#core-concepts}

### Key Components

1. **Service Definition**: Contract defined in `.proto` files
2. **Server**: Implements the service logic
3. **Client**: Calls the service methods
4. **Stub**: Generated client code for making calls

### Communication Flow
```
Client -> Stub -> Network -> Server -> Service Implementation
```

### Data Serialization
gRPC uses Protocol Buffers (protobuf) for serializing structured data. It's:
- Language-neutral
- Platform-neutral
- Extensible
- Smaller and faster than JSON/XML

---

## 3. Protocol Buffers {#protocol-buffers}

### What are Protocol Buffers?
Protocol Buffers are Google's language-neutral, platform-neutral, extensible mechanism for serializing structured data.

### Basic Syntax

```protobuf
// user.proto
syntax = "proto3";

package user;

// Define a message (like a class/struct)
message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  repeated string hobbies = 4;  // Array of strings
  UserType type = 5;
  int64 created_at = 6;
}

// Define an enum
enum UserType {
  REGULAR = 0;
  PREMIUM = 1;
  ADMIN = 2;
}

// Define a service
service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
  rpc ListUsers(ListUsersRequest) returns (stream User);
}

message GetUserRequest {
  int32 user_id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
  UserType type = 3;
}

message UpdateUserRequest {
  int32 user_id = 1;
  string name = 2;
  string email = 3;
  UserType type = 4;
}

message DeleteUserRequest {
  int32 user_id = 1;
}

message DeleteUserResponse {
  bool success = 1;
  string message = 2;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}
```

### Data Types

| Proto Type | JavaScript | Description |
|------------|------------|-------------|
| double | number | 64-bit float |
| float | number | 32-bit float |
| int32 | number | 32-bit signed integer |
| int64 | string/number | 64-bit signed integer |
| bool | boolean | Boolean value |
| string | string | UTF-8 string |
| bytes | Uint8Array | Byte array |
| repeated | Array | Array of values |

---

## 4. Setting Up Development Environment {#setup}

### Prerequisites
- Node.js 14+ 
- npm or yarn package manager

### Installation

```bash
# Create new project
mkdir my-grpc-project
cd my-grpc-project
npm init -y

# Install gRPC and Protocol Buffers
npm install @grpc/grpc-js @grpc/proto-loader

# Install development dependencies
npm install --save-dev @types/node nodemon

# Install protobuf compiler (optional, for code generation)
# On macOS: brew install protobuf
# On Ubuntu: sudo apt install protobuf-compiler
# On Windows: Download from https://github.com/protocolbuffers/protobuf/releases
```

### Project Structure
```
my-grpc-project/
├── protos/
│   └── user.proto
├── src/
│   ├── server.js
│   ├── client.js
│   └── services/
│       └── userService.js
├── package.json
└── README.md
```

### Package.json Scripts
```json
{
  "name": "my-grpc-project",
  "version": "1.0.0",
  "scripts": {
    "start": "node src/server.js",
    "dev": "nodemon src/server.js",
    "client": "node src/client.js"
  },
  "dependencies": {
    "@grpc/grpc-js": "^1.9.0",
    "@grpc/proto-loader": "^0.7.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.1"
  }
}
```

---

## 5. Your First gRPC Service {#first-service}

### Step 1: Define the Service (protos/user.proto)

```protobuf
syntax = "proto3";

package user;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
  int64 updated_at = 5;
}

message GetUserRequest {
  int32 user_id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

message UpdateUserRequest {
  int32 user_id = 1;
  string name = 2;
  string email = 3;
}

message DeleteUserRequest {
  int32 user_id = 1;
}

message DeleteUserResponse {
  bool success = 1;
  string message = 2;
}
```

### Step 2: Create User Service Implementation (src/services/userService.js)

```javascript
const grpc = require('@grpc/grpc-js');

class UserService {
  constructor() {
    // In-memory database (use real database in production)
    this.users = new Map();
    this.nextId = 1;
  }

  // Unary RPC: Get a single user
  getUser(call, callback) {
    const userId = call.request.user_id;

    if (!this.users.has(userId)) {
      const error = new Error(`User with ID ${userId} not found`);
      error.code = grpc.status.NOT_FOUND;
      return callback(error);
    }

    const user = this.users.get(userId);
    callback(null, user);
  }

  // Unary RPC: Create a new user
  createUser(call, callback) {
    const { name, email } = call.request;

    // Validation
    if (!name || !email) {
      const error = new Error('Name and email are required');
      error.code = grpc.status.INVALID_ARGUMENT;
      return callback(error);
    }

    // Check if email already exists
    for (const user of this.users.values()) {
      if (user.email === email) {
        const error = new Error('Email already exists');
        error.code = grpc.status.ALREADY_EXISTS;
        return callback(error);
      }
    }

    // Create new user
    const user = {
      id: this.nextId,
      name,
      email,
      created_at: Date.now(),
      updated_at: Date.now()
    };

    this.users.set(this.nextId, user);
    this.nextId++;

    callback(null, user);
  }

  // Unary RPC: Update an existing user
  updateUser(call, callback) {
    const { user_id, name, email } = call.request;

    if (!this.users.has(user_id)) {
      const error = new Error(`User with ID ${user_id} not found`);
      error.code = grpc.status.NOT_FOUND;
      return callback(error);
    }

    const user = this.users.get(user_id);
    
    // Update fields
    if (name) user.name = name;
    if (email) user.email = email;
    user.updated_at = Date.now();

    this.users.set(user_id, user);
    callback(null, user);
  }

  // Unary RPC: Delete a user
  deleteUser(call, callback) {
    const userId = call.request.user_id;

    if (!this.users.has(userId)) {
      const error = new Error(`User with ID ${userId} not found`);
      error.code = grpc.status.NOT_FOUND;
      return callback(error);
    }

    this.users.delete(userId);
    
    const response = {
      success: true,
      message: `User ${userId} deleted successfully`
    };

    callback(null, response);
  }
}

module.exports = UserService;
```

### Step 3: Create the Server (src/server.js)

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');
const UserService = require('./services/userService');

// Load the protobuf definition
const PROTO_PATH = path.join(__dirname, '../protos/user.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const userProto = grpc.loadPackageDefinition(packageDefinition).user;

function main() {
  // Create gRPC server
  const server = new grpc.Server();

  // Create service instance
  const userService = new UserService();

  // Add service to server
  server.addService(userProto.UserService.service, {
    getUser: userService.getUser.bind(userService),
    createUser: userService.createUser.bind(userService),
    updateUser: userService.updateUser.bind(userService),
    deleteUser: userService.deleteUser.bind(userService),
  });

  // Start server
  const port = '0.0.0.0:50051';
  server.bindAsync(port, grpc.ServerCredentials.createInsecure(), (error, port) => {
    if (error) {
      console.error('Failed to start server:', error);
      return;
    }
    
    console.log(`gRPC server started on port ${port}`);
    server.start();
  });

  // Graceful shutdown
  process.on('SIGINT', () => {
    console.log('Shutting down gRPC server...');
    server.tryShutdown(() => {
      console.log('Server shut down.');
      process.exit(0);
    });
  });
}

main();
```

### Step 4: Create the Client (src/client.js)

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

// Load the protobuf definition
const PROTO_PATH = path.join(__dirname, '../protos/user.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const userProto = grpc.loadPackageDefinition(packageDefinition).user;

async function main() {
  // Create client
  const client = new userProto.UserService(
    'localhost:50051',
    grpc.credentials.createInsecure()
  );

  try {
    console.log('🚀 Starting gRPC client demo...\n');

    // 1. Create a user
    console.log('1. Creating a user...');
    const createUserRequest = {
      name: 'John Doe',
      email: 'john@example.com'
    };

    const createdUser = await callUnaryMethod(client, 'createUser', createUserRequest);
    console.log('✅ Created user:', createdUser);
    console.log('');

    // 2. Get the user
    console.log('2. Getting the user...');
    const getUserRequest = { user_id: createdUser.id };
    const retrievedUser = await callUnaryMethod(client, 'getUser', getUserRequest);
    console.log('✅ Retrieved user:', retrievedUser);
    console.log('');

    // 3. Update the user
    console.log('3. Updating the user...');
    const updateUserRequest = {
      user_id: createdUser.id,
      name: 'John Smith',
      email: 'johnsmith@example.com'
    };
    const updatedUser = await callUnaryMethod(client, 'updateUser', updateUserRequest);
    console.log('✅ Updated user:', updatedUser);
    console.log('');

    // 4. Try to get non-existent user (will fail)
    console.log('4. Trying to get non-existent user...');
    try {
      const nonExistentUserRequest = { user_id: 999 };
      await callUnaryMethod(client, 'getUser', nonExistentUserRequest);
    } catch (error) {
      console.log('❌ Expected error:', error.message);
    }
    console.log('');

    // 5. Delete the user
    console.log('5. Deleting the user...');
    const deleteUserRequest = { user_id: createdUser.id };
    const deleteResponse = await callUnaryMethod(client, 'deleteUser', deleteUserRequest);
    console.log('✅ Delete response:', deleteResponse);

  } catch (error) {
    console.error('❌ Client error:', error.message);
  } finally {
    client.close();
    console.log('\n🏁 Client demo completed.');
  }
}

// Helper function to promisify gRPC calls
function callUnaryMethod(client, methodName, request) {
  return new Promise((resolve, reject) => {
    client[methodName](request, (error, response) => {
      if (error) {
        reject(error);
      } else {
        resolve(response);
      }
    });
  });
}

// Run the client
main().catch(console.error);
```

### Step 5: Run the Example

```bash
# Terminal 1: Start the server
npm run dev

# Terminal 2: Run the client
npm run client
```

---

## 6. Service Types {#service-types}

gRPC supports four types of service methods. Let's expand our user service to include all types:

### Updated Proto File (protos/user.proto)

```protobuf
syntax = "proto3";

package user;

service UserService {
  // 1. Unary RPC (Request-Response)
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  
  // 2. Server Streaming RPC (One request, stream of responses)
  rpc ListUsers(ListUsersRequest) returns (stream User);
  rpc WatchUser(WatchUserRequest) returns (stream UserEvent);
  
  // 3. Client Streaming RPC (Stream of requests, one response)
  rpc CreateMultipleUsers(stream CreateUserRequest) returns (CreateMultipleUsersResponse);
  
  // 4. Bidirectional Streaming RPC (Stream both ways)
  rpc ChatWithUsers(stream ChatMessage) returns (stream ChatMessage);
}

message User {
  int32 id = 1;
  string name = 2;
  string email = 3;
  int64 created_at = 4;
  int64 updated_at = 5;
}

// Unary messages
message GetUserRequest {
  int32 user_id = 1;
}

message CreateUserRequest {
  string name = 1;
  string email = 2;
}

// Server streaming messages
message ListUsersRequest {
  int32 page_size = 1;
  int32 offset = 2;
}

message WatchUserRequest {
  int32 user_id = 1;
}

message UserEvent {
  string event_type = 1; // "created", "updated", "deleted"
  User user = 2;
  int64 timestamp = 3;
}

// Client streaming messages
message CreateMultipleUsersResponse {
  int32 created_count = 1;
  repeated int32 user_ids = 2;
  repeated string errors = 3;
}

// Bidirectional streaming messages
message ChatMessage {
  int32 user_id = 1;
  string username = 2;
  string message = 3;
  int64 timestamp = 4;
}
```

### Extended User Service (src/services/userService.js)

```javascript
const grpc = require('@grpc/grpc-js');
const { EventEmitter } = require('events');

class UserService extends EventEmitter {
  constructor() {
    super();
    this.users = new Map();
    this.nextId = 1;
    this.chatClients = new Set();
  }

  // 1. UNARY RPC - Get single user
  getUser(call, callback) {
    const userId = call.request.user_id;

    if (!this.users.has(userId)) {
      const error = new Error(`User with ID ${userId} not found`);
      error.code = grpc.status.NOT_FOUND;
      return callback(error);
    }

    callback(null, this.users.get(userId));
  }

  // 1. UNARY RPC - Create single user
  createUser(call, callback) {
    const { name, email } = call.request;

    if (!name || !email) {
      const error = new Error('Name and email are required');
      error.code = grpc.status.INVALID_ARGUMENT;
      return callback(error);
    }

    const user = {
      id: this.nextId,
      name,
      email,
      created_at: Date.now(),
      updated_at: Date.now()
    };

    this.users.set(this.nextId, user);
    this.nextId++;

    // Emit event for watchers
    this.emit('userEvent', {
      event_type: 'created',
      user,
      timestamp: Date.now()
    });

    callback(null, user);
  }

  // 2. SERVER STREAMING RPC - List all users
  listUsers(call) {
    const { page_size = 10, offset = 0 } = call.request;
    
    const users = Array.from(this.users.values())
      .slice(offset, offset + page_size);

    let index = 0;
    const sendNextUser = () => {
      if (index < users.length) {
        call.write(users[index]);
        index++;
        // Simulate delay for demo purposes
        setTimeout(sendNextUser, 500);
      } else {
        call.end();
      }
    };

    sendNextUser();
  }

  // 2. SERVER STREAMING RPC - Watch user changes
  watchUser(call) {
    const userId = call.request.user_id;

    if (!this.users.has(userId)) {
      const error = new Error(`User with ID ${userId} not found`);
      error.code = grpc.status.NOT_FOUND;
      call.emit('error', error);
      return;
    }

    // Send initial user state
    const initialEvent = {
      event_type: 'initial',
      user: this.users.get(userId),
      timestamp: Date.now()
    };
    call.write(initialEvent);

    // Listen for user events
    const eventHandler = (event) => {
      if (event.user.id === userId) {
        call.write(event);
      }
    };

    this.on('userEvent', eventHandler);

    // Clean up when client disconnects
    call.on('cancelled', () => {
      this.removeListener('userEvent', eventHandler);
    });
  }

  // 3. CLIENT STREAMING RPC - Create multiple users
  createMultipleUsers(call, callback) {
    const createdUsers = [];
    const errors = [];

    call.on('data', (createUserRequest) => {
      const { name, email } = createUserRequest;

      // Validate request
      if (!name || !email) {
        errors.push(`Invalid user data: name="${name}", email="${email}"`);
        return;
      }

      // Check for duplicate email
      for (const user of this.users.values()) {
        if (user.email === email) {
          errors.push(`Email already exists: ${email}`);
          return;
        }
      }

      // Create user
      const user = {
        id: this.nextId,
        name,
        email,
        created_at: Date.now(),
        updated_at: Date.now()
      };

      this.users.set(this.nextId, user);
      createdUsers.push(user);
      this.nextId++;

      // Emit event
      this.emit('userEvent', {
        event_type: 'created',
        user,
        timestamp: Date.now()
      });
    });

    call.on('end', () => {
      const response = {
        created_count: createdUsers.length,
        user_ids: createdUsers.map(user => user.id),
        errors
      };
      callback(null, response);
    });

    call.on('error', (error) => {
      console.error('Client streaming error:', error);
      callback(error);
    });
  }

  // 4. BIDIRECTIONAL STREAMING RPC - Chat system
  chatWithUsers(call) {
    // Add client to active chat clients
    this.chatClients.add(call);

    call.on('data', (chatMessage) => {
      const { user_id, username, message } = chatMessage;

      // Broadcast message to all connected clients
      const broadcastMessage = {
        user_id,
        username,
        message,
        timestamp: Date.now()
      };

      // Send to all clients except sender
      for (const client of this.chatClients) {
        if (client !== call) {
          try {
            client.write(broadcastMessage);
          } catch (error) {
            // Remove disconnected clients
            this.chatClients.delete(client);
          }
        }
      }
    });

    call.on('end', () => {
      this.chatClients.delete(call);
      call.end();
    });

    call.on('error', (error) => {
      console.error('Chat streaming error:', error);
      this.chatClients.delete(call);
    });

    // Send welcome message
    call.write({
      user_id: 0,
      username: 'System',
      message: 'Welcome to the chat!',
      timestamp: Date.now()
    });
  }
}

module.exports = UserService;
```

### Client Examples for All Service Types (src/streaming-client.js)

```javascript
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const path = require('path');

const PROTO_PATH = path.join(__dirname, '../protos/user.proto');

const packageDefinition = protoLoader.loadSync(PROTO_PATH, {
  keepCase: true,
  longs: String,
  enums: String,
  defaults: true,
  oneofs: true,
});

const userProto = grpc.loadPackageDefinition(packageDefinition).user;

class StreamingClient {
  constructor() {
    this.client = new userProto.UserService(
      'localhost:50051',
      grpc.credentials.createInsecure()
    );
  }

  // 2. SERVER STREAMING - List users
  async demoServerStreaming() {
    console.log('📡 Demo: Server Streaming (List Users)');
    
    return new Promise((resolve, reject) => {
      const call = this.client.listUsers({ page_size: 5, offset: 0 });
      
      call.on('data', (user) => {
        console.log('📥 Received user:', user);
      });
      
      call.on('end', () => {
        console.log('✅ Server streaming completed\n');
        resolve();
      });
      
      call.on('error', (error) => {
        console.error('❌ Server streaming error:', error.message);
        reject(error);
      });
    });
  }

  // 2. SERVER STREAMING - Watch user changes
  async demoWatchUser(userId) {
    console.log(`👀 Demo: Watching user ${userId} for changes`);
    
    const call = this.client.watchUser({ user_id: userId });
    
    call.on('data', (event) => {
      console.log('📥 User event:', event);
    });
    
    call.on('error', (error) => {
      console.error('❌ Watch user error:', error.message);
    });

    // Stop watching after 10 seconds
    setTimeout(() => {
      call.cancel();
      console.log('⏹ Stopped watching user\n');
    }, 10000);
  }

  // 3. CLIENT STREAMING - Create multiple users
  async demoClientStreaming() {
    console.log('📤 Demo: Client Streaming (Create Multiple Users)');
    
    return new Promise((resolve, reject) => {
      const call = this.client.createMultipleUsers((error, response) => {
        if (error) {
          console.error('❌ Client streaming error:', error.message);
          reject(error);
        } else {
          console.log('✅ Batch creation result:', response);
          console.log('');
          resolve(response);
        }
      });

      // Send multiple user creation requests
      const users = [
        { name: 'Alice Smith', email: 'alice@example.com' },
        { name: 'Bob Johnson', email: 'bob@example.com' },
        { name: 'Charlie Brown', email: 'charlie@example.com' },
        { name: 'Diana Wilson', email: 'diana@example.com' },
      ];

      users.forEach((user, index) => {
        setTimeout(() => {
          console.log(`📤 Sending user ${index + 1}:`, user);
          call.write(user);
          
          // End the stream after sending all users
          if (index === users.length - 1) {
            setTimeout(() => call.end(), 100);
          }
        }, index * 500);
      });
    });
  }

  // 4. BIDIRECTIONAL STREAMING - Chat
  async demoBidirectionalStreaming() {
    console.log('💬 Demo: Bidirectional Streaming (Chat)');
    
    const call = this.client.chatWithUsers();
    
    // Listen for incoming messages
    call.on('data', (message) => {
      console.log(`💬 ${message.username}: ${message.message}`);
    });
    
    call.on('error', (error) => {
      console.error('❌ Chat error:', error.message);
    });

    // Send some chat messages
    const messages = [
      'Hello everyone!',
      'How is everyone doing?',
      'This is a gRPC chat demo',
      'Pretty cool, right?'
    ];

    messages.forEach((message, index) => {
      setTimeout(() => {
        const chatMessage = {
          user_id: 1,
          username: 'DemoUser',
          message,
          timestamp: Date.now()
        };
        console.log(`📤 Sending: ${message}`);
        call.write(chatMessage);
      }, (index + 1) * 2000);
    });

    // End chat after 10 seconds
    setTimeout(() => {
      call.end();
      console.log('💬 Chat ended\n');
    }, 10000);
  }

  close() {
    this.client.close();
  }
}

// Run streaming demos
async function main() {
  const client = new StreamingClient();

  try {
    // First create some users for the streaming demos
    console.log('🚀 Setting up data for streaming demos...\n');
    
    // Create a few users first
    await client.createUser({ name: 'Test User 1', email: 'test1@example.com' });
    await client.createUser({ name: 'Test User 2', email: 'test2@example.com' });
    await client.createUser({ name: 'Test User 3', email: 'test3@example.com' });

    // Demo server streaming
    await client.demoServerStreaming();
    
    // Demo client streaming
    await client.demoClientStreaming();
    
    // Demo watch user (server streaming with events)
    client.demoWatchUser(1);
    
    // Demo bidirectional streaming
    await client.demoBidirectionalStreaming();
    
  } catch (error) {
    console.error('❌ Demo error:', error.message);
  } finally {
    setTimeout(() => {
      client.close();
      console.log('🏁 All streaming demos completed.');
    }, 12000);
  }
}

// Helper method for unary calls
StreamingClient.prototype.createUser = function(request) {
  return new Promise((resolve, reject) => {
    this.client.createUser(request, (error, response) => {
      if (error) reject(error);
      else resolve(response);
    });
  });
};

main().catch(console.error);
```

---

## 7. Advanced Features {#advanced-features}

### Authentication & Security

#### 1. SSL/TLS Encryption

```javascript
// server-ssl.js - Secure server
const grpc = require('@grpc/grpc-js');
const fs = require('fs');
const path = require('path');

function createSecureServer() {
  const server = new grpc.Server();
  const userService = new UserService();

  server.addService(userProto.UserService.service, {
    getUser: userService.getUser.bind(userService),
    createUser: userService.createUser.bind(userService),
    listUsers: userService.listUsers.bind(userService),
    createMultipleUsers: userService.createMultipleUsers.bind(userService),
  });

  // Load SSL certificates
  const certPath = path.join(__dirname, '../certs');
  const serverCert = fs.readFileSync(path.join(certPath, 'server-cert.pem'));
  const serverKey = fs.readFileSync(path.join(certPath, 'server-key.pem'));
  const caCert = fs.readFileSync(path.join(certPath, 'ca-cert.pem'));

  // Create SSL credentials
  const sslCredentials = grpc.ServerCredentials.createSsl(
    caCert,        // Root CA certificate
    [{
      private_key: serverKey,
      cert_chain: serverCert,
    }],
    true           // Require client certificate
  );

  const port = '0.0.0.0:50051';
  server.bindAsync(port, sslCredentials, (error, port) => {
    if (error) {
      console.error('Failed to start secure server:', error);
      return;
    }
    
    console.log(`🔒 Secure gRPC server started on port ${port}`);
    server.start();
  });
}

// client-ssl.js - Secure client
function createSecureClient() {
  const certPath = path.join(__dirname, '../certs');
  const caCert = fs.readFileSync(path.join(certPath, 'ca-cert.pem'));
  const clientCert = fs.readFileSync(path.join(certPath, 'client-cert.pem'));
  const clientKey = fs.readFileSync(path.join(certPath, 'client-key.pem'));

  const sslCredentials = grpc.credentials.createSsl(
    caCert,      // Root CA certificate
    clientKey,   // Client private key
    clientCert   // Client certificate
  );

  const client = new userProto.UserService(
    'localhost:50051',
    sslCredentials
  );

  return client;
}

// Generate self-signed certificates for testing
// Run these commands in your project root:
/*
# Create certificates directory
mkdir certs && cd certs

# Generate CA private key
openssl genrsa -out ca-key.pem 4096

# Generate CA certificate
openssl req -new -x509 -key ca-key.pem -sha256 -subj "/C=US/ST=CA/O=MyOrg/CN=MyCA" -days 3650 -out ca-cert.pem

# Generate server private key
openssl genrsa -out server-key.pem 4096

# Generate server certificate signing request
openssl req -new -key server-key.pem -out server.csr -subj "/C=US/ST=CA/O=MyOrg/CN=localhost"

# Generate server certificate
openssl x509 -req -in server.csr -CA ca-cert.pem -CAkey ca-key.pem -out server-cert.pem -days 365 -sha256

# Generate client private key
openssl genrsa -out client-key.pem 4096

# Generate client certificate signing request
openssl req -new -key client-key.pem -out client.csr -subj "/C=US/ST=CA/O=MyOrg/CN=client"

# Generate client certificate
openssl x509 -req -in client.csr -CA ca-cert.pem -CAkey ca-key.pem -out client-cert.pem -days 365 -sha256
*/
```

#### 2. Token-based Authentication with Interceptors

```javascript
// auth-interceptor.js - Server-side authentication
class AuthInterceptor {
  constructor() {
    this.validTokens = new Set(['valid-token-123', 'admin-token-456']);
  }

  intercept(methodDefinition, handler) {
    return (call, callback) => {
      // Get metadata from the call
      const metadata = call.metadata.getMap();
      const authorization = metadata.authorization;

      // Check for valid token
      if (!authorization || !this.validTokens.has(authorization)) {
        const error = new Error('Invalid or missing authorization token');
        error.code = grpc.status.UNAUTHENTICATED;
        
        if (callback) {
          return callback(error);
        } else {
          // For streaming calls
          call.emit('error', error);
          return;
        }
      }

      // Token is valid, proceed with the call
      return handler(call, callback);
    };
  }
}

// Apply interceptor to server
function createAuthenticatedServer() {
  const server = new grpc.Server();
  const userService = new UserService();
  const authInterceptor = new AuthInterceptor();

  // Wrap service methods with authentication
  const authenticatedService = {};
  for (const [methodName, method] of Object.entries({
    getUser: userService.getUser.bind(userService),
    createUser: userService.createUser.bind(userService),
    listUsers: userService.listUsers.bind(userService),
    createMultipleUsers: userService.createMultipleUsers.bind(userService),
  })) {
    authenticatedService[methodName] = authInterceptor.intercept(null, method);
  }

  server.addService(userProto.UserService.service, authenticatedService);

  const port = '0.0.0.0:50051';
  server.bindAsync(port, grpc.ServerCredentials.createInsecure(), (error, port) => {
    if (error) {
      console.error('Failed to start authenticated server:', error);
      return;
    }
    
    console.log(`🔐 Authenticated gRPC server started on port ${port}`);
    server.start();
  });
}

// client-auth.js - Client with authentication
function createAuthenticatedClient(token) {
  const client = new userProto.UserService(
    'localhost:50051',
    grpc.credentials.createInsecure()
  );

  // Add authentication metadata to all calls
  const authMetadata = new grpc.Metadata();
  authMetadata.add('authorization', token);

  // Wrapper function to add auth to all calls
  const originalMethods = {};
  
  // Store original methods
  Object.getOwnPropertyNames(Object.getPrototypeOf(client))
    .filter(name => typeof client[name] === 'function' && !name.startsWith('_'))
    .forEach(methodName => {
      originalMethods[methodName] = client[methodName].bind(client);
    });

  // Override methods to include auth metadata
  Object.keys(originalMethods).forEach(methodName => {
    client[methodName] = function(request, callbackOrMetadata, callback) {
      let metadata = authMetadata;
      let finalCallback = callback;

      // Handle different call signatures
      if (typeof callbackOrMetadata === 'function') {
        finalCallback = callbackOrMetadata;
      } else if (callbackOrMetadata && callbackOrMetadata instanceof grpc.Metadata) {
        // Merge provided metadata with auth metadata
        metadata = callbackOrMetadata.clone();
        metadata.add('authorization', token);
      }

      return originalMethods[methodName](request, metadata, finalCallback);
    };
  });

  return client;
}

// Usage example
async function testAuthenticatedClient() {
  // Valid token
  const authenticatedClient = createAuthenticatedClient('valid-token-123');
  
  try {
    const user = await callUnaryMethod(authenticatedClient, 'createUser', {
      name: 'Authenticated User',
      email: 'auth@example.com'
    });
    console.log('✅ Authenticated call successful:', user);
  } catch (error) {
    console.error('❌ Authenticated call failed:', error.message);
  }

  // Invalid token
  const unauthenticatedClient = createAuthenticatedClient('invalid-token');
  
  try {
    await callUnaryMethod(unauthenticatedClient, 'createUser', {
      name: 'Unauthorized User',
      email: 'unauth@example.com'
    });
  } catch (error) {
    console.error('❌ Expected authentication error:', error.message);
  }
}
```

### Error Handling and Retry Logic

```javascript
// error-handling.js - Advanced error handling
class RetryableClient {
  constructor(address, credentials, options = {}) {
    this.client = new userProto.UserService(address, credentials);
    this.maxRetries = options.maxRetries || 3;
    this.retryDelay = options.retryDelay || 1000;
    this.retryableStatusCodes = new Set([
      grpc.status.UNAVAILABLE,
      grpc.status.RESOURCE_EXHAUSTED,
      grpc.status.ABORTED,
    ]);
  }

  async callWithRetry(methodName, request, metadata = null) {
    let lastError;
    
    for (let attempt = 0; attempt <= this.maxRetries; attempt++) {
      try {
        return await this.makeCall(methodName, request, metadata);
      } catch (error) {
        lastError = error;
        
        // Don't retry on non-retryable errors
        if (!this.isRetryable(error) || attempt === this.maxRetries) {
          throw error;
        }
        
        console.log(`⚠️ Attempt ${attempt + 1} failed, retrying in ${this.retryDelay}ms...`);
        
        // Exponential backoff
        await this.sleep(this.retryDelay * Math.pow(2, attempt));
      }
    }
    
    throw lastError;
  }

  makeCall(methodName, request, metadata) {
    return new Promise((resolve, reject) => {
      const call = metadata 
        ? this.client[methodName](request, metadata, (error, response) => {
            if (error) reject(error);
            else resolve(response);
          })
        : this.client[methodName](request, (error, response) => {
            if (error) reject(error);
            else resolve(response);
          });
    });
  }

  isRetryable(error) {
    return this.retryableStatusCodes.has(error.code);
  }

  sleep(ms) {
    return new Promise(resolve => setTimeout(resolve, ms));
  }

  close() {
    this.client.close();
  }
}

// Custom error classes for better error handling
class UserServiceError extends Error {
  constructor(message, code, details = null) {
    super(message);
    this.name = 'UserServiceError';
    this.code = code;
    this.details = details;
  }
}

class ValidationError extends UserServiceError {
  constructor(message, field = null) {
    super(message, grpc.status.INVALID_ARGUMENT);
    this.name = 'ValidationError';
    this.field = field;
  }
}

class NotFoundError extends UserServiceError {
  constructor(resource, id) {
    super(`${resource} with ID ${id} not found`, grpc.status.NOT_FOUND);
    this.name = 'NotFoundError';
    this.resource = resource;
    this.id = id;
  }
}

// Enhanced user service with better error handling
class EnhancedUserService extends UserService {
  createUser(call, callback) {
    try {
      const { name, email } = call.request;

      // Validate input
      if (!name || name.trim().length === 0) {
        throw new ValidationError('Name is required and cannot be empty', 'name');
      }

      if (!email || !this.isValidEmail(email)) {
        throw new ValidationError('Valid email is required', 'email');
      }

      // Check for duplicate email
      for (const user of this.users.values()) {
        if (user.email.toLowerCase() === email.toLowerCase()) {
          const error = new Error('Email already exists');
          error.code = grpc.status.ALREADY_EXISTS;
          error.details = { email, existing_user_id: user.id };
          throw error;
        }
      }

      // Create user
      const user = {
        id: this.nextId,
        name: name.trim(),
        email: email.toLowerCase(),
        created_at: Date.now(),
        updated_at: Date.now()
      };

      this.users.set(this.nextId, user);
      this.nextId++;

      callback(null, user);
    } catch (error) {
      callback(error);
    }
  }

  getUser(call, callback) {
    try {
      const userId = call.request.user_id;

      if (!userId || userId <= 0) {
        throw new ValidationError('Valid user ID is required', 'user_id');
      }

      if (!this.users.has(userId)) {
        throw new NotFoundError('User', userId);
      }

      callback(null, this.users.get(userId));
    } catch (error) {
      callback(error);
    }
  }

  isValidEmail(email) {
    const emailRegex = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;
    return emailRegex.test(email);
  }
}
```

### Metadata and Context

```javascript
// metadata-example.js - Working with metadata
class MetadataAwareService extends UserService {
  createUser(call, callback) {
    // Read metadata from client
    const metadata = call.metadata.getMap();
    const clientId = metadata['client-id'] || 'unknown';
    const apiVersion = metadata['api-version'] || '1.0';
    const requestId = metadata['request-id'] || this.generateRequestId();

    console.log(`📧 Request from client: ${clientId}, API version: ${apiVersion}, Request ID: ${requestId}`);

    // Set response metadata
    const responseMetadata = new grpc.Metadata();
    responseMetadata.add('server-version', '2.0');
    responseMetadata.add('request-id', requestId);
    responseMetadata.add('response-time', Date.now().toString());

    call.sendMetadata(responseMetadata);

    // Log request for audit purposes
    this.logRequest(requestId, clientId, 'createUser', call.request);

    // Call parent implementation
    super.createUser(call, (error, response) => {
      if (error) {
        // Add error metadata
        const errorMetadata = new grpc.Metadata();
        errorMetadata.add('error-id', this.generateRequestId());
        errorMetadata.add('error-timestamp', Date.now().toString());
        
        // Attach metadata to error (this is server-specific)
        error.metadata = errorMetadata;
      }
      
      callback(error, response);
    });
  }

  generateRequestId() {
    return `req_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  logRequest(requestId, clientId, method, request) {
    const logEntry = {
      timestamp: new Date().toISOString(),
      requestId,
      clientId,
      method,
      request: JSON.stringify(request),
    };
    console.log('📋 Audit log:', logEntry);
  }
}

// Client with metadata
class MetadataClient {
  constructor() {
    this.client = new userProto.UserService(
      'localhost:50051',
      grpc.credentials.createInsecure()
    );
    this.clientId = `client_${Math.random().toString(36).substr(2, 9)}`;
  }

  async createUserWithMetadata(userData) {
    return new Promise((resolve, reject) => {
      // Create metadata
      const metadata = new grpc.Metadata();
      metadata.add('client-id', this.clientId);
      metadata.add('api-version', '2.0');
      metadata.add('request-id', `req_${Date.now()}`);
      metadata.add('user-agent', 'gRPC-NodeJS-Client/1.0');

      const call = this.client.createUser(userData, metadata, (error, response) => {
        if (error) {
          console.error('❌ Error metadata:', error.metadata?.getMap());
          reject(error);
        } else {
          resolve(response);
        }
      });

      // Listen for server metadata
      call.on('metadata', (metadata) => {
        console.log('📥 Server metadata:', metadata.getMap());
      });
    });
  }
}
```

### Timeouts and Deadlines

```javascript
// timeout-example.js - Handling timeouts and deadlines
class TimeoutAwareClient {
  constructor() {
    this.client = new userProto.UserService(
      'localhost:50051',
      grpc.credentials.createInsecure()
    );
  }

  // Call with timeout
  async callWithTimeout(methodName, request, timeoutMs = 5000) {
    return new Promise((resolve, reject) => {
      // Set deadline
      const deadline = new Date();
      deadline.setMilliseconds(deadline.getMilliseconds() + timeoutMs);

      const call = this.client[methodName](
        request,
        { deadline },
        (error, response) => {
          if (error) {
            if (error.code === grpc.status.DEADLINE_EXCEEDED) {
              reject(new Error(`Request timed out after ${timeoutMs}ms`));
            } else {
              reject(error);
            }
          } else {
            resolve(response);
          }
        }
      );
    });
  }

  // Cancellable call
  createCancellableCall(methodName, request) {
    let call;
    
    const promise = new Promise((resolve, reject) => {
      call = this.client[methodName](request, (error, response) => {
        if (error) {
          if (error.code === grpc.status.CANCELLED) {
            reject(new Error('Request was cancelled'));
          } else {
            reject(error);
          }
        } else {
          resolve(response);
        }
      });
    });

    // Return both promise and cancel function
    return {
      promise,
      cancel: () => {
        if (call) {
          call.cancel();
        }
      }
    };
  }
}

// Usage examples
async function timeoutExamples() {
  const client = new TimeoutAwareClient();

  try {
    // Call with 3-second timeout
    const user = await client.callWithTimeout('createUser', {
      name: 'Fast User',
      email: 'fast@example.com'
    }, 3000);
    console.log('✅ Fast call completed:', user);
  } catch (error) {
    console.error('❌ Fast call failed:', error.message);
  }

  // Cancellable call
  const { promise, cancel } = client.createCancellableCall('createUser', {
    name: 'Slow User',
    email: 'slow@example.com'
  });

  // Cancel the call after 1 second
  setTimeout(() => {
    cancel();
    console.log('🛑 Call cancelled');
  }, 1000);

  try {
    await promise;
  } catch (error) {
    console.error('❌ Cancelled call error:', error.message);
  }
}
```

---

## 8. Best Practices {#best-practices}

### 1. Service Design Principles

```javascript
// best-practices.js - Service design best practices

// ✅ GOOD: Focused, cohesive service
const UserServiceProto = `
syntax = "proto3";
package user.v1;

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc CreateUser(CreateUserRequest) returns (User);
  rpc UpdateUser(UpdateUserRequest) returns (User);
  rpc DeleteUser(DeleteUserRequest) returns (DeleteUserResponse);
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse);
}
`;

// ❌ BAD: Mixed concerns
const BadServiceProto = `
service EverythingService {
  rpc GetUser(GetUserRequest) returns (User);
  rpc SendEmail(EmailRequest) returns (EmailResponse);
  rpc ProcessPayment(PaymentRequest) returns (PaymentResponse);
  rpc UploadFile(stream FileChunk) returns (UploadResponse);
}
`;

// ✅ GOOD: Consistent naming conventions
const GoodNaming = `
message User {
  int32 user_id = 1;        // snake_case for fields
  string first_name = 2;
  string last_name = 3;
  repeated string email_addresses = 4;  // plural for repeated
}

service UserService {         // PascalCase for services
  rpc GetUser(GetUserRequest) returns (User);  // PascalCase for methods
}
`;

// ✅ GOOD: Versioned APIs
const VersionedAPI = `
syntax = "proto3";
package mycompany.user.v1;  // Include version in package

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
}

// When you need breaking changes, create v2
package mycompany.user.v2;
`;
```

### 2. Error Handling Strategy

```javascript
// error-strategy.js - Comprehensive error handling
class ServiceErrorHandler {
  static createError(code, message, details = null) {
    const error = new Error(message);
    error.code = code;
    error.details = details;
    return error;
  }

  static handleValidationError(field, message) {
    return this.createError(
      grpc.status.INVALID_ARGUMENT,
      `Validation failed for field '${field}': ${message}`,
      { field, type: 'validation' }
    );
  }

  static handleNotFound(resource, id) {
    return this.createError(
      grpc.status.NOT_FOUND,
      `${resource} with ID '${id}' not found`,
      { resource, id, type: 'not_found' }
    );
  }

  static handleDuplicate(resource, field, value) {
    return this.createError(
      grpc.status.ALREADY_EXISTS,
      `${resource} with ${field} '${value}' already exists`,
      { resource, field, value, type: 'duplicate' }
    );
  }

  static handlePermissionDenied(action, resource) {
    return this.createError(
      grpc.status.PERMISSION_DENIED,
      `Permission denied: cannot ${action} ${resource}`,
      { action, resource, type: 'permission' }
    );
  }
}

// Usage in service methods
class BestPracticeUserService extends UserService {
  createUser(call, callback) {
    try {
      const { name, email } = call.request;

      // Input validation
      if (!name?.trim()) {
        throw ServiceErrorHandler.handleValidationError('name', 'is required');
      }

      if (!email?.trim()) {
        throw ServiceErrorHandler.handleValidationError('email', 'is required');
      }

      if (!this.isValidEmail(email)) {
        throw ServiceErrorHandler.handleValidationError('email', 'must be valid format');
      }

      // Business logic validation
      if (this.emailExists(email)) {
        throw ServiceErrorHandler.handleDuplicate('User', 'email', email);
      }

      // Create user
      const user = this.doCreateUser(name.trim(), email.trim());
      callback(null, user);

    } catch (error) {
      // Log error for debugging
      console.error('Create user error:', {
        error: error.message,
        code: error.code,
        details: error.details,
        request: call.request
      });
      
      callback(error);
    }
  }

  isValidEmail(email) {
    return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
  }

  emailExists(email) {
    return Array.from(this.users.values())
      .some(user => user.email.toLowerCase() === email.toLowerCase());
  }

  doCreateUser(name, email) {
    const user = {
      id: this.nextId++,
      name,
      email: email.toLowerCase(),
      created_at: Date.now(),
      updated_at: Date.now()
    };

    this.users.set(user.id, user);
    return user;
  }
}
```

### 3. Performance Optimization

```javascript
// performance.js - Performance optimization techniques
class PerformanceOptimizedService extends UserService {
  constructor() {
    super();
    
    // Connection pooling for database connections
    this.connectionPool = this.createConnectionPool();
    
    // Caching frequently accessed data
    this.cache = new Map();
    this.cacheExpiry = new Map();
    this.cacheTimeout = 5 * 60 * 1000; // 5 minutes
    
    // Rate limiting
    this.rateLimiter = new Map(); // client_id -> { count, resetTime }
    this.rateLimit = 100; // requests per minute
  }

  // Batch operations for better performance
  async createMultipleUsers(call, callback) {
    const users = [];
    const errors = [];
    let batchSize = 0;
    const maxBatchSize = 100;

    call.on('data', async (request) => {
      batchSize++;
      
      // Process in batches to avoid memory issues
      if (batchSize >= maxBatchSize) {
        console.log('⚠️ Batch size limit reached, processing...');
        // In a real app, you'd flush the current batch to database
      }

      try {
        const user = await this.validateAndCreateUser(request);
        users.push(user);
      } catch (error) {
        errors.push({
          request,
          error: error.message
        });
      }
    });

    call.on('end', () => {
      const response = {
        created_count: users.length,
        user_ids: users.map(u => u.id),
        errors: errors.map(e => e.error)
      };
      callback(null, response);
    });
  }

  // Implement caching for frequently accessed data
  async getUserWithCache(call, callback) {
    const userId = call.request.user_id;
    const cacheKey = `user:${userId}`;

    // Check cache first
    if (this.cache.has(cacheKey) && !this.isCacheExpired(cacheKey)) {
      console.log('💾 Cache hit for user:', userId);
      return callback(null, this.cache.get(cacheKey));
    }

    // Cache miss - get from database
    try {
      const user = await this.getUserFromDatabase(userId);
      
      if (user) {
        // Store in cache
        this.cache.set(cacheKey, user);
        this.cacheExpiry.set(cacheKey, Date.now() + this.cacheTimeout);
        console.log('💾 User cached:', userId);
      }
      
      callback(null, user);
    } catch (error) {
      callback(error);
    }
  }

  // Rate limiting middleware
  checkRateLimit(clientId) {
    const now = Date.now();
    const clientData = this.rateLimiter.get(clientId) || { count: 0, resetTime: now + 60000 };

    // Reset counter if time window expired
    if (now > clientData.resetTime) {
      clientData.count = 0;
      clientData.resetTime = now + 60000;
    }

    clientData.count++;
    this.rateLimiter.set(clientId, clientData);

    if (clientData.count > this.rateLimit) {
      const error = new Error('Rate limit exceeded');
      error.code = grpc.status.RESOURCE_EXHAUSTED;
      throw error;
    }
  }

  isCacheExpired(key) {
    const expiry = this.cacheExpiry.get(key);
    return !expiry || Date.now() > expiry;
  }

  async getUserFromDatabase(userId) {
    // Simulate database call
    return new Promise((resolve) => {
      setTimeout(() => {
        resolve(this.users.get(userId));
      }, 10); // Simulate 10ms database latency
    });
  }

  createConnectionPool() {
    // In a real application, you'd create a proper connection pool
    return {
      acquire: () => Promise.resolve({}),
      release: () => {},
      destroy: () => {}
    };
  }
}

// Client-side connection pooling
class ConnectionPoolClient {
  constructor(address, credentials, poolSize = 5) {
    this.address = address;
    this.credentials = credentials;
    this.pool = [];
    this.currentIndex = 0;

    // Create connection pool
    for (let i = 0; i < poolSize; i++) {
      this.pool.push(new userProto.UserService(address, credentials));
    }
  }

  getClient() {
    const client = this.pool[this.currentIndex];
    this.currentIndex = (this.currentIndex + 1) % this.pool.length;
    return client;
  }

  async callMethod(methodName, request) {
    const client = this.getClient();
    return new Promise((resolve, reject) => {
      client[methodName](request, (error, response) => {
        if (error) reject(error);
        else resolve(response);
      });
    });
  }

  close() {
    this.pool.forEach(client => client.close());
  }
}
```

### 4. Testing Strategies

```javascript
// testing.js - Comprehensive testing approach
const assert = require('assert');
const grpc = require('@grpc/grpc-js');

class UserServiceTester {
  constructor() {
    this.server = null;
    this.client = null;
  }

  async setup() {
    // Create test server
    this.server = new grpc.Server();
    const userService = new UserService();

    this.server.addService(userProto.UserService.service, {
      getUser: userService.getUser.bind(userService),
      createUser: userService.createUser.bind(userService),
      updateUser: userService.updateUser.bind(userService),
      deleteUser: userService.deleteUser.bind(userService),
    });

    // Start server on random port
    const port = await this.startServer();
    
    // Create test client
    this.client = new userProto.UserService(
      `localhost:${port}`,
      grpc.credentials.createInsecure()
    );
  }

  async teardown() {
    if (this.client) {
      this.client.close();
    }
    
    if (this.server) {
      await new Promise((resolve) => {
        this.server.tryShutdown(resolve);
      });
    }
  }

  async startServer() {
    return new Promise((resolve, reject) => {
      this.server.bindAsync(
        '127.0.0.1:0', // Random port
        grpc.ServerCredentials.createInsecure(),
        (error, port) => {
          if (error) {
            reject(error);
          } else {
            this.server.start();
            resolve(port);
          }
        }
      );
    });
  }

  // Unit tests
  async testCreateUser() {
    console.log('🧪 Testing user creation...');
    
    const request = {
      name: 'Test User',
      email: 'test@example.com'
    };

    const user = await this.callMethod('createUser', request);
    
    assert.strictEqual(user.name, request.name);
    assert.strictEqual(user.email, request.email);
    assert(user.id > 0);
    assert(user.created_at > 0);
    
    console.log('✅ User creation test passed');
    return user;
  }

  async testGetUser(userId) {
    console.log('🧪 Testing user retrieval...');
    
    const request = { user_id: userId };
    const user = await this.callMethod('getUser', request);
    
    assert.strictEqual(user.id, userId);
    assert(user.name);
    assert(user.email);
    
    console.log('✅ User retrieval test passed');
    return user;
  }

  async testGetNonExistentUser() {
    console.log('🧪 Testing non-existent user retrieval...');
    
    try {
      await this.callMethod('getUser', { user_id: 999 });
      assert.fail('Expected error for non-existent user');
    } catch (error) {
      assert.strictEqual(error.code, grpc.status.NOT_FOUND);
      console.log('✅ Non-existent user test passed');
    }
  }

  async testCreateUserValidation() {
    console.log('🧪 Testing user creation validation...');
    
    // Test missing name
    try {
      await this.callMethod('createUser', { email: 'test@example.com' });
      assert.fail('Expected validation error for missing name');
    } catch (error) {
      assert.strictEqual(error.code, grpc.status.INVALID_ARGUMENT);
    }

    // Test missing email
    try {
      await this.callMethod('createUser', { name: 'Test User' });
      assert.fail('Expected validation error for missing email');
    } catch (error) {
      assert.strictEqual(error.code, grpc.status.INVALID_ARGUMENT);
    }

    console.log('✅ User validation tests passed');
  }

  async testConcurrentOperations() {
    console.log('🧪 Testing concurrent operations...');
    
    const promises = [];
    const userCount = 10;

    // Create multiple users concurrently
    for (let i = 0; i < userCount; i++) {
      promises.push(
        this.callMethod('createUser', {
          name: `Concurrent User ${i}`,
          email: `user${i}@example.com`
        })
      );
    }

    const users = await Promise.all(promises);
    assert.strictEqual(users.length, userCount);
    
    // Verify all users have unique IDs
    const ids = users.map(u => u.id);
    const uniqueIds = new Set(ids);
    assert.strictEqual(uniqueIds.size, userCount);

    console.log('✅ Concurrent operations test passed');
  }

  async runAllTests() {
    try {
      await this.setup();
      
      console.log('🚀 Starting gRPC service tests...\n');

      // Run tests
      await this.testCreateUserValidation();
      const user = await this.testCreateUser();
      await this.testGetUser(user.id);
      await this.testGetNonExistentUser();
      await this.testConcurrentOperations();

      console.log('\n🎉 All tests passed!');
    } catch (error) {
      console.error('❌ Test failed:', error);
      throw error;
    } finally {
      await this.teardown();
    }
  }

  callMethod(methodName, request) {
    return new Promise((resolve, reject) => {
      this.client[methodName](request, (error, response) => {
        if (error) reject(error);
        else resolve(response);
      });
    });
  }
}

// Load testing
class LoadTester {
  constructor(address, concurrency = 10) {
    this.address = address;
    this.concurrency = concurrency;
    this.clients = [];
    
    // Create multiple clients for load testing
    for (let i = 0; i < concurrency; i++) {
      this.clients.push(new userProto.UserService(
        address,
        grpc.credentials.createInsecure()
      ));
    }
  }

  async runLoadTest(duration = 10000) {
    console.log(`🔥 Starting load test for ${duration}ms with ${this.concurrency} concurrent clients...`);
    
    const startTime = Date.now();
    const results = {
      totalRequests: 0,
      successfulRequests: 0,
      failedRequests: 0,
      averageLatency: 0,
      maxLatency: 0,
      minLatency: Infinity
    };

    const promises = [];

    // Start concurrent load generators
    for (let i = 0; i < this.concurrency; i++) {
      promises.push(this.loadGenerator(i, startTime + duration, results));
    }

    await Promise.all(promises);

    const totalTime = Date.now() - startTime;
    const requestsPerSecond = (results.totalRequests / totalTime) * 1000;

    console.log('📊 Load test results:');
    console.log(`   Total requests: ${results.totalRequests}`);
    console.log(`   Successful: ${results.successfulRequests}`);
    console.log(`   Failed: ${results.failedRequests}`);
    console.log(`   Requests/second: ${requestsPerSecond.toFixed(2)}`);
    console.log(`   Average latency: ${results.averageLatency.toFixed(2)}ms`);
    console.log(`   Min latency: ${results.minLatency.toFixed(2)}ms`);
    console.log(`   Max latency: ${results.maxLatency.toFixed(2)}ms`);

    this.cleanup();
  }

  async loadGenerator(clientIndex, endTime, results) {
    const client = this.clients[clientIndex];
    
    while (Date.now() < endTime) {
      const requestStart = Date.now();
      
      try {
        await this.makeRequest(client, clientIndex);
        
        const latency = Date.now() - requestStart;
        results.successfulRequests++;
        results.averageLatency = (results.averageLatency * (results.totalRequests) + latency) / (results.totalRequests + 1);
        results.maxLatency = Math.max(results.maxLatency, latency);
        results.minLatency = Math.min(results.minLatency, latency);
        
      } catch (error) {
        results.failedRequests++;
      }
      
      results.totalRequests++;
      
      // Small delay to prevent overwhelming the server
      await new Promise(resolve => setTimeout(resolve, 10));
    }
  }

  makeRequest(client, clientIndex) {
    return new Promise((resolve, reject) => {
      const request = {
        name: `Load Test User ${clientIndex}_${Date.now()}`,
        email: `loadtest${clientIndex}_${Date.now()}@example.com`
      };

      client.createUser(request, (error, response) => {
        if (error) reject(error);
        else resolve(response);
      });
    });
  }

  cleanup() {
    this.clients.forEach(client => client.close());
  }
}

// Run tests
async function runTests() {
  const tester = new UserServiceTester();
  await tester.runAllTests();
}

async function runLoadTest() {
  // Make sure server is running first
  const loadTester = new LoadTester('localhost:50051', 5);
  await loadTester.runLoadTest(5000); // 5 second load test
}

// Export for use
module.exports = { UserServiceTester, LoadTester, runTests, runLoadTest };
```

---

## 9. Real-World Examples {#real-world-examples}

### Example 1: Real-time Chat Service

```javascript
// chat-service.js - Complete chat application
const grpc = require('@grpc/grpc-js');
const protoLoader = require('@grpc/proto-loader');
const { EventEmitter } = require('events');

// Chat proto definition
const chatProto = `
syntax = "proto3";
package chat;

service ChatService {
  rpc JoinRoom(JoinRoomRequest) returns (stream ChatMessage);
  rpc SendMessage(SendMessageRequest) returns (SendMessageResponse);
  rpc LeaveRoom(LeaveRoomRequest) returns (LeaveRoomResponse);
  rpc GetRoomHistory(GetRoomHistoryRequest) returns (GetRoomHistoryResponse);
  rpc CreateRoom(CreateRoomRequest) returns (Room);
  rpc ListRooms(ListRoomsRequest) returns (ListRoomsResponse);
}

message ChatMessage {
  string message_id = 1;
  string room_id = 2;
  string user_id = 3;
  string username = 4;
  string content = 5;
  int64 timestamp = 6;
  MessageType type = 7;
}

enum MessageType {
  USER_MESSAGE = 0;
  SYSTEM_MESSAGE = 1;
  USER_JOINED = 2;
  USER_LEFT = 3;
}

message JoinRoomRequest {
  string room_id = 1;
  string user_id = 2;
  string username = 3;
}

message SendMessageRequest {
  string room_id = 1;
  string user_id = 2;
  string content = 3;
}

message SendMessageResponse {
  string message_id = 1;
  bool success = 2;
  string error = 3;
}

message LeaveRoomRequest {
  string room_id = 1;
  string user_id = 2;
}

message LeaveRoomResponse {
  bool success = 1;
}

message GetRoomHistoryRequest {
  string room_id = 1;
  int32 limit = 2;
  string before_message_id = 3;
}

message GetRoomHistoryResponse {
  repeated ChatMessage messages = 1;
  bool has_more = 2;
}

message CreateRoomRequest {
  string name = 1;
  string description = 2;
  string created_by = 3;
}

message Room {
  string room_id = 1;
  string name = 2;
  string description = 3;
  string created_by = 4;
  int64 created_at = 5;
  int32 member_count = 6;
}

message ListRoomsRequest {
  int32 limit = 1;
  int32 offset = 2;
}

message ListRoomsResponse {
  repeated Room rooms = 1;
  int32 total_count = 2;
}
`;

class ChatService extends EventEmitter {
  constructor() {
    super();
    this.rooms = new Map(); // room_id -> Room
    this.roomMessages = new Map(); // room_id -> Message[]
    this.roomMembers = new Map(); // room_id -> Set<user_id>
    this.activeStreams = new Map(); // room_id -> Set<stream>
    this.userStreams = new Map(); // user_id -> stream
  }

  // Join a chat room and start receiving messages
  joinRoom(call) {
    const { room_id, user_id, username } = call.request;

    console.log(`👤 ${username} joining room ${room_id}`);

    // Validate room exists
    if (!this.rooms.has(room_id)) {
      const error = new Error(`Room ${room_id} not found`);
      error.code = grpc.status.NOT_FOUND;
      call.emit('error', error);
      return;
    }

    // Add user to room members
    if (!this.roomMembers.has(room_id)) {
      this.roomMembers.set(room_id, new Set());
    }
    this.roomMembers.get(room_id).add(user_id);

    // Add stream to active streams
    if (!this.activeStreams.has(room_id)) {
      this.activeStreams.set(room_id, new Set());
    }
    this.activeStreams.get(room_id).add(call);
    this.userStreams.set(user_id, call);

    // Send recent message history
    const history = this.roomMessages.get(room_id) || [];
    const recentMessages = history.slice(-20); // Last 20 messages
    
    recentMessages.forEach(message => {
      try {
        call.write(message);
      } catch (error) {
        console.error('Error sending history message:', error);
      }
    });

    // Send join notification to all users in room
    const joinMessage = {
      message_id: this.generateId(),
      room_id,
      user_id: 'system',
      username: 'System',
      content: `${username} joined the room`,
      timestamp: Date.now(),
      type: 2 // USER_JOINED
    };

    this.broadcastToRoom(room_id, joinMessage, user_id);
    this.addMessageToHistory(room_id, joinMessage);

    // Handle client disconnect
    call.on('cancelled', () => {
      this.handleUserLeave(room_id, user_id, username);
    });

    call.on('error', (error) => {
      console.error(`Stream error for user ${user_id}:`, error);
      this.handleUserLeave(room_id, user_id, username);
    });
  }

  // Send a message to a room
  sendMessage(call, callback) {
    const { room_id, user_id, content } = call.request;

    try {
      // Validate room and user
      if (!this.rooms.has(room_id)) {
        throw new Error(`Room ${room_id} not found`);
      }

      if (!this.roomMembers.get(room_id)?.has(user_id)) {
        throw new Error(`User ${user_id} not in room ${room_id}`);
      }

      // Get username (in real app, get from user service)
      const username = `User_${user_id}`;

      // Create message
      const message = {
        message_id: this.generateId(),
        room_id,
        user_id,
        username,
        content,
        timestamp: Date.now(),
        type: 0 // USER_MESSAGE
      };

      // Broadcast to all users in room
      this.broadcastToRoom(room_id, message);
      this.addMessageToHistory(room_id, message);

      callback(null, {
        message_id: message.message_id,
        success: true,
        error: ''
      });

    } catch (error) {
      callback(null, {
        message_id: '',
        success: false,
        error: error.message
      });
    }
  }

  // Create a new chat room
  createRoom(call, callback) {
    const { name, description, created_by } = call.request;

    try {
      if (!name || !created_by) {
        throw new Error('Room name and creator are required');
      }

      const room = {
        room_id: this.generateId(),
        name,
        description: description || '',
        created_by,
        created_at: Date.now(),
        member_count: 0
      };

      this.rooms.set(room.room_id, room);
      this.roomMessages.set(room.room_id, []);

      console.log(`🏠 Created room: ${room.name} (${room.room_id})`);
      callback(null, room);

    } catch (error) {
      const grpcError = new Error(error.message);
      grpcError.code = grpc.status.INVALID_ARGUMENT;
      callback(grpcError);
    }
  }

  // List available rooms
  listRooms(call, callback) {
    const { limit = 10, offset = 0 } = call.request;

    const allRooms = Array.from(this.rooms.values());
    const totalCount = allRooms.length;
    const rooms = allRooms.slice(offset, offset + limit);

    // Update member counts
    rooms.forEach(room => {
      const members = this.roomMembers.get(room.room_id);
      room.member_count = members ? members.size : 0;
    });

    callback(null, {
      rooms,
      total_count: totalCount
    });
  }

  // Get room message history
  getRoomHistory(call, callback) {
    const { room_id, limit = 50 } = call.request;

    if (!this.rooms.has(room_id)) {
      const error = new Error(`Room ${room_id} not found`);
      error.code = grpc.status.NOT_FOUND;
      return callback(error);
    }

    const messages = this.roomMessages.get(room_id) || [];
    const limitedMessages = messages.slice(-limit);

    callback(null, {
      messages: limitedMessages,
      has_more: messages.length > limit
    });
  }

  // Handle user leaving room
  leaveRoom(call, callback) {
    const { room_id, user_id } = call.request;
    
    // Get username (in real app, get from user service)
    const username = `User_${user_id}`;
    
    this.handleUserLeave(room_id, user_id, username);
    
    callback(null, { success: true });
  }

  // Internal methods
  handleUserLeave(roomId, userId, username) {
    console.log(`👋 ${username} leaving room ${roomId}`);

    // Remove from room members
    this.roomMembers.get(roomId)?.delete(userId);

    // Remove stream
    const userStream = this.userStreams.get(userId);
    if (userStream) {
      this.activeStreams.get(roomId)?.delete(userStream);
      this.userStreams.delete(userId);
    }

    // Send leave notification
    const leaveMessage = {
      message_id: this.generateId(),
      room_id: roomId,
      user_id: 'system',
      username: 'System',
      content: `${username} left the room`,
      timestamp: Date.now(),
      type: 3 // USER_LEFT
    };

    this.broadcastToRoom(roomId, leaveMessage, userId);
    this.addMessageToHistory(roomId, leaveMessage);
  }

  broadcastToRoom(roomId, message, excludeUserId = null) {
    const streams = this.activeStreams.get(roomId);
    if (!streams) return;

    for (const stream of streams) {
      // Skip the sender if specified
      if (excludeUserId && this.getStreamUserId(stream) === excludeUserId) {
        continue;
      }

      try {
        stream.write(message);
      } catch (error) {
        console.error('Error broadcasting message:', error);
        streams.delete(stream);
      }
    }
  }

  addMessageToHistory(roomId, message) {
    if (!this.roomMessages.has(roomId)) {
      this.roomMessages.set(roomId, []);
    }

    const messages = this.roomMessages.get(roomId);
    messages.push(message);

    // Keep only last 1000 messages per room
    if (messages.length > 1000) {
      messages.splice(0, messages.length - 1000);
    }
  }

  getStreamUserId(stream) {
    for (const [userId, userStream] of this.userStreams) {
      if (userStream === stream) {
        return userId;
      }
    }
    return null;
  }

  generateId() {
    return `${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }
}

// Chat client example
class ChatClient {
  constructor(address = 'localhost:50051') {
    this.client = new chatProto.ChatService(
      address,
      grpc.credentials.createInsecure()
    );
    this.userId = `user_${Math.random().toString(36).substr(2, 9)}`;
    this.username = `ChatUser_${this.userId.substr(-4)}`;
    this.currentRoom = null;
    this.messageStream = null;
  }

  async createRoom(name, description = '') {
    return new Promise((resolve, reject) => {
      this.client.createRoom({
        name,
        description,
        created_by: this.userId
      }, (error, room) => {
        if (error) reject(error);
        else resolve(room);
      });
    });
  }

  async listRooms() {
    return new Promise((resolve, reject) => {
      this.client.listRooms({ limit: 10, offset: 0 }, (error, response) => {
        if (error) reject(error);
        else resolve(response.rooms);
      });
    });
  }

  async joinRoom(roomId) {
    if (this.messageStream) {
      this.messageStream.end();
    }

    this.currentRoom = roomId;
    this.messageStream = this.client.joinRoom({
      room_id: roomId,
      user_id: this.userId,
      username: this.username
    });

    this.messageStream.on('data', (message) => {
      this.displayMessage(message);
    });

    this.messageStream.on('error', (error) => {
      console.error('💥 Stream error:', error.message);
    });

    this.messageStream.on('end', () => {
      console.log('📡 Stream ended');
    });

    console.log(`🚪 Joined room: ${roomId}`);
  }

  async sendMessage(content) {
    if (!this.currentRoom) {
      console.log('❌ Not in any room');
      return;
    }

    return new Promise((resolve, reject) => {
      this.client.sendMessage({
        room_id: this.currentRoom,
        user_id: this.userId,
        content
      }, (error, response) => {
        if (error) reject(error);
        else resolve(response);
      });
    });
  }

  displayMessage(message) {
    const timestamp = new Date(parseInt(message.timestamp)).toLocaleTimeString();
    const messageType = this.getMessageTypeSymbol(message.type);
    
    if (message.type === 0) { // USER_MESSAGE
      console.log(`${messageType} [${timestamp}] ${message.username}: ${message.content}`);
    } else {
      console.log(`${messageType} [${timestamp}] ${message.content}`);
    }
  }

  getMessageTypeSymbol(type) {
    switch (type) {
      case 0: return '💬'; // USER_MESSAGE
      case 1: return '🤖'; // SYSTEM_MESSAGE
      case 2: return '👋'; // USER_JOINED
      case 3: return '👋'; // USER_LEFT
      default: return '❓';
    }
  }

  disconnect() {
    if (this.messageStream) {
      this.messageStream.end();
    }
    this.client.close();
  }
}

// Demo usage
async function runChatDemo() {
  const client = new ChatClient();

  try {
    // Create a room
    const room = await client.createRoom('Demo Room', 'A room for testing');
    console.log('🏠 Created room:', room);

    // List rooms
    const rooms = await client.listRooms();
    console.log('📋 Available rooms:', rooms);

    // Join the room
    await client.joinRoom(room.room_id);

    // Send some messages
    await new Promise(resolve => setTimeout(resolve, 1000));
    await client.sendMessage('Hello everyone!');
    await client.sendMessage('This is a gRPC chat demo');
    await client.sendMessage('Pretty cool, right?');

    // Keep running for a bit
    await new Promise(resolve => setTimeout(resolve, 5000));

  } catch (error) {
    console.error('❌ Chat demo error:', error);
  } finally {
    client.disconnect();
  }
}

module.exports = { ChatService, ChatClient, runChatDemo };
```

### Example 2: File Upload/Download Service

```javascript
// file-service.js - File transfer with streaming
const fs = require('fs');
const path = require('path');
const crypto = require('crypto');
const grpc = require('@grpc/grpc-js');

const fileProto = `
syntax = "proto3";
package file;

service FileService {
  rpc UploadFile(stream FileChunk) returns (UploadResponse);
  rpc DownloadFile(DownloadRequest) returns (stream FileChunk);
  rpc ListFiles(ListFilesRequest) returns (ListFilesResponse);
  rpc DeleteFile(DeleteFileRequest) returns (DeleteFileResponse);
  rpc GetFileInfo(GetFileInfoRequest) returns (FileInfo);
}

message FileChunk {
  string file_id = 1;
  string filename = 2;
  bytes data = 3;
  int64 chunk_number = 4;
  int64 total_chunks = 5;
  string checksum = 6;
}

message UploadResponse {
  string file_id = 1;
  bool success = 2;
  string message = 3;
  int64 total_size = 4;
  string checksum = 5;
}

message DownloadRequest {
  string file_id = 1;
}

message ListFilesRequest {
  string user_id = 1;
  int32 limit = 2;
  int32 offset = 3;
}

message ListFilesResponse {
  repeated FileInfo files = 1;
  int32 total_count = 2;
}

message DeleteFileRequest {
  string file_id = 1;
  string user_id = 2;
}

message DeleteFileResponse {
  bool success = 1;
  string message = 2;
}

message GetFileInfoRequest {
  string file_id = 1;
}

message FileInfo {
  string file_id = 1;
  string filename = 2;
  int64 size = 3;
  string content_type = 4;
  string checksum = 5;
  int64 uploaded_at = 6;
  string uploaded_by = 7;
}
`;

class FileService {
  constructor(uploadDir = './uploads') {
    this.uploadDir = uploadDir;
    this.files = new Map(); // file_id -> FileInfo
    this.uploadDir = uploadDir;
    
    // Ensure upload directory exists
    if (!fs.existsSync(uploadDir)) {
      fs.mkdirSync(uploadDir, { recursive: true });
    }
  }

  // Handle file upload with streaming
  uploadFile(call, callback) {
    let fileId = null;
    let filename = null;
    let totalSize = 0;
    let chunks = new Map();
    let totalChunks = 0;
    let expectedChecksum = null;
    let uploadedBy = null;

    call.on('data', (chunk) => {
      try {
        // Initialize file info from first chunk
        if (!fileId) {
          fileId = chunk.file_id || this.generateFileId();
          filename = chunk.filename;
          totalChunks = chunk.total_chunks;
          
          // Get user info from metadata
          const metadata = call.metadata.getMap();
          uploadedBy = metadata['user-id'] || 'anonymous';
          
          console.log(`📤 Starting upload: ${filename} (${totalChunks} chunks)`);
        }

        // Store chunk
        chunks.set(chunk.chunk_number, chunk.data);
        totalSize += chunk.data.length;

        if (chunk.checksum) {
          expectedChecksum = chunk.checksum;
        }

        console.log(`📦 Received chunk ${chunk.chunk_number}/${totalChunks} (${chunk.data.length} bytes)`);

      } catch (error) {
        console.error('Error processing chunk:', error);
        callback(error);
      }
    });

    call.on('end', async () => {
      try {
        // Verify all chunks received
        if (chunks.size !== totalChunks) {
          throw new Error(`Missing chunks: expected ${totalChunks}, got ${chunks.size}`);
        }

        // Reassemble file
        const fileBuffer = await this.assembleFile(chunks, totalChunks);
        
        // Verify checksum if provided
        if (expectedChecksum) {
          const actualChecksum = crypto.createHash('sha256').update(fileBuffer).digest('hex');
          if (actualChecksum !== expectedChecksum) {
            throw new Error('Checksum verification failed');
          }
        }

        // Save file to disk
        const filePath = path.join(this.uploadDir, `${fileId}_${filename}`);
        fs.writeFileSync(filePath, fileBuffer);

        // Store file info
        const fileInfo = {
          file_id: fileId,
          filename,
          size: totalSize,
          content_type: this.getContentType(filename),
          checksum: crypto.createHash('sha256').update(fileBuffer).digest('hex'),
          uploaded_at: Date.now(),
          uploaded_by: uploadedBy
        };

        this.files.set(fileId, fileInfo);

        console.log(`✅ Upload completed: ${filename} (${totalSize} bytes)`);

        callback(null, {
          file_id: fileId,
          success: true,
          message: 'Upload completed successfully',
          total_size: totalSize,
          checksum: fileInfo.checksum
        });

      } catch (error) {
        console.error('Upload error:', error);
        callback(error);
      }
    });

    call.on('error', (error) => {
      console.error('Upload stream error:', error);
      callback(error);
    });
  }

  // Handle file download with streaming
  downloadFile(call) {
    const fileId = call.request.file_id;
    
    if (!this.files.has(fileId)) {
      const error = new Error(`File ${fileId} not found`);
      error.code = grpc.status.NOT_FOUND;
      call.emit('error', error);
      return;
    }

    const fileInfo = this.files.get(fileId);
    const filePath = path.join(this.uploadDir, `${fileId}_${fileInfo.filename}`);

    if (!fs.existsSync(filePath)) {
      const error = new Error(`File data not found on disk`);
      error.code = grpc.status.INTERNAL;
      call.emit('error', error);
      return;
    }

    try {
      const fileBuffer = fs.readFileSync(filePath);
      const chunkSize = 64 * 1024; // 64KB chunks
      const totalChunks = Math.ceil(fileBuffer.length / chunkSize);

      console.log(`📥 Starting download: ${fileInfo.filename} (${totalChunks} chunks)`);

      let chunkNumber = 0;
      
      const sendNextChunk = () => {
        if (chunkNumber >= totalChunks) {
          console.log(`✅ Download completed: ${fileInfo.filename}`);
          call.end();
          return;
        }

        const start = chunkNumber * chunkSize;
        const end = Math.min(start + chunkSize, fileBuffer.length);
        const chunkData = fileBuffer.slice(start, end);

        const chunk = {
          file_id: fileId,
          filename: fileInfo.filename,
          data: chunkData,
          chunk_number: chunkNumber + 1,
          total_chunks: totalChunks,
          checksum: chunkNumber === totalChunks - 1 ? fileInfo.checksum : ''
        };

        call.write(chunk);
        chunkNumber++;

        // Send chunks with small delay to prevent overwhelming client
        setTimeout(sendNextChunk, 10);
      };

      sendNextChunk();

    } catch (error) {
      console.error('Download error:', error);
      call.emit('error', error);
    }
  }

  // List files
  listFiles(call, callback) {
    const { user_id, limit = 10, offset = 0 } = call.request;
    
    let files = Array.from(this.files.values());
    
    // Filter by user if specified
    if (user_id) {
      files = files.filter(file => file.uploaded_by === user_id);
    }

    const totalCount = files.length;
    const limitedFiles = files.slice(offset, offset + limit);

    callback(null, {
      files: limitedFiles,
      total_count: totalCount
    });
  }

  // Get file info
  getFileInfo(call, callback) {
    const fileId = call.request.file_id;
    
    if (!this.files.has(fileId)) {
      const error = new Error(`File ${fileId} not found`);
      error.code = grpc.status.NOT_FOUND;
      return callback(error);
    }

    callback(null, this.files.get(fileId));
  }

  // Delete file
  deleteFile(call, callback) {
    const { file_id, user_id } = call.request;
    
    if (!this.files.has(file_id)) {
      const error = new Error(`File ${file_id} not found`);
      error.code = grpc.status.NOT_FOUND;
      return callback(error);
    }

    const fileInfo = this.files.get(file_id);
    
    // Check permissions (in real app, implement proper authorization)
    if (user_id && fileInfo.uploaded_by !== user_id) {
      const error = new Error('Permission denied');
      error.code = grpc.status.PERMISSION_DENIED;
      return callback(error);
    }

    try {
      // Delete file from disk
      const filePath = path.join(this.uploadDir, `${file_id}_${fileInfo.filename}`);
      if (fs.existsSync(filePath)) {
        fs.unlinkSync(filePath);
      }

      // Remove from memory
      this.files.delete(file_id);

      console.log(`🗑️ Deleted file: ${fileInfo.filename}`);

      callback(null, {
        success: true,
        message: 'File deleted successfully'
      });

    } catch (error) {
      console.error('Delete error:', error);
      callback(error);
    }
  }

  // Helper methods
  async assembleFile(chunks, totalChunks) {
    const buffers = [];
    
    for (let i = 1; i <= totalChunks; i++) {
      if (!chunks.has(i)) {
        throw new Error(`Missing chunk ${i}`);
      }
      buffers.push(chunks.get(i));
    }
    
    return Buffer.concat(buffers);
  }

  generateFileId() {
    return `file_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  getContentType(filename) {
    const ext = path.extname(filename).toLowerCase();
    const mimeTypes = {
      '.txt': 'text/plain',
      '.pdf': 'application/pdf',
      '.jpg': 'image/jpeg',
      '.jpeg': 'image/jpeg',
      '.png': 'image/png',
      '.gif': 'image/gif',
      '.mp4': 'video/mp4',
      '.mp3': 'audio/mpeg',
      '.zip': 'application/zip',
      '.json': 'application/json'
    };
    return mimeTypes[ext] || 'application/octet-stream';
  }
}

// File client for uploading and downloading
class FileClient {
  constructor(address = 'localhost:50051') {
    this.client = new fileProto.FileService(
      address,
      grpc.credentials.createInsecure()
    );
    this.userId = `user_${Math.random().toString(36).substr(2, 9)}`;
  }

  async uploadFile(filePath) {
    return new Promise((resolve, reject) => {
      if (!fs.existsSync(filePath)) {
        return reject(new Error(`File not found: ${filePath}`));
      }

      const filename = path.basename(filePath);
      const fileBuffer = fs.readFileSync(filePath);
      const fileId = this.generateFileId();
      const chunkSize = 64 * 1024; // 64KB chunks
      const totalChunks = Math.ceil(fileBuffer.length / chunkSize);
      const checksum = crypto.createHash('sha256').update(fileBuffer).digest('hex');

      console.log(`📤 Uploading ${filename} (${fileBuffer.length} bytes, ${totalChunks} chunks)`);

      // Create metadata
      const metadata = new grpc.Metadata();
      metadata.add('user-id', this.userId);

      const call = this.client.uploadFile(metadata, (error, response) => {
        if (error) {
          reject(error);
        } else {
          console.log('✅ Upload response:', response);
          resolve(response);
        }
      });

      // Send file in chunks
      let chunkNumber = 0;
      
      const sendNextChunk = () => {
        if (chunkNumber >= totalChunks) {
          call.end();
          return;
        }

        const start = chunkNumber * chunkSize;
        const end = Math.min(start + chunkSize, fileBuffer.length);
        const chunkData = fileBuffer.slice(start, end);

        const chunk = {
          file_id: fileId,
          filename,
          data: chunkData,
          chunk_number: chunkNumber + 1,
          total_chunks: totalChunks,
          checksum: chunkNumber === totalChunks - 1 ? checksum : ''
        };

        console.log(`📦 Sending chunk ${chunkNumber + 1}/${totalChunks}`);
        call.write(chunk);
        chunkNumber++;

        // Send next chunk with small delay
        setTimeout(sendNextChunk, 10);
      };

      sendNextChunk();
    });
  }

  async downloadFile(fileId, outputPath) {
    return new Promise((resolve, reject) => {
      const call = this.client.downloadFile({ file_id: fileId });
      const chunks = new Map();
      let totalChunks = 0;
      let filename = '';
      let expectedChecksum = '';

      call.on('data', (chunk) => {
        if (!filename) {
          filename = chunk.filename;
          totalChunks = chunk.total_chunks;
          console.log(`📥 Downloading ${filename} (${totalChunks} chunks)`);
        }

        chunks.set(chunk.chunk_number, chunk.data);
        
        if (chunk.checksum) {
          expectedChecksum = chunk.checksum;
        }

        console.log(`📦 Received chunk ${chunk.chunk_number}/${totalChunks}`);
      });

      call.on('end', () => {
        try {
          // Verify all chunks received
          if (chunks.size !== totalChunks) {
            throw new Error(`Missing chunks: expected ${totalChunks}, got ${chunks.size}`);
          }

          // Reassemble file
          const buffers = [];
          for (let i = 1; i <= totalChunks; i++) {
            buffers.push(chunks.get(i));
          }
          const fileBuffer = Buffer.concat(buffers);

          // Verify checksum
          if (expectedChecksum) {
            const actualChecksum = crypto.createHash('sha256').update(fileBuffer).digest('hex');
            if (actualChecksum !== expectedChecksum) {
              throw new Error('Checksum verification failed');
            }
          }

          // Save file
          const finalPath = outputPath || `./${filename}`;
          fs.writeFileSync(finalPath, fileBuffer);

          console.log(`✅ Download completed: ${finalPath} (${fileBuffer.length} bytes)`);
          resolve({ filename, size: fileBuffer.length, path: finalPath });

        } catch (error) {
          reject(error);
        }
      });

      call.on('error', (error) => {
        reject(error);
      });
    });
  }

  async listFiles() {
    return new Promise((resolve, reject) => {
      this.client.listFiles({
        user_id: this.userId,
        limit: 20,
        offset: 0
      }, (error, response) => {
        if (error) reject(error);
        else resolve(response);
      });
    });
  }

  async getFileInfo(fileId) {
    return new Promise((resolve, reject) => {
      this.client.getFileInfo({ file_id: fileId }, (error, fileInfo) => {
        if (error) reject(error);
        else resolve(fileInfo);
      });
    });
  }

  async deleteFile(fileId) {
    return new Promise((resolve, reject) => {
      this.client.deleteFile({
        file_id: fileId,
        user_id: this.userId
      }, (error, response) => {
        if (error) reject(error);
        else resolve(response);
      });
    });
  }

  generateFileId() {
    return `file_${Date.now()}_${Math.random().toString(36).substr(2, 9)}`;
  }

  close() {
    this.client.close();
  }
}

// Demo file operations
async function runFileDemo() {
  const client = new FileClient();

  try {
    // Create a test file
    const testFilePath = './test-file.txt';
    const testContent = 'This is a test file for gRPC file transfer demo!\n'.repeat(1000);
    fs.writeFileSync(testFilePath, testContent);

    console.log('📁 Created test file:', testFilePath);

    // Upload file
    const uploadResponse = await client.uploadFile(testFilePath);
    console.log('📤 Upload successful:', uploadResponse);

    // List files
    const fileList = await client.listFiles();
    console.log('📋 Files:', fileList);

    // Get file info
    const fileInfo = await client.getFileInfo(uploadResponse.file_id);
    console.log('ℹ️ File info:', fileInfo);

    // Download file
    const downloadPath = './downloaded-file.txt';
    await client.downloadFile(uploadResponse.file_id, downloadPath);
    console.log('📥 Downloaded to:', downloadPath);

    // Verify files are identical
    const originalContent = fs.readFileSync(testFilePath);
    const downloadedContent = fs.readFileSync(downloadPath);
    
    if (originalContent.equals(downloadedContent)) {
      console.log('✅ File integrity verified!');
    } else {
      console.log('❌ File integrity check failed!');
    }

    // Clean up
    fs.unlinkSync(testFilePath);
    fs.unlinkSync(downloadPath);

  } catch (error) {
    console.error('❌ File demo error:', error);
  } finally {
    client.close();
  }
}

module.exports = { FileService, FileClient, runFileDemo };
```

---

## 10. Troubleshooting {#troubleshooting}

### Common Issues and Solutions

#### 1. Module and Path Issues

```javascript
// troubleshooting.js - Common solutions
const path = require('path');
const fs = require('fs');

// Issue: Proto file not found
function loadProtoSafely(protoPath) {
  const absolutePath = path.resolve(protoPath);
  
  if (!fs.existsSync(absolutePath)) {
    throw new Error(`Proto file not found: ${absolutePath}`);
  }

  console.log(`📄 Loading proto from: ${absolutePath}`);
  
  return protoLoader.loadSync(absolutePath, {
    keepCase: true,
    longs: String,
    enums: String,
    defaults: true,
    oneofs: true,
  });
}

// Issue: Port already in use
async function findAvailablePort(startPort = 50051) {
  const net = require('net');
  
  return new Promise((resolve, reject) => {
    const server = net.createServer();
    
    server.listen(startPort, () => {
      const port = server.address().port;
      server.close(() => resolve(port));
    });
    
    server.on('error', (err) => {
      if (err.code === 'EADDRINUSE') {
        resolve(findAvailablePort(startPort + 1));
      } else {
        reject(err);
      }
    });
  });
}

// Issue: Connection refused
class RobustClient {
  constructor(address, maxRetries = 3) {
    this.address = address;
    this.maxRetries = maxRetries;
    this.client = null;
  }

  async connect() {
    for (let attempt = 1; attempt <= this.maxRetries; attempt++) {
      try {
        this.client = new userProto.UserService(
          this.address,
          grpc.credentials.createInsecure()
        );

        // Test connection
        await this.testConnection();
        console.log(`✅ Connected to ${this.address}`);
        return;

      } catch (error) {
        console.log(`❌ Connection attempt ${attempt} failed: ${error.message}`);
        
        if (attempt === this.maxRetries) {
          throw new Error(`Failed to connect after ${this.maxRetries} attempts`);
        }
        
        // Wait before retry
        await new Promise(resolve => setTimeout(resolve, 1000 * attempt));
      }
    }
  }

  async testConnection() {
    return new Promise((resolve, reject) => {
      // Try a simple call with timeout
      const deadline = new Date();
      deadline.setSeconds(deadline.getSeconds() + 5);

      this.client.getUser(
        { user_id: 1 },
        { deadline },
        (error, response) => {
          // We expect this to fail (user probably doesn't exist)
          // but if server is running, we'll get a proper gRPC error
          if (error && error.code !== grpc.status.UNAVAILABLE) {
            resolve(); // Server is responding
          } else if (!error) {
            resolve(); // Unexpected success
          } else {
            reject(error); // Server unavailable
          }
        }
      );
    });
  }
}
```

#### 2. Debug Logging and Monitoring

```javascript
// debug-utils.js - Debugging helpers
class DebugInterceptor {
  constructor(serviceName) {
    this.serviceName = serviceName;
  }

  intercept(methodDefinition, handler) {
    const methodName = methodDefinition ? methodDefinition.path : 'unknown';
    
    return (call, callback) => {
      const startTime = Date.now();
      const requestId = this.generateRequestId();
      
      console.log(`🔍 [${this.serviceName}] ${methodName} started - ID: ${requestId}`);
      
      // Log request metadata
      if (call.metadata) {
        console.log(`📋 Request metadata:`, call.metadata.getMap());
      }

      // Wrap callback to log response
      const wrappedCallback = callback ? (error, response) => {
        const duration = Date.now() - startTime;
        
        if (error) {
          console.log(`❌ [${this.serviceName}] ${methodName} failed in ${duration}ms - ID: ${requestId}`);
          console.log(`   Error: ${error.message} (code: ${error.code})`);
        } else {
          console.log(`✅ [${this.serviceName}] ${methodName} completed in ${duration}ms - ID: ${requestId}`);
        }
        
        callback(error, response);
      } : undefined;

      // Call original handler
      return handler(call, wrappedCallback);
    };
  }

  generateRequestId() {
    return Math.random().toString(36).substr(2, 9);
  }
}

// Performance monitoring
class PerformanceMonitor {
  constructor() {
    this.metrics = {
      requests: 0,
      errors: 0,
      totalLatency: 0,
      maxLatency: 0,
      minLatency: Infinity
    };
    this.startTime = Date.now();
  }

  recordRequest(latency, error = null) {
    this.metrics.requests++;
    this.metrics.totalLatency += latency;
    this.metrics.maxLatency = Math.max(this.metrics.maxLatency, latency);
    this.metrics.minLatency = Math.min(this.metrics.minLatency, latency);
    
    if (error) {
      this.metrics.errors++;
    }
  }

  getStats() {
    const uptime = Date.now() - this.startTime;
    const avgLatency = this.metrics.requests > 0 
      ? this.metrics.totalLatency / this.metrics.requests 
      : 0;
    const requestsPerSecond = (this.metrics.requests / uptime) * 1000;
    const errorRate = this.metrics.requests > 0 
      ? (this.metrics.errors / this.metrics.requests) * 100 
      : 0;

    return {
      uptime: uptime,
      totalRequests: this.metrics.requests,
      totalErrors: this.metrics.errors,
      errorRate: `${errorRate.toFixed(2)}%`,
      requestsPerSecond: requestsPerSecond.toFixed(2),
      averageLatency: `${avgLatency.toFixed(2)}ms`,
      minLatency: `${this.metrics.minLatency}ms`,
      maxLatency: `${this.metrics.maxLatency}ms`
    };
  }

  printStats() {
    console.log('\n📊 Performance Metrics:');
    const stats = this.getStats();
    Object.entries(stats).forEach(([key, value]) => {
      console.log(`   ${key}: ${value}`);
    });
  }
}

// Health check service
const healthProto = `
syntax = "proto3";
package health;

service HealthCheck {
  rpc Check(HealthCheckRequest) returns (HealthCheckResponse);
  rpc Watch(HealthCheckRequest) returns (stream HealthCheckResponse);
}

message HealthCheckRequest {
  string service = 1;
}

message HealthCheckResponse {
  enum ServingStatus {
    UNKNOWN = 0;
    SERVING = 1;
    NOT_SERVING = 2;
    SERVICE_UNKNOWN = 3;
  }
  ServingStatus status = 1;
}
`;

class HealthCheckService {
  constructor() {
    this.serviceStatuses = new Map();
    this.serviceStatuses.set('', 'SERVING'); // Overall health
  }

  check(call, callback) {
    const service = call.request.service || '';
    const status = this.serviceStatuses.get(service) || 'SERVICE_UNKNOWN';
    
    callback(null, { status: this.getStatusNumber(status) });
  }

  watch(call) {
    const service = call.request.service || '';
    
    // Send current status
    const status = this.serviceStatuses.get(service) || 'SERVICE_UNKNOWN';
    call.write({ status: this.getStatusNumber(status) });

    // In a real implementation, you'd monitor service health
    // and send updates when status changes
    
    // Keep connection alive
    const interval = setInterval(() => {
      if (!call.cancelled) {
        call.write({ status: this.getStatusNumber('SERVING') });
      } else {
        clearInterval(interval);
      }
    }, 5000);

    call.on('cancelled', () => {
      clearInterval(interval);
    });
  }

  getStatusNumber(status) {
    const statusMap = {
      'UNKNOWN': 0,
      'SERVING': 1,
      'NOT_SERVING': 2,
      'SERVICE_UNKNOWN': 3
    };
    return statusMap[status] || 0;
  }

  setServiceStatus(service, status) {
    this.serviceStatuses.set(service, status);
  }
}
```

#### 3. Production Deployment Checklist

```javascript
// production-config.js - Production best practices
class ProductionServer {
  constructor() {
    this.server = null;
    this.monitor = new PerformanceMonitor();
    this.healthCheck = new HealthCheckService();
  }

  async start() {
    try {
      // Load configuration from environment
      const config = this.loadConfiguration();
      
      // Create server with production settings
      this.server = new grpc.Server({
        'grpc.keepalive_time_ms': 30000,
        'grpc.keepalive_timeout_ms': 5000,
        'grpc.keepalive_permit_without_calls': true,
        'grpc.http2.max_pings_without_data': 0,
        'grpc.http2.min_time_between_pings_ms': 10000,
        'grpc.http2.min_ping_interval_without_data_ms': 300000,
        'grpc.max_receive_message_length': 4 * 1024 * 1024, // 4MB
        'grpc.max_send_message_length': 4 * 1024 * 1024     // 4MB
      });

      // Add services with monitoring
      this.addServicesWithMonitoring();

      // Add health check service
      this.server.addService(healthProto.HealthCheck.service, {
        check: this.healthCheck.check.bind(this.healthCheck),
        watch: this.healthCheck.watch.bind(this.healthCheck)
      });

      // Setup SSL if configured
      const credentials = config.ssl.enabled 
        ? this.createSSLCredentials(config.ssl)
        : grpc.ServerCredentials.createInsecure();

      // Start server
      await this.bindServer(config.host, config.port, credentials);
      
      // Setup graceful shutdown
      this.setupGracefulShutdown();
      
      // Start monitoring
      this.startMonitoring();

      console.log(`🚀 Production server started on ${config.host}:${config.port}`);

    } catch (error) {
      console.error('❌ Failed to start production server:', error);
      process.exit(1);
    }
  }

  loadConfiguration() {
    return {
      host: process.env.GRPC_HOST || '0.0.0.0',
      port: parseInt(process.env.GRPC_PORT) || 50051,
      ssl: {
        enabled: process.env.SSL_ENABLED === 'true',
        certPath: process.env.SSL_CERT_PATH,
        keyPath: process.env.SSL_KEY_PATH,
        caPath: process.env.SSL_CA_PATH
      },
      monitoring: {
        enabled: process.env.MONITORING_ENABLED !== 'false',
        interval: parseInt(process.env.MONITORING_INTERVAL) || 60000
      }
    };
  }

  addServicesWithMonitoring() {
    const userService = new UserService();
    const debugInterceptor = new DebugInterceptor('UserService');

    // Wrap service methods with monitoring
    const monitoredService = {};
    
    ['getUser', 'createUser', 'updateUser', 'deleteUser'].forEach(methodName => {
      const originalMethod = userService[methodName].bind(userService);
      
      monitoredService[methodName] = (call, callback) => {
        const startTime = Date.now();
        
        const wrappedCallback = (error, response) => {
          const latency = Date.now() - startTime;
          this.monitor.recordRequest(latency, error);
          
          if (callback) {
            callback(error, response);
          }
        };

        // Apply debug interceptor
        const interceptedMethod = debugInterceptor.intercept(null, originalMethod);
        return interceptedMethod(call, wrappedCallback);
      };
    });

    this.server.addService(userProto.UserService.service, monitoredService);
  }

  createSSLCredentials(sslConfig) {
    const fs = require('fs');
    
    const serverKey = fs.readFileSync(sslConfig.keyPath);
    const serverCert = fs.readFileSync(sslConfig.certPath);
    const caCert = sslConfig.caPath ? fs.readFileSync(sslConfig.caPath) : null;

    return grpc.ServerCredentials.createSsl(
      caCert,
      [{ private_key: serverKey, cert_chain: serverCert }],
      false // Don't require client certificates
    );
  }

  bindServer(host, port, credentials) {
    return new Promise((resolve, reject) => {
      this.server.bindAsync(`${host}:${port}`, credentials, (error, boundPort) => {
        if (error) {
          reject(error);
        } else {
          this.server.start();
          resolve(boundPort);
        }
      });
    });
  }

  setupGracefulShutdown() {
    const shutdownHandler = () => {
      console.log('\n🛑 Shutting down gracefully...');
      
      this.healthCheck.setServiceStatus('', 'NOT_SERVING');
      
      this.server.tryShutdown((error) => {
        if (error) {
          console.error('❌ Error during shutdown:', error);
          process.exit(1);
        } else {
          console.log('✅ Server shut down gracefully');
          process.exit(0);
        }
      });

      // Force shutdown after 10 seconds
      setTimeout(() => {
        console.log('⚠️ Force shutting down...');
        process.exit(1);
      }, 10000);
    };

    process.on('SIGINT', shutdownHandler);
    process.on('SIGTERM', shutdownHandler);
  }

  startMonitoring() {
    const config = this.loadConfiguration();
    
    if (config.monitoring.enabled) {
      setInterval(() => {
        this.monitor.printStats();
      }, config.monitoring.interval);
    }
  }
}

// Environment variables for production
const productionEnvExample = `
# Production environment variables
GRPC_HOST=0.0.0.0
GRPC_PORT=50051
SSL_ENABLED=true
SSL_CERT_PATH=/path/to/server.crt
SSL_KEY_PATH=/path/to/server.key
SSL_CA_PATH=/path/to/ca.crt
MONITORING_ENABLED=true
MONITORING_INTERVAL=60000
NODE_ENV=production
`;

module.exports = {
  ProductionServer,
  DebugInterceptor,
  PerformanceMonitor,
  HealthCheckService,
  productionEnvExample
};
```

---

## Conclusion

This comprehensive guide covers gRPC with JavaScript/Node.js from basics to production-ready implementations. Key takeaways:

### 🎯 **Getting Started**
1. **Start Simple**: Begin with unary RPCs and basic services
2. **Understand Proto**: Master Protocol Buffers syntax and data types
3. **Practice Streaming**: Experiment with all four service types

### 🔧 **Development Best Practices**
1. **Service Design**: Keep services focused and use consistent naming
2. **Error Handling**: Implement comprehensive error handling with proper status codes
3. **Testing**: Write unit tests, integration tests, and load tests
4. **Monitoring**: Add logging, metrics, and health checks

### 🚀 **Production Ready**
1. **Security**: Use SSL/TLS and implement authentication
2. **Performance**: Optimize with connection pooling and caching
3. **Reliability**: Add retry logic, timeouts, and graceful shutdown
4. **Monitoring**: Track performance metrics and service health

### 📚 **Next Steps**
- Practice building different types of services (chat, file transfer, etc.)
- Explore gRPC-Web for browser integration
- Learn about service mesh (Istio, Envoy) for microservices
- Study distributed tracing and observability

### 🔗 **Useful Resources**
- [Official gRPC Documentation](https://grpc.io/docs/)
- [Protocol Buffers Documentation](https://developers.google.com/protocol-buffers)
- [gRPC Node.js Examples](https://github.com/grpc/grpc-node/tree/master/examples)
- [gRPC Best Practices](https://grpc.io/docs/guides/best-practices/)

The examples in this guide are production-ready starting points that you can extend and customize for your specific needs. Happy coding with gRPC! 🚀