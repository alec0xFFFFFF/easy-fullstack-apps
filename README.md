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

Create `apps/web/app/routes/api+/auth.tsx` for iOS authentication:

```typescript
import { json, type ActionFunctionArgs } from "@remix-run/node";
import { login, requireUserId } from "~/utils/auth.server";
import { prisma } from "~/utils/db.server";

export async function action({ request }: ActionFunctionArgs) {
  const formData = await request.formData();
  const action = formData.get("action");
  
  switch (action) {
    case "login": {
      const email = formData.get("email") as string;
      const password = formData.get("password") as string;
      
      const user = await login({ email, password });
      
      if (!user) {
        return json({ error: "Invalid credentials" }, { status: 401 });
      }
      
      return json({ user, token: "session-token" }); // Implement proper JWT
    }
    
    case "me": {
      const userId = await requireUserId(request);
      const user = await prisma.user.findUnique({
        where: { id: userId },
        select: {
          id: true,
          email: true,
          createdAt: true,
          phone: true,
        },
      });
      
      return json({ user });
    }
    
    default:
      return json({ error: "Invalid action" }, { status: 400 });
  }
}
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

### 3.4 iOS API Client

Create `apps/ios-project/YourApp/Services/APIClient.swift`:

```swift
import Foundation

class APIClient: ObservableObject {
    static let shared = APIClient()
    
    private let baseURL = "https://yourdomain.com" // Your Railway app URL
    private let session = URLSession.shared
    
    private init() {}
    
    // MARK: - Authentication
    
    func login(email: String, password: String) async throws -> AuthResponse {
        let url = URL(string: "\(baseURL)/api/auth")!
        var request = URLRequest(url: url)
        request.httpMethod = "POST"
        request.setValue("application/x-www-form-urlencoded", forHTTPHeaderField: "Content-Type")
        
        let formData = "action=login&email=\(email)&password=\(password)"
        request.httpBody = formData.data(using: .utf8)
        
        let (data, _) = try await session.data(for: request)
        return try JSONDecoder().decode(AuthResponse.self, from: data)
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

## 🔧 Step 6: Testing & Development

### 6.1 Web Development

```bash
cd apps/web

# Start development server
npm run dev

# Run tests
npm run test

# Run migrations
npm run db:migrate
```

### 6.2 iOS Development

1. Open `apps/ios-project/YourApp.xcworkspace` in Xcode
2. Update bundle identifier and team
3. Run on simulator or device

### 6.3 API Testing

Test your API endpoints:

```bash
# Test items endpoint
curl -X GET "https://yourdomain.com/api/items" \
  -H "Authorization: Bearer your-token"

# Test create item
curl -X POST "https://yourdomain.com/api/items" \
  -H "Authorization: Bearer your-token" \
  -d "name=Test Item&description=A test item"
```

## 📱 Step 7: iOS Build & Distribution

### 7.1 Build Scripts

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

## 🎯 Step 8: Production Checklist

### 8.1 Epic Stack Features Already Included ✅
- [x] Authentication with sessions
- [x] Database ORM with Prisma
- [x] Email functionality
- [x] Testing setup (Playwright, Vitest)
- [x] TypeScript configuration
- [x] Styling with Tailwind
- [x] Code formatting and linting
- [x] Error handling
- [x] Security best practices

### 8.2 Additional Configuration
- [ ] PostHog analytics configured
- [ ] Railway deployment successful
- [ ] iOS app signed and uploaded
- [ ] API endpoints tested
- [ ] Environment variables set
- [ ] Domain configured

## 🔄 Step 9: Continuous Integration

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