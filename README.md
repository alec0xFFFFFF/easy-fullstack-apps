# Full-Stack App Recipe: Epic Stack + iOS App

> A comprehensive guide for building full-stack applications using Epic Stack (Remix) + iOS native app with shared authentication, analytics, and API endpoints.

## 🏗️ Architecture Overview

This recipe creates a monorepo with:
- **Epic Stack Web App** - Production-ready Remix application with authentication, database, and API endpoints
- **iOS Native App** - SwiftUI app that consumes the web app's API endpoints  
- **Shared Services** - PostHog analytics, Epic Stack authentication, unified branding
- **Railway Deployment** - PostgreSQL database with seamless deployment

## 📋 Prerequisites

### Development Tools
- **Node.js** (v18+ LTS)
- **npm** or **yarn**
- **Xcode** (latest version)
- **Git**

### External Services
- **PostHog** account (for analytics)
- **Railway** account (for deployment & PostgreSQL)
- **Apple Developer** account (for iOS deployment)
- **Domain** for production hosting

## 🚀 Step 1: Epic Stack Setup

### 1.1 Initialize Epic Stack Project

```bash
# Create new Epic Stack project
npx epicli new my-fullstack-app

# Navigate to project
cd my-fullstack-app

# The Epic Stack CLI will guide you through:
# - Project name selection
# - Database setup (choose PostgreSQL)
# - Authentication setup
# - Deployment configuration
```

### 1.2 Add iOS Project to Monorepo

```bash
# Convert to monorepo structure
mkdir -p apps/ios-project
mv * apps/web/ 2>/dev/null || true
mv .* apps/web/ 2>/dev/null || true

# Move essential files back to root
mv apps/web/.gitignore .
mv apps/web/package.json .
mv apps/web/README.md .
```

### 1.3 Update Root Package.json

```json
{
  "name": "my-fullstack-app",
  "private": true,
  "workspaces": [
    "apps/web"
  ],
  "scripts": {
    "dev": "npm run dev --workspace=apps/web",
    "build": "npm run build --workspace=apps/web",
    "test": "npm run test --workspace=apps/web",
    "ios:build": "cd apps/ios-project && xcodebuild -workspace *.xcworkspace -scheme * -configuration Release"
  }
}
```

## 🗄️ Step 2: Database & API Setup

### 2.1 Configure PostgreSQL on Railway

1. **Create Railway Project**:
   ```bash
   # Install Railway CLI
   npm install -g @railway/cli
   
   # Login to Railway
   railway login
   
   # Create new project
   railway create my-fullstack-app
   
   # Add PostgreSQL database
   railway add postgresql
   ```

2. **Update Database Configuration**:
   ```bash
   # In apps/web/.env
   DATABASE_URL="postgresql://user:password@host:port/database"
   ```

### 2.2 Customize Database Schema

Update `apps/web/prisma/schema.prisma`:

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Keep Epic Stack's User model and add your app-specific models
model User {
  id    String @id @default(cuid())
  email String @unique

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  image    UserImage?
  password Password?
  notes    Note[]
  roles    Role[]
  session  Session[]
  
  // Add your app-specific fields
  items    Item[]
  phone    String?  @unique

  @@map("users")
}

// Add your app-specific models
model Item {
  id          String   @id @default(cuid())
  name        String
  description String?
  imageUrl    String?
  category    String?
  quantity    Int      @default(1)
  userId      String
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("items")
}

// Keep other Epic Stack models...
model Note {
  id      String @id @default(cuid())
  title   String
  content String

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  owner   User   @relation(fields: [ownerId], references: [id], onDelete: Cascade, onUpdate: Cascade)
  ownerId String

  images NoteImage[]

  @@map("notes")
}

// ... rest of Epic Stack models
```

### 2.3 Create API Routes for iOS

Create `apps/web/app/routes/api+/items.tsx`:

```typescript
import { json, type ActionFunctionArgs, type LoaderFunctionArgs } from "@remix-run/node";
import { requireUserId } from "~/utils/auth.server";
import { prisma } from "~/utils/db.server";

export async function loader({ request }: LoaderFunctionArgs) {
  const userId = await requireUserId(request);
  
  const items = await prisma.item.findMany({
    where: { userId },
    orderBy: { createdAt: "desc" },
  });
  
  return json({ items });
}

export async function action({ request }: ActionFunctionArgs) {
  const userId = await requireUserId(request);
  const formData = await request.formData();
  const method = request.method;
  
  switch (method) {
    case "POST": {
      const name = formData.get("name") as string;
      const description = formData.get("description") as string;
      const imageUrl = formData.get("imageUrl") as string;
      const category = formData.get("category") as string;
      const quantity = parseInt(formData.get("quantity") as string) || 1;
      
      const item = await prisma.item.create({
        data: {
          name,
          description,
          imageUrl,
          category,
          quantity,
          userId,
        },
      });
      
      return json({ item });
    }
    
    case "PUT": {
      const id = formData.get("id") as string;
      const name = formData.get("name") as string;
      const description = formData.get("description") as string;
      const quantity = parseInt(formData.get("quantity") as string) || 1;
      
      const item = await prisma.item.update({
        where: { id, userId },
        data: { name, description, quantity },
      });
      
      return json({ item });
    }
    
    case "DELETE": {
      const id = formData.get("id") as string;
      
      await prisma.item.delete({
        where: { id, userId },
      });
      
      return json({ success: true });
    }
    
    default:
      return json({ error: "Method not allowed" }, { status: 405 });
  }
}
```

## 🔐 Step 2.4: Stytch SMS Authentication Setup

### 2.4.1 Configure Stytch

1. **Create Stytch Account**:
   - Go to [Stytch Dashboard](https://stytch.com/dashboard)
   - Create a new project
   - Get your Project ID and Secret

2. **Install Stytch SDK**:
   ```bash
   cd apps/web
   npm install stytch
   ```

3. **Environment Variables**:
   ```bash
   # Add to apps/web/.env
   STYTCH_PROJECT_ID="project-test-..."
   STYTCH_SECRET="secret-test-..."
   ```

### 2.4.2 Database Schema Updates

Update `apps/web/prisma/schema.prisma` to support Stytch authentication:

```prisma
// User model with Stytch integration
model User {
  id    String @id @default(cuid())
  email String @unique

  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt

  image    UserImage?
  notes    Note[]
  roles    Role[]
  session  Session[]
  
  // Stytch and app-specific fields
  items       Item[]
  phone       String?  @unique
  connections Connection[]

  @@map("users")
}

// Connection model for provider authentication
model Connection {
  id           String @id @default(cuid())
  providerName String
  providerId   String
  userId       String
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@unique([providerName, providerId])
  @@map("connections")
}

// Session model for secure authentication
model Session {
  id             String   @id @default(cuid())
  expirationDate DateTime
  userId         String
  createdAt      DateTime @default(now())
  updatedAt      DateTime @updatedAt

  user User @relation(fields: [userId], references: [id], onDelete: Cascade)

  @@index([userId])
  @@map("sessions")
}

// Your app models
model Item {
  id          String   @id @default(cuid())
  name        String
  description String?
  imageUrl    String?
  category    String?
  quantity    Int      @default(1)
  userId      String
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  user User @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@map("items")
}
```

### 2.4.3 Stytch Service Implementation

Create `apps/web/app/utils/stytch.server.ts`:

```typescript
import { Client } from 'stytch';

const stytch = new Client({
  project_id: process.env.STYTCH_PROJECT_ID!,
  secret: process.env.STYTCH_SECRET!,
});

export { stytch };
```

### 2.4.4 Authentication Route for SMS

Create `apps/web/app/routes/api+/login.tsx`:

```typescript
import { json, type ActionFunctionArgs } from "@remix-run/node";
import { stytch } from "~/utils/stytch.server";
import { prisma } from "~/utils/db.server";
import { createSession } from "~/utils/session.server";

export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const flow = formData.get("flow") as string;
  
  try {
    switch (flow) {
      case "send": {
        const phoneNumber = formData.get("phoneNumber") as string;
        
        // Send SMS with Stytch
        const response = await stytch.otps.sms.send({
          phone_number: phoneNumber,
          expiration_minutes: 5,
        });
        
        return json({ 
          success: true, 
          methodId: response.method_id 
        });
      }
      
      case "verify": {
        const methodId = formData.get("methodId") as string;
        const code = formData.get("code") as string;
        
        // Verify SMS code with Stytch
        const response = await stytch.otps.authenticate({
          method_id: methodId,
          code: code,
        });
        
        if (response.status_code === 200) {
          const phoneNumber = response.phone_number;
          
          // Find or create user and connection
          let connection = await prisma.connection.findUnique({
            where: {
              providerName_providerId: {
                providerName: "stytch",
                providerId: phoneNumber,
              },
            },
            include: { user: true },
          });
          
          if (!connection) {
            // Create new user and connection
            const user = await prisma.user.create({
              data: {
                phone: phoneNumber,
                email: `${phoneNumber}@temp.com`, // Temporary email
                connections: {
                  create: {
                    providerName: "stytch",
                    providerId: phoneNumber,
                  },
                },
              },
            });
            
            connection = await prisma.connection.findUnique({
              where: {
                providerName_providerId: {
                  providerName: "stytch",
                  providerId: phoneNumber,
                },
              },
              include: { user: true },
            });
          }
          
          if (connection) {
            // Create session
            const session = await createSession(connection.user.id);
            
            return json({ 
              success: true, 
              user: {
                id: connection.user.id,
                phone: connection.user.phone,
                email: connection.user.email,
              },
              sessionId: session.id,
            }, {
              headers: {
                "Set-Cookie": await commitSession(session),
              },
            });
          }
        }
        
        return json({ error: "Invalid verification code" }, { status: 400 });
      }
      
      default:
        return json({ error: "Invalid flow" }, { status: 400 });
    }
  } catch (error) {
    console.error('Authentication error:', error);
    return json({ error: "Authentication failed" }, { status: 500 });
  }
}
```

### 2.4.5 Secure API Routes

Create `apps/web/app/routes/api+/items.tsx` with proper authentication:

```typescript
import { json, type ActionFunctionArgs, type LoaderFunctionArgs } from "@remix-run/node";
import { requireUserId } from "~/utils/auth.server";
import { prisma } from "~/utils/db.server";

export async function loader({ request }: LoaderFunctionArgs) {
  const userId = await requireUserId(request);
  
  const items = await prisma.item.findMany({
    where: { userId },
    orderBy: { createdAt: "desc" },
  });
  
  return json({ items });
}

export async function action({ request }: ActionFunctionArgs) {
  const userId = await requireUserId(request);
  const formData = await request.formData();
  const method = request.method;
  
  switch (method) {
    case "POST": {
      const name = formData.get("name") as string;
      const description = formData.get("description") as string;
      const imageUrl = formData.get("imageUrl") as string;
      const category = formData.get("category") as string;
      const quantity = parseInt(formData.get("quantity") as string) || 1;
      
      const item = await prisma.item.create({
        data: {
          name,
          description,
          imageUrl,
          category,
          quantity,
          userId,
        },
      });
      
      return json({ item });
    }
    
    case "PUT": {
      const id = formData.get("id") as string;
      const name = formData.get("name") as string;
      const description = formData.get("description") as string;
      const quantity = parseInt(formData.get("quantity") as string) || 1;
      
      const item = await prisma.item.update({
        where: { id, userId },
        data: { name, description, quantity },
      });
      
      return json({ item });
    }
    
    case "DELETE": {
      const id = formData.get("id") as string;
      
      await prisma.item.delete({
        where: { id, userId },
      });
      
      return json({ success: true });
    }
    
    default:
      return json({ error: "Method not allowed" }, { status: 405 });
  }
}
```

### 2.4.6 Session Management

Update `apps/web/app/utils/session.server.ts`:

```typescript
import { createCookieSessionStorage } from "@remix-run/node";
import { prisma } from "./db.server";

const sessionStorage = createCookieSessionStorage({
  cookie: {
    name: "__session",
    httpOnly: true,
    secure: process.env.NODE_ENV === "production",
    secrets: [process.env.SESSION_SECRET!],
    sameSite: "lax",
    path: "/",
    maxAge: 60 * 60 * 24 * 30, // 30 days
  },
});

export async function createSession(userId: string) {
  const session = await prisma.session.create({
    data: {
      userId,
      expirationDate: new Date(Date.now() + 1000 * 60 * 60 * 24 * 30), // 30 days
    },
  });
  
  const cookieSession = await sessionStorage.getSession();
  cookieSession.set("sessionId", session.id);
  
  return cookieSession;
}

export async function requireUserId(request: Request) {
  const cookie = request.headers.get("Cookie");
  const session = await sessionStorage.getSession(cookie);
  const sessionId = session.get("sessionId");
  
  if (!sessionId) {
    throw new Response("Unauthorized", { status: 401 });
  }
  
  const dbSession = await prisma.session.findUnique({
    where: { id: sessionId },
    include: { user: true },
  });
  
  if (!dbSession || dbSession.expirationDate < new Date()) {
    throw new Response("Unauthorized", { status: 401 });
  }
  
  return dbSession.userId;
}

export const { getSession, commitSession, destroySession } = sessionStorage;
```
```

## 📱 Step 3: iOS App Setup

### 3.1 Create iOS Project

```bash
cd apps/ios-project
```

Use Xcode to create a new iOS project:
1. Open Xcode
2. File → New → Project
3. iOS → App
4. Product Name: Your app name
5. Bundle Identifier: `com.yourcompany.yourapp`
6. Language: Swift, Interface: SwiftUI
7. Save in `apps/ios-project/`

### 3.2 iOS Project Structure

```
apps/ios-project/
├── YourApp.xcodeproj/
├── YourApp.xcworkspace/
├── YourApp/
│   ├── YourAppApp.swift
│   ├── Models/
│   │   ├── User.swift
│   │   └── Item.swift
│   ├── Services/
│   │   ├── APIClient.swift
│   │   └── AnalyticsManager.swift
│   ├── Views/
│   │   ├── ContentView.swift
│   │   ├── LoginView.swift
│   │   └── ItemsView.swift
│   └── Utils/
└── YourAppPackage/
    └── Package.swift
```

### 3.3 iOS Package Dependencies

Create `apps/ios-project/YourAppPackage/Package.swift`:

```swift
// swift-tools-version: 5.9

import PackageDescription

let package = Package(
    name: "YourAppPackage",
    platforms: [
        .iOS(.v16)
    ],
    products: [
        .library(
            name: "YourAppFeature",
            targets: ["YourAppFeature"]
        ),
    ],
    dependencies: [
        .package(url: "https://github.com/PostHog/posthog-ios", from: "3.0.0"),
    ],
    targets: [
        .target(
            name: "YourAppFeature",
            dependencies: [
                .product(name: "PostHog", package: "posthog-ios"),
            ]
        ),
    ]
)
```

### 3.4 iOS API Client with Stytch Authentication

Create `apps/ios-project/YourApp/Services/APIClient.swift`:

```swift
import Foundation

class APIClient: ObservableObject {
    static let shared = APIClient()
    
    private let baseURL = "https://yourdomain.com" // Your Railway app URL
    private let session = URLSession.shared
    
    @Published var isAuthenticated = false
    @Published var currentUser: User?
    
    private init() {
        loadStoredSession()
    }
    
    // MARK: - Session Management
    
    private func loadStoredSession() {
        if let sessionData = UserDefaults.standard.data(forKey: "user_session"),
           let user = try? JSONDecoder().decode(User.self, from: sessionData) {
            self.currentUser = user
            self.isAuthenticated = true
        }
    }
    
    private func saveSession(_ user: User) {
        if let encoded = try? JSONEncoder().encode(user) {
            UserDefaults.standard.set(encoded, forKey: "user_session")
        }
        self.currentUser = user
        self.isAuthenticated = true
    }
    
    private func clearSession() {
        UserDefaults.standard.removeObject(forKey: "user_session")
        self.currentUser = nil
        self.isAuthenticated = false
    }
    
    // MARK: - Stytch SMS Authentication
    
    func sendSMSCode(to phoneNumber: String) async throws -> SMSResponse {
        let url = URL(string: "\(baseURL)/api/login")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")
        
        let formData = "flow=send&phoneNumber=\(phoneNumber)"
        request.httpBody = formData.data(using: .utf8)
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw APIError.networkError
        }
        
        return try JSONDecoder().decode(SMSResponse.self, from: data)
    }
    
    func verifyCode(methodId: String, code: String) async throws -> AuthResponse {
        let url = URL(string: "\(baseURL)/api/login")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")
        
        let formData = "flow=verify&methodId=\(methodId)&code=\(code)"
        request.httpBody = formData.data(using: .utf8)
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse else {
            throw APIError.networkError
        }
        
        if httpResponse.statusCode == 200 {
            let authResponse = try JSONDecoder().decode(AuthResponse.self, from: data)
            if let user = authResponse.user {
                saveSession(user)
            }
            return authResponse
        } else {
            let errorResponse = try JSONDecoder().decode(ErrorResponse.self, from: data)
            throw APIError.authenticationFailed(errorResponse.error)
        }
    }
    
    func logout() {
        clearSession()
    }
    
    // MARK: - Authenticated API Calls
    
    private func authenticatedRequest(url: URL, method: String = "GET", body: Data? = nil) async throws -> URLRequest {
        var request = URLRequest(url: url)
        request.httpMethod = method
        
        if let body = body {
            request.httpBody = body
            request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")
        }
        
        // Add session cookie for authentication
        if let sessionData = UserDefaults.standard.data(forKey: "user_session") {
            // In production, you might want to store and send the session cookie
            // For now, we rely on the web app's session management
        }
        
        return request
    }
    
    func fetchItems() async throws -> [Item] {
        let url = URL(string: "\(baseURL)/api/items")!
        let request = try await authenticatedRequest(url: url)
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw APIError.networkError
        }
        
        let itemsResponse = try JSONDecoder().decode(ItemsResponse.self, from: data)
        return itemsResponse.items
    }
    
    func createItem(name: String, description: String? = nil, imageUrl: String? = nil, category: String? = nil, quantity: Int = 1) async throws -> Item {
        let url = URL(string: "\(baseURL)/api/items")!
        
        var formData = "name=\(name)&quantity=\(quantity)"
        if let description = description {
            formData += "&description=\(description)"
        }
        if let imageUrl = imageUrl {
            formData += "&imageUrl=\(imageUrl)"
        }
        if let category = category {
            formData += "&category=\(category)"
        }
        
        let request = try await authenticatedRequest(
            url: url,
            method: "POST",
            body: formData.data(using: .utf8)
        )
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw APIError.networkError
        }
        
        let itemResponse = try JSONDecoder().decode(ItemResponse.self, from: data)
        return itemResponse.item
    }
    
    func updateItem(_ item: Item) async throws -> Item {
        let url = URL(string: "\(baseURL)/api/items")!
        
        let formData = "id=\(item.id)&name=\(item.name)&description=\(item.description ?? "")&quantity=\(item.quantity)"
        
        let request = try await authenticatedRequest(
            url: url,
            method: "PUT",
            body: formData.data(using: .utf8)
        )
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw APIError.networkError
        }
        
        let itemResponse = try JSONDecoder().decode(ItemResponse.self, from: data)
        return itemResponse.item
    }
    
    func deleteItem(_ item: Item) async throws {
        let url = URL(string: "\(baseURL)/api/items")!
        
        let formData = "id=\(item.id)"
        
        let request = try await authenticatedRequest(
            url: url,
            method: "DELETE",
            body: formData.data(using: .utf8)
        )
        
        let (data, response) = try await session.data(for: request)
        
        guard let httpResponse = response as? HTTPURLResponse,
              httpResponse.statusCode == 200 else {
            throw APIError.networkError
        }
    }
}
    
    func getCurrentUser() async throws -> User {
        let url = URL(string: "\(baseURL)/api/auth")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(getAuthToken())", forHTTPHeaderField: "Authorization")
        
        let formData = "action=me"
        request.httpBody = formData.data(using: .utf8)
        
        let (data, _) = try await session.data(for: request)
        let response = try JSONDecoder().decode(UserResponse.self, from: data)
        return response.user
    }
    
    // MARK: - Items API
    
    func fetchItems() async throws -> [Item] {
        let url = URL(string: "\(baseURL)/api/items")!
        var request = URLRequest(url: url)
        request.setValue("Bearer \(getAuthToken())", forHTTPHeaderField: "Authorization")
        
        let (data, _) = try await session.data(for: request)
        let response = try JSONDecoder().decode(ItemsResponse.self, from: data)
        return response.items
    }
    
    func createItem(name: String, description: String?, imageUrl: String?, category: String?) async throws -> Item {
        let url = URL(string: "\(baseURL)/api/items")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")
        request.setValue("Bearer \(getAuthToken())", forHTTPHeaderField: "Authorization")
        
        var formData = "name=\(name.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")"
        if let description = description {
            formData += "&description=\(description.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")"
        }
        if let imageUrl = imageUrl {
            formData += "&imageUrl=\(imageUrl.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")"
        }
        if let category = category {
            formData += "&category=\(category.addingPercentEncoding(withAllowedCharacters: .urlQueryAllowed) ?? "")"
        }
        
        request.httpBody = formData.data(using: .utf8)
        
        let (data, _) = try await session.data(for: request)
        let response = try JSONDecoder().decode(ItemResponse.self, from: data)
        return response.item
    }
    
    private func getAuthToken() -> String {
        return UserDefaults.standard.string(forKey: "auth_token") ?? ""
    }
}

// MARK: - Response Models

struct AuthResponse: Codable {
    let user: User
    let token: String
}

struct UserResponse: Codable {
    let user: User
}

struct ItemsResponse: Codable {
    let items: [Item]
}

struct ItemResponse: Codable {
    let item: Item
}
```

### 3.5 iOS Models

Create `apps/ios-project/YourApp/Models/User.swift`:

```swift
import Foundation

struct User: Codable, Identifiable {
    let id: String
    let email: String
    let phone: String?
    let createdAt: String
}
```

Create `apps/ios-project/YourApp/Models/Item.swift`:

```swift
import Foundation

struct Item: Codable, Identifiable {
    let id: String
    let name: String
    let description: String?
    let imageUrl: String?
    let category: String?
    let quantity: Int
    let createdAt: String
    let updatedAt: String
}

// MARK: - Authentication Models

struct User: Codable {
    let id: String
    let phone: String?
    let email: String
    let createdAt: String
}

struct AuthResponse: Codable {
    let success: Bool
    let user: User?
    let sessionId: String?
    let error: String?
}

struct SMSResponse: Codable {
    let success: Bool
    let methodId: String
    let error: String?
}

struct ErrorResponse: Codable {
    let error: String
}

// MARK: - API Response Models

struct ItemsResponse: Codable {
    let items: [Item]
}

struct ItemResponse: Codable {
    let item: Item
}

// MARK: - API Errors

enum APIError: Error, LocalizedError {
    case networkError
    case authenticationFailed(String)
    case invalidData
    
    var errorDescription: String? {
        switch self {
        case .networkError:
            return "Network error occurred"
        case .authenticationFailed(let message):
            return "Authentication failed: \(message)"
        case .invalidData:
            return "Invalid data received"
        }
    }
}
```

### 3.5 iOS Authentication Views

Create `apps/ios-project/YourApp/Views/LoginView.swift`:

```swift
import SwiftUI

struct LoginView: View {
    @StateObject private var apiClient = APIClient.shared
    @State private var phoneNumber = ""
    @State private var verificationCode = ""
    @State private var methodId = ""
    @State private var currentStep: AuthStep = .phoneEntry
    @State private var isLoading = false
    @State private var errorMessage = ""
    @State private var showError = false
    
    enum AuthStep {
        case phoneEntry
        case codeVerification
    }
    
    var body: some View {
        NavigationView {
            VStack(spacing: 24) {
                VStack(spacing: 16) {
                    Image(systemName: "phone.fill")
                        .font(.system(size: 60))
                        .foregroundColor(.blue)
                    
                    Text("Welcome to Your App")
                        .font(.title)
                        .fontWeight(.bold)
                    
                    Text("Enter your phone number to get started")
                        .font(.body)
                        .foregroundColor(.secondary)
                        .multilineTextAlignment(.center)
                }
                
                VStack(spacing: 16) {
                    if currentStep == .phoneEntry {
                        phoneEntryView
                    } else {
                        codeVerificationView
                    }
                }
                
                Spacer()
            }
            .padding()
            .navigationBarHidden(true)
            .alert("Error", isPresented: $showError) {
                Button("OK") { }
            } message: {
                Text(errorMessage)
            }
        }
    }
    
    private var phoneEntryView: some View {
        VStack(spacing: 16) {
            VStack(alignment: .leading, spacing: 8) {
                Text("Phone Number")
                    .font(.headline)
                
                TextField("Enter your phone number", text: $phoneNumber)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .keyboardType(.phonePad)
                    .textContentType(.telephoneNumber)
            }
            
            Button(action: sendSMSCode) {
                HStack {
                    if isLoading {
                        ProgressView()
                            .scaleEffect(0.8)
                            .foregroundColor(.white)
                    }
                    
                    Text("Send Verification Code")
                        .fontWeight(.medium)
                }
                .frame(maxWidth: .infinity)
                .padding()
                .background(Color.blue)
                .foregroundColor(.white)
                .cornerRadius(10)
            }
            .disabled(phoneNumber.isEmpty || isLoading)
        }
    }
    
    private var codeVerificationView: some View {
        VStack(spacing: 16) {
            VStack(alignment: .leading, spacing: 8) {
                Text("Verification Code")
                    .font(.headline)
                
                Text("Enter the code sent to \(phoneNumber)")
                    .font(.caption)
                    .foregroundColor(.secondary)
                
                TextField("Enter verification code", text: $verificationCode)
                    .textFieldStyle(RoundedBorderTextFieldStyle())
                    .keyboardType(.numberPad)
                    .textContentType(.oneTimeCode)
            }
            
            Button(action: verifyCode) {
                HStack {
                    if isLoading {
                        ProgressView()
                            .scaleEffect(0.8)
                            .foregroundColor(.white)
                    }
                    
                    Text("Verify Code")
                        .fontWeight(.medium)
                }
                .frame(maxWidth: .infinity)
                .padding()
                .background(Color.blue)
                .foregroundColor(.white)
                .cornerRadius(10)
            }
            .disabled(verificationCode.isEmpty || isLoading)
            
            Button("Change Phone Number") {
                currentStep = .phoneEntry
                verificationCode = ""
                methodId = ""
            }
            .foregroundColor(.blue)
        }
    }
    
    private func sendSMSCode() {
        guard !phoneNumber.isEmpty else { return }
        
        isLoading = true
        errorMessage = ""
        
        Task {
            do {
                let response = try await apiClient.sendSMSCode(to: phoneNumber)
                
                await MainActor.run {
                    if response.success {
                        methodId = response.methodId
                        currentStep = .codeVerification
                    } else {
                        errorMessage = response.error ?? "Failed to send SMS"
                        showError = true
                    }
                    isLoading = false
                }
            } catch {
                await MainActor.run {
                    errorMessage = error.localizedDescription
                    showError = true
                    isLoading = false
                }
            }
        }
    }
    
    private func verifyCode() {
        guard !verificationCode.isEmpty, !methodId.isEmpty else { return }
        
        isLoading = true
        errorMessage = ""
        
        Task {
            do {
                let response = try await apiClient.verifyCode(methodId: methodId, code: verificationCode)
                
                await MainActor.run {
                    if response.success {
                        // Authentication successful, APIClient will handle state updates
                        isLoading = false
                    } else {
                        errorMessage = response.error ?? "Invalid verification code"
                        showError = true
                        isLoading = false
                    }
                }
            } catch {
                await MainActor.run {
                    errorMessage = error.localizedDescription
                    showError = true
                    isLoading = false
                }
            }
        }
    }
}

struct LoginView_Previews: PreviewProvider {
    static var previews: some View {
        LoginView()
    }
}
```

### 3.6 iOS Main App Integration

Update `apps/ios-project/YourApp/YourAppApp.swift`:

```swift
import SwiftUI

@main
struct YourAppApp: App {
    @StateObject private var apiClient = APIClient.shared
    
    var body: some Scene {
        WindowGroup {
            Group {
                if apiClient.isAuthenticated {
                    ContentView()
                } else {
                    LoginView()
                }
            }
            .environmentObject(apiClient)
        }
    }
}
```

### 3.7 iOS Content View with Authentication

Update `apps/ios-project/YourApp/Views/ContentView.swift`:

```swift
import SwiftUI

struct ContentView: View {
    @EnvironmentObject var apiClient: APIClient
    @State private var items: [Item] = []
    @State private var isLoading = false
    @State private var showingAddItem = false
    
    var body: some View {
        NavigationView {
            VStack {
                if isLoading {
                    ProgressView("Loading items...")
                        .frame(maxWidth: .infinity, maxHeight: .infinity)
                } else if items.isEmpty {
                    emptyStateView
                } else {
                    itemsList
                }
            }
            .navigationTitle("My Items")
            .navigationBarTitleDisplayMode(.large)
            .toolbar {
                ToolbarItem(placement: .navigationBarTrailing) {
                    Menu {
                        Button("Add Item") {
                            showingAddItem = true
                        }
                        
                        Button("Logout") {
                            apiClient.logout()
                        }
                    } label: {
                        Image(systemName: "ellipsis.circle")
                    }
                }
            }
            .sheet(isPresented: $showingAddItem) {
                AddItemView { newItem in
                    items.append(newItem)
                }
            }
        }
        .task {
            await loadItems()
        }
    }
    
    private var emptyStateView: some View {
        VStack(spacing: 20) {
            Image(systemName: "tray")
                .font(.system(size: 60))
                .foregroundColor(.gray)
            
            Text("No items yet")
                .font(.title2)
                .fontWeight(.medium)
            
            Text("Tap the menu button to add your first item")
                .font(.body)
                .foregroundColor(.secondary)
                .multilineTextAlignment(.center)
        }
        .padding()
    }
    
    private var itemsList: some View {
        List {
            ForEach(items, id: \.id) { item in
                ItemRow(item: item)
            }
            .onDelete(perform: deleteItems)
        }
        .refreshable {
            await loadItems()
        }
    }
    
    private func loadItems() async {
        isLoading = true
        
        do {
            let fetchedItems = try await apiClient.fetchItems()
            await MainActor.run {
                items = fetchedItems
                isLoading = false
            }
        } catch {
            await MainActor.run {
                isLoading = false
            }
            print("Error loading items: \(error)")
        }
    }
    
    private func deleteItems(offsets: IndexSet) {
        for index in offsets {
            let item = items[index]
            
            Task {
                do {
                    try await apiClient.deleteItem(item)
                    await MainActor.run {
                        items.remove(at: index)
                    }
                } catch {
                    print("Error deleting item: \(error)")
                }
            }
        }
    }
}

struct ItemRow: View {
    let item: Item
    
    var body: some View {
        HStack {
            VStack(alignment: .leading, spacing: 4) {
                Text(item.name)
                    .font(.headline)
                
                if let description = item.description {
                    Text(description)
                        .font(.caption)
                        .foregroundColor(.secondary)
                }
            }
            
            Spacer()
            
            Text("\(item.quantity)")
                .font(.caption)
                .padding(.horizontal, 8)
                .padding(.vertical, 4)
                .background(Color.blue.opacity(0.1))
                .foregroundColor(.blue)
                .cornerRadius(8)
        }
        .padding(.vertical, 2)
    }
}

struct AddItemView: View {
    @Environment(\.dismiss) var dismiss
    @EnvironmentObject var apiClient: APIClient
    
    @State private var name = ""
    @State private var description = ""
    @State private var quantity = 1
    @State private var isLoading = false
    
    let onItemAdded: (Item) -> Void
    
    var body: some View {
        NavigationView {
            Form {
                Section("Item Details") {
                    TextField("Name", text: $name)
                    TextField("Description", text: $description)
                    
                    Stepper("Quantity: \(quantity)", value: $quantity, in: 1...999)
                }
            }
            .navigationTitle("Add Item")
            .navigationBarTitleDisplayMode(.inline)
            .toolbar {
                ToolbarItem(placement: .navigationBarLeading) {
                    Button("Cancel") {
                        dismiss()
                    }
                }
                
                ToolbarItem(placement: .navigationBarTrailing) {
                    Button("Save") {
                        saveItem()
                    }
                    .disabled(name.isEmpty || isLoading)
                }
            }
        }
    }
    
    private func saveItem() {
        isLoading = true
        
        Task {
            do {
                let newItem = try await apiClient.createItem(
                    name: name,
                    description: description.isEmpty ? nil : description,
                    quantity: quantity
                )
                
                await MainActor.run {
                    onItemAdded(newItem)
                    dismiss()
                }
            } catch {
                await MainActor.run {
                    isLoading = false
                }
                print("Error creating item: \(error)")
            }
        }
    }
}

struct ContentView_Previews: PreviewProvider {
    static var previews: some View {
        ContentView()
            .environmentObject(APIClient.shared)
    }
}
```

## 🚀 Step 4: Railway Deployment

### 4.1 Configure Railway

1. **Connect Repository**:
   ```bash
   railway link
   ```

2. **Set Environment Variables**:
   ```bash
   railway variables set DATABASE_URL="postgresql://..."
   railway variables set SESSION_SECRET="your-secret"
   railway variables set POSTHOG_API_KEY="your-key"
   ```

3. **Deploy**:
   ```bash
   railway deploy
   ```

### 4.2 Custom Railway Configuration

Create `railway.toml`:

```toml
[build]
builder = "nixpacks"
buildCommand = "npm run build"

[deploy]
startCommand = "npm start"
restartPolicyType = "on_failure"
restartPolicyMaxRetries = 10

[environments.production]
variables = { NODE_ENV = "production" }
```

## 🎨 Step 5: Customize Epic Stack

### 5.1 Remove Unnecessary Features

```bash
cd apps/web

# Remove note-related files (if not needed)
rm -rf app/routes/users+/
rm -rf app/routes/settings+/

# Clean up database
# Remove unused models from prisma/schema.prisma
# Keep User, Session, Password, Role models
# Add your Item model
```

### 5.2 Update Branding

1. **Update `app/root.tsx`**:
   ```tsx
   export const meta: MetaFunction = () => {
     return [
       { title: "Your App Name" },
       { name: "description", content: "Your app description" },
     ];
   };
   ```

2. **Update favicons** in `public/favicons/`

3. **Update app icons** in iOS project

## 🔒 Step 6: API Security & Best Practices

### 6.1 Authentication Middleware

Create `apps/web/app/utils/auth.server.ts` to secure your API endpoints:

```typescript
import { redirect } from "@remix-run/node";
import { prisma } from "./db.server";
import { getSession } from "./session.server";

export async function requireUserId(request: Request) {
  const cookie = request.headers.get("Cookie");
  const session = await getSession(cookie);
  const sessionId = session.get("sessionId");
  
  if (!sessionId) {
    throw new Response("Unauthorized", { status: 401 });
  }
  
  const dbSession = await prisma.session.findUnique({
    where: { id: sessionId },
    include: { user: true },
  });
  
  if (!dbSession || dbSession.expirationDate < new Date()) {
    throw new Response("Unauthorized", { status: 401 });
  }
  
  return dbSession.userId;
}

export async function requireUser(request: Request) {
  const userId = await requireUserId(request);
  
  const user = await prisma.user.findUnique({
    where: { id: userId },
    select: {
      id: true,
      email: true,
      phone: true,
      createdAt: true,
    },
  });
  
  if (!user) {
    throw new Response("User not found", { status: 404 });
  }
  
  return user;
}

export async function requireAdminUser(request: Request) {
  const user = await requireUser(request);
  
  const userWithRoles = await prisma.user.findUnique({
    where: { id: user.id },
    include: {
      roles: {
        include: {
          permissions: true,
        },
      },
    },
  });
  
  const hasAdminRole = userWithRoles?.roles.some(
    role => role.name === "admin"
  );
  
  if (!hasAdminRole) {
    throw new Response("Forbidden", { status: 403 });
  }
  
  return user;
}
```

### 6.2 Rate Limiting

Create `apps/web/app/utils/rate-limit.server.ts`:

```typescript
import { LRUCache } from "lru-cache";

const rateLimitCache = new LRUCache<string, number>({
  max: 1000,
  ttl: 60 * 1000, // 1 minute
});

export function rateLimit(
  identifier: string,
  maxRequests: number = 10,
  windowMs: number = 60 * 1000
) {
  const key = `rate_limit:${identifier}`;
  const currentCount = rateLimitCache.get(key) || 0;
  
  if (currentCount >= maxRequests) {
    throw new Response("Too many requests", { status: 429 });
  }
  
  rateLimitCache.set(key, currentCount + 1);
}

export function getClientIP(request: Request): string {
  const forwardedFor = request.headers.get("X-Forwarded-For");
  const realIP = request.headers.get("X-Real-IP");
  
  if (forwardedFor) {
    return forwardedFor.split(",")[0].trim();
  }
  
  if (realIP) {
    return realIP;
  }
  
  return "unknown";
}
```

### 6.3 Input Validation

Create `apps/web/app/utils/validation.server.ts`:

```typescript
import { z } from "zod";

// Install zod: npm install zod

export const ItemSchema = z.object({
  name: z.string().min(1, "Name is required").max(100, "Name too long"),
  description: z.string().max(500, "Description too long").optional(),
  quantity: z.number().int().min(1, "Quantity must be at least 1").max(9999, "Quantity too large"),
  category: z.string().max(50, "Category too long").optional(),
  imageUrl: z.string().url("Invalid URL").optional(),
});

export const PhoneSchema = z.string().regex(
  /^\+?[1-9]\d{1,14}$/,
  "Invalid phone number format"
);

export const SMSCodeSchema = z.string().regex(
  /^\d{6}$/,
  "Verification code must be 6 digits"
);

export function validateFormData<T>(
  schema: z.ZodSchema<T>,
  data: FormData
): T {
  const object = Object.fromEntries(data.entries());
  
  // Convert string numbers to numbers
  for (const [key, value] of Object.entries(object)) {
    if (typeof value === "string" && !isNaN(Number(value)) && value !== "") {
      object[key] = Number(value);
    }
  }
  
  const result = schema.safeParse(object);
  
  if (!result.success) {
    const errors = result.error.errors.map(err => err.message).join(", ");
    throw new Response(`Validation error: ${errors}`, { status: 400 });
  }
  
  return result.data;
}
```

### 6.4 Secure API Route Example

Update `apps/web/app/routes/api+/items.tsx` with comprehensive security:

```typescript
import { json, type ActionFunctionArgs, type LoaderFunctionArgs } from "@remix-run/node";
import { requireUserId } from "~/utils/auth.server";
import { rateLimit, getClientIP } from "~/utils/rate-limit.server";
import { validateFormData, ItemSchema } from "~/utils/validation.server";
import { prisma } from "~/utils/db.server";

export async function loader({ request }: LoaderFunctionArgs) {
  // Rate limiting
  const clientIP = getClientIP(request);
  rateLimit(`api_items_${clientIP}`, 100, 60 * 1000); // 100 requests per minute
  
  // Authentication
  const userId = await requireUserId(request);
  
  // Pagination
  const url = new URL(request.url);
  const page = parseInt(url.searchParams.get("page") || "1");
  const limit = Math.min(parseInt(url.searchParams.get("limit") || "20"), 100);
  const skip = (page - 1) * limit;
  
  const [items, totalCount] = await Promise.all([
    prisma.item.findMany({
      where: { userId },
      orderBy: { createdAt: "desc" },
      skip,
      take: limit,
      select: {
        id: true,
        name: true,
        description: true,
        imageUrl: true,
        category: true,
        quantity: true,
        createdAt: true,
        updatedAt: true,
      },
    }),
    prisma.item.count({ where: { userId } }),
  ]);
  
  return json({
    items,
    pagination: {
      page,
      limit,
      totalCount,
      totalPages: Math.ceil(totalCount / limit),
    },
  });
}

export async function action({ request }: ActionFunctionArgs) {
  const clientIP = getClientIP(request);
  const method = request.method;
  
  // Rate limiting by method
  switch (method) {
    case "POST":
      rateLimit(`api_items_create_${clientIP}`, 10, 60 * 1000); // 10 creates per minute
      break;
    case "PUT":
      rateLimit(`api_items_update_${clientIP}`, 30, 60 * 1000); // 30 updates per minute
      break;
    case "DELETE":
      rateLimit(`api_items_delete_${clientIP}`, 20, 60 * 1000); // 20 deletes per minute
      break;
  }
  
  // Authentication
  const userId = await requireUserId(request);
  const formData = await request.formData();
  
  switch (method) {
    case "POST": {
      const validatedData = validateFormData(ItemSchema, formData);
      
      const item = await prisma.item.create({
        data: {
          ...validatedData,
          userId,
        },
        select: {
          id: true,
          name: true,
          description: true,
          imageUrl: true,
          category: true,
          quantity: true,
          createdAt: true,
          updatedAt: true,
        },
      });
      
      return json({ item }, { status: 201 });
    }
    
    case "PUT": {
      const id = formData.get("id") as string;
      
      if (!id) {
        throw new Response("Item ID is required", { status: 400 });
      }
      
      // Verify ownership
      const existingItem = await prisma.item.findUnique({
        where: { id },
        select: { userId: true },
      });
      
      if (!existingItem || existingItem.userId !== userId) {
        throw new Response("Item not found", { status: 404 });
      }
      
      const validatedData = validateFormData(ItemSchema.partial(), formData);
      
      const item = await prisma.item.update({
        where: { id },
        data: validatedData,
        select: {
          id: true,
          name: true,
          description: true,
          imageUrl: true,
          category: true,
          quantity: true,
          createdAt: true,
          updatedAt: true,
        },
      });
      
      return json({ item });
    }
    
    case "DELETE": {
      const id = formData.get("id") as string;
      
      if (!id) {
        throw new Response("Item ID is required", { status: 400 });
      }
      
      // Verify ownership and delete
      const deletedItem = await prisma.item.deleteMany({
        where: { id, userId },
      });
      
      if (deletedItem.count === 0) {
        throw new Response("Item not found", { status: 404 });
      }
      
      return json({ success: true });
    }
    
    default:
      throw new Response("Method not allowed", { status: 405 });
  }
}
```

### 6.5 iOS Security Considerations

Update your iOS `APIClient.swift` to handle authentication properly:

```swift
// Add these properties to APIClient
private var sessionCookie: String?

// Update the authenticatedRequest method
private func authenticatedRequest(url: URL, method: String = "GET", body: Data? = nil) async throws -> URLRequest {
    var request = URLRequest(url: url)
    request.httpMethod = method
    
    if let body = body {
        request.httpBody = body
        request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")
    }
    
    // Add session cookie if available
    if let sessionCookie = self.sessionCookie {
        request.setValue(sessionCookie, forHTTPHeaderField: "Cookie")
    }
    
    return request
}

// Update verifyCode method to capture session cookie
func verifyCode(methodId: String, code: String) async throws -> AuthResponse {
    // ... existing code ...
    
    let (data, response) = try await session.data(for: request)
    
    // Extract session cookie
    if let httpResponse = response as? HTTPURLResponse,
       let cookies = httpResponse.allHeaderFields["Set-Cookie"] as? String {
        self.sessionCookie = cookies
        UserDefaults.standard.set(cookies, forKey: "session_cookie")
    }
    
    // ... rest of existing code ...
}

// Load session cookie on init
private func loadStoredSession() {
    if let sessionData = UserDefaults.standard.data(forKey: "user_session"),
       let user = try? JSONDecoder().decode(User.self, from: sessionData) {
        self.currentUser = user
        self.isAuthenticated = true
        self.sessionCookie = UserDefaults.standard.string(forKey: "session_cookie")
    }
}
```

### 6.6 Environment Variables Security

Create secure environment configuration:

```bash
# Production Environment Variables
NODE_ENV=production
DATABASE_URL="postgresql://encrypted_connection_string"
SESSION_SECRET="use-a-long-random-string-here"
STYTCH_PROJECT_ID="project-live-xxx"
STYTCH_SECRET="secret-live-xxx"

# Rate Limiting
RATE_LIMIT_ENABLED=true
RATE_LIMIT_WINDOW_MS=60000
RATE_LIMIT_MAX_REQUESTS=100

# Security Headers
SECURE_HEADERS_ENABLED=true
HELMET_ENABLED=true
```

### 6.7 Security Headers

Add security headers to your app:

```typescript
// Add to apps/web/app/root.tsx
export function headers() {
  return {
    "X-Frame-Options": "DENY",
    "X-Content-Type-Options": "nosniff",
    "Referrer-Policy": "strict-origin-when-cross-origin",
    "Permissions-Policy": "camera=(), microphone=(), geolocation=()",
  };
}
```

## 🔧 Step 7: Testing & Development

### 7.1 Web Development

```bash
cd apps/web

# Start development server
npm run dev

# Run tests
npm run test

# Run migrations
npm run db:migrate
```

### 7.2 iOS Development

1. Open `apps/ios-project/YourApp.xcworkspace` in Xcode
2. Update bundle identifier and team
3. Run on simulator or device

### 7.3 API Testing with Stytch Authentication

Test your API endpoints with proper authentication:

```bash
# Test SMS send
curl -X POST "https://yourdomain.com/api/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "flow=send&phoneNumber=+1234567890"

# Test SMS verify
curl -X POST "https://yourdomain.com/api/login" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "flow=verify&methodId=METHOD_ID&code=123456"

# Test authenticated items endpoint (with session cookie)
curl -X GET "https://yourdomain.com/api/items" \
  -H "Cookie: __session=your-session-cookie"

# Test create item
curl -X POST "https://yourdomain.com/api/items" \
  -H "Cookie: __session=your-session-cookie" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "name=Test Item&description=A test item&quantity=1"
```

## 📱 Step 8: iOS Build & Distribution

### 8.1 Build Scripts

Create `apps/ios-project/build-and-upload.sh`:

```bash
#!/bin/bash

set -e

SCHEME="YourApp"
WORKSPACE="YourApp.xcworkspace"
BUILD_DIR="build"
ARCHIVE_PATH="$BUILD_DIR/$SCHEME.xcarchive"
IPA_DIR="$BUILD_DIR/ipa"

echo "🚀 Building and uploading $SCHEME to TestFlight..."

# Clean and create build directory
rm -rf "$BUILD_DIR"
mkdir -p "$BUILD_DIR"

# Build and archive
echo "📦 Creating archive..."
xcodebuild -workspace "$WORKSPACE" \
  -scheme "$SCHEME" \
  -configuration Release \
  -archivePath "$ARCHIVE_PATH" \
  -destination generic/platform=iOS \
  archive

# Export IPA
echo "📤 Exporting IPA..."
xcodebuild -exportArchive \
  -archivePath "$ARCHIVE_PATH" \
  -exportPath "$IPA_DIR" \
  -exportOptionsPlist ExportOptions.plist

# Upload to TestFlight
echo "🛫 Uploading to TestFlight..."
xcrun altool --upload-app \
  --type ios \
  --file "$IPA_DIR/$SCHEME.ipa" \
  --username "$XCODE_CLI_ACCT" \
  --password "$XCODE_CLI_PW"

echo "✅ Upload complete!"
```

## 🎯 Step 9: Production Checklist

### 9.1 Epic Stack Features Already Included ✅
- [x] Authentication with sessions
- [x] Database ORM with Prisma
- [x] Email functionality
- [x] Testing setup (Playwright, Vitest)
- [x] TypeScript configuration
- [x] Styling with Tailwind
- [x] Code formatting and linting
- [x] Error handling
- [x] Security best practices

### 9.2 Stytch SMS Authentication Features ✅
- [x] SMS-based passwordless authentication
- [x] Secure session management
- [x] Cross-platform authentication (web + iOS)
- [x] Rate limiting and input validation
- [x] Proper error handling
- [x] Session persistence

### 9.3 Additional Configuration
- [ ] PostHog analytics configured
- [ ] Railway deployment successful
- [ ] iOS app signed and uploaded
- [ ] Stytch production keys configured
- [ ] API endpoints tested with authentication
- [ ] Environment variables set
- [ ] Domain configured
- [ ] Security headers enabled

## 🔄 Step 10: Continuous Integration

The Epic Stack includes GitHub Actions out of the box. Update `.github/workflows/deploy.yml` for Railway:

```yaml
name: Deploy
on:
  push:
    branches: [main]
  pull_request:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
        with:
          node-version: 18
      - run: npm ci
      - run: npm run build
      - run: npm run test
      - name: Deploy to Railway
        if: github.ref == 'refs/heads/main'
        uses: railway/deploy@v1
        with:
          railway-token: ${{ secrets.RAILWAY_TOKEN }}
```

## 🎉 Create New Apps with This Recipe

### Quick Start Command

```bash
# 1. Create new Epic Stack project
npx epicli new my-new-app

# 2. Convert to monorepo and add iOS
cd my-new-app
mkdir -p apps/ios-project
mv * apps/web/ 2>/dev/null || true
mv .* apps/web/ 2>/dev/null || true
mv apps/web/.gitignore apps/web/package.json apps/web/README.md .

# 3. Follow the recipe steps above
```

### What You Get Out of the Box

Thanks to the Epic Stack foundation:
- ✅ **Production-ready Remix app** with authentication
- ✅ **PostgreSQL database** with Prisma ORM
- ✅ **Type-safe forms** with Conform
- ✅ **Comprehensive testing** setup
- ✅ **Security best practices** implemented
- ✅ **Railway deployment** ready
- ✅ **Professional UI** with Tailwind + Radix
- ✅ **iOS app** consuming web APIs

## 🤖 Step 10: AI-Powered Development with Cursor & MCP

### 10.1 Setting Up Model Context Protocol (MCP)

MCP enables AI assistants to access external tools and data sources. For iOS development, XcodeBuildMCP provides comprehensive Xcode integration.

#### Configure MCP in Cursor

1. **Open Cursor Settings** and navigate to MCP configuration:
   ```json
   {
     "mcpServers": {
       "XcodeBuildMCP": {
         "command": "npx",
         "args": [
           "-y",
           "xcodebuildmcp@latest"
         ]
       }
     }
   }
   ```

2. **Alternative installation using mise** (avoids Node.js dependency):
   ```bash
   # Install mise
   brew install mise
   ```
   
   ```json
   {
     "mcpServers": {
       "XcodeBuildMCP": {
         "command": "mise",
         "args": [
           "x",
           "npm:xcodebuildmcp@1.10.4",
           "--",
           "xcodebuildmcp"
         ]
       }
     }
   }
   ```

3. **Enable experimental features** (optional):
   ```json
   {
     "mcpServers": {
       "XcodeBuildMCP": {
         "command": "npx",
         "args": ["-y", "xcodebuildmcp@latest"],
         "env": {
           "INCREMENTAL_BUILDS_ENABLED": "true",
           "SENTRY_DISABLED": "true"
         }
       }
     }
   }
   ```

### 10.2 XcodeBuildMCP Capabilities

#### Core Build Operations
- **Project Discovery**: Automatically find `.xcodeproj` and `.xcworkspace` files
- **Scheme Management**: List and select build schemes
- **Multi-target Building**: Build for iOS, macOS, watchOS, tvOS, visionOS
- **Simulator Support**: Build and run on iOS simulators
- **Device Support**: Build and deploy to physical devices
- **Swift Package Integration**: Build and test Swift packages

#### Advanced Features
- **UI Automation**: Interact with simulator UI programmatically
- **Log Capture**: Capture and analyze app logs
- **Screenshot/Screen Recording**: Visual debugging capabilities
- **Performance Testing**: Run XCTest suites with detailed results
- **Code Signing**: Automatic provisioning profile management

### 10.3 Development Workflows

#### 10.3.1 Initial Project Setup

```bash
# Let AI help discover your project structure
@cursor "Discover available Xcode projects and schemes in this workspace"

# AI will use XcodeBuildMCP to:
# 1. Find .xcodeproj and .xcworkspace files
# 2. List available schemes
# 3. Check build settings
# 4. Verify dependencies
```

#### 10.3.2 Building and Testing

```bash
# Build for simulator
@cursor "Build the iOS app for iPhone 16 simulator"

# Build and run with logging
@cursor "Build and run the app on simulator, then capture logs"

# Run tests
@cursor "Run all unit tests on iPhone 16 simulator and show results"

# Build for device
@cursor "Build the app for physical device deployment"
```

#### 10.3.3 UI Testing and Debugging

```bash
# Interactive UI testing
@cursor "Take a screenshot of the simulator, then tap the 'Add Item' button"

# Automated UI flows
@cursor "Test the complete user onboarding flow: signup, login, add first item"

# Debug UI issues
@cursor "Help me debug why the floating action button isn't responding to taps"
```

### 10.4 Best Practices for AI-Assisted iOS Development

#### 10.4.1 Effective Prompting

**✅ Good prompts:**
- "Build the iOS app for iPhone 16 simulator and show any compilation errors"
- "Run the test suite and help me fix any failing tests"
- "Take a screenshot and help me understand why the UI layout is broken"

**❌ Avoid vague prompts:**
- "Fix my app"
- "Make it work"
- "Test everything"

#### 10.4.2 Iterative Development

1. **Start with discovery**: Let AI explore your project structure
2. **Build incrementally**: Fix compilation errors one at a time
3. **Test early and often**: Run tests after each significant change
4. **Use visual feedback**: Screenshots help AI understand UI issues

#### 10.4.3 Code Review and Refactoring

```bash
# AI-assisted code review
@cursor "Review my SwiftUI code in ContentView.swift and suggest improvements"

# Refactoring help
@cursor "Help me refactor this view model to use async/await patterns"

# Architecture guidance
@cursor "Analyze my app structure and suggest MVVM improvements"
```

### 10.5 Common Workflows

#### 10.5.1 New Feature Development

```bash
# 1. Planning phase
@cursor "I want to add a camera feature to scan items. What iOS frameworks should I use?"

# 2. Implementation phase
@cursor "Help me implement AVFoundation camera capture in SwiftUI"

# 3. Testing phase
@cursor "Build and test the camera feature on iPhone 16 simulator"

# 4. Debugging phase
@cursor "The camera isn't working on device. Help me debug the issue"
```

#### 10.5.2 Performance Optimization

```bash
# Profile and optimize
@cursor "Run performance tests and help me identify bottlenecks"

# Memory management
@cursor "Analyze my view controllers for memory leaks"

# Build optimization
@cursor "Help me optimize build times by analyzing dependencies"
```

#### 10.5.3 Deployment Pipeline

```bash
# Archive and export
@cursor "Create an archive for App Store distribution"

# TestFlight upload
@cursor "Upload the latest build to TestFlight"

# Code signing issues
@cursor "Help me fix code signing errors for device deployment"
```

### 10.6 Troubleshooting

#### 10.6.1 Common Issues

**Build Failures:**
```bash
# Let AI diagnose build issues
@cursor "My iOS app won't build. Can you run diagnostics and help me fix the errors?"
```

**Simulator Issues:**
```bash
# Reset simulator state
@cursor "Reset the iOS simulator and install a fresh copy of my app"
```

**Device Deployment:**
```bash
# Fix provisioning
@cursor "Help me fix provisioning profile issues for device deployment"
```

#### 10.6.2 Diagnostic Commands

```bash
# Run comprehensive diagnostics
npx --package xcodebuildmcp@latest xcodebuildmcp-diagnostic

# Check specific issues
@cursor "Run diagnostics and tell me what's wrong with my Xcode setup"
```

### 10.7 Advanced MCP Configuration

#### 10.7.1 Selective Tool Registration

For better performance, enable only the tools you need:

```json
{
  "mcpServers": {
    "XcodeBuildMCP": {
      "command": "npx",
      "args": ["-y", "xcodebuildmcp@latest"],
      "env": {
        "XCODEBUILDMCP_GROUP_IOS_SIMULATOR_WORKFLOW": "true",
        "XCODEBUILDMCP_GROUP_SWIFT_PACKAGE_WORKFLOW": "true"
      }
    }
  }
}
```

#### 10.7.2 Privacy and Security

```json
{
  "mcpServers": {
    "XcodeBuildMCP": {
      "command": "npx",
      "args": ["-y", "xcodebuildmcp@latest"],
      "env": {
        "SENTRY_DISABLED": "true"
      }
    }
  }
}
```

### 10.8 Integration with Epic Stack

#### 10.8.1 API Development

```bash
# Test API endpoints from iOS
@cursor "Help me test the /api/items endpoint from my iOS app"

# Debug authentication
@cursor "Debug why my iOS app authentication isn't working with the Remix API"
```

#### 10.8.2 Full-Stack Testing

```bash
# End-to-end testing
@cursor "Test the complete flow: web signup, then iOS app login with same credentials"

# Data synchronization
@cursor "Help me verify that data created in the web app appears in the iOS app"
```

### 10.9 Performance Tips

1. **Use incremental builds** when enabled
2. **Leverage build caching** for faster iterations
3. **Run tests on specific devices** to reduce overhead
4. **Use selective tool registration** to reduce context overhead
5. **Capture logs selectively** to focus on relevant information

### 10.10 Learning Resources

- **XcodeBuildMCP Documentation**: https://github.com/cameroncooke/XcodeBuildMCP
- **MCP Protocol Specification**: https://modelcontextprotocol.io/
- **Cursor AI Features**: https://cursor.com/features
- **Swift Development with AI**: Apple's official Swift documentation

## 📞 Support & Resources

- **Epic Stack Documentation**: https://www.epicweb.dev/epic-stack
- **Epic Stack GitHub**: https://github.com/epicweb-dev/epic-stack
- **Railway Documentation**: https://railway.app/docs
- **PostHog Documentation**: https://posthog.com/docs
- **XcodeBuildMCP**: https://github.com/cameroncooke/XcodeBuildMCP
- **Model Context Protocol**: https://modelcontextprotocol.io/
- **Cursor AI Features**: https://cursor.com/features

This recipe gives you a battle-tested foundation to build production-ready full-stack applications fast! 🚀 