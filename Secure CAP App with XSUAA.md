# Secure CAP App with XSUAA | Add Role-Based Authentication in SAP BTP

## Table of Contents
- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step 1: Define Security Model in CDS](#step-1-define-security-model-in-cds)
- [Step 2: Configure XSUAA Security](#step-2-configure-xsuaa-security)
- [Step 3: Update Package Dependencies](#step-3-update-package-dependencies)
- [Step 4: Configure MTA Deployment](#step-4-configure-mta-deployment)
- [Step 5: Create Application Router](#step-5-create-application-router)
- [Step 6: Implement Custom Authorization Logic](#step-6-implement-custom-authorization-logic)
- [Step 7: Deploy to SAP BTP](#step-7-deploy-to-sap-btp)
- [Step 8: Assign Roles to Users](#step-8-assign-roles-to-users)
- [Step 9: Test Authentication](#step-9-test-authentication)
- [Additional Security Best Practices](#additional-security-best-practices)
- [Troubleshooting](#troubleshooting)

---

## Overview

XSUAA (SAP Authorization and Trust Management Service) provides OAuth 2.0-based authentication and authorization for applications on SAP BTP. This guide demonstrates how to implement role-based access control in your CAP (Cloud Application Programming Model) application.

### Key Components
- **XSUAA Service**: Handles OAuth 2.0 authentication and JWT token validation
- **Scopes**: Define fine-grained permissions
- **Role Templates**: Group scopes together
- **Role Collections**: Assigned to users in SAP BTP Cockpit
- **AppRouter**: Entry point that handles authentication flow

---

## Prerequisites

- SAP BTP account with Cloud Foundry environment
- Node.js and npm installed
- Cloud Foundry CLI installed
- MBT (Multi-Target Application) build tool installed
- Basic knowledge of CAP development

```bash
# Install required tools
npm install -g @sap/cds-dk
npm install -g mbt
```

---

## Step 1: Define Security Model in CDS

Create your data model with authorization annotations in `db/schema.cds`:

```cds
namespace myapp;

using { managed, cuid } from '@sap/cds/common';

// Define entities with authorization
entity Books : managed, cuid {
  title       : String(100);
  author      : String(100);
  stock       : Integer;
  price       : Decimal(10,2);
}

entity Orders : managed, cuid {
  book        : Association to Books;
  quantity    : Integer;
  totalPrice  : Decimal(10,2);
  status      : String(20);
}

// Annotations for authorization
annotate Books with @(restrict: [
  { grant: 'READ', to: 'Viewer' },
  { grant: ['READ','WRITE'], to: 'Admin' }
]);

annotate Orders with @(restrict: [
  { grant: 'READ', where: 'createdBy = $user' },
  { grant: ['READ','WRITE'], to: 'Admin' }
]);
```

### Authorization Annotations Explained

- `@restrict`: Defines access control rules
- `grant`: Specifies allowed operations (READ, WRITE, CREATE, UPDATE, DELETE)
- `to`: Specifies the role name
- `where`: Adds conditions for row-level security

---

## Step 2: Configure XSUAA Security

Create `xs-security.json` in your project root:

```json
{
  "xsappname": "myapp",
  "tenant-mode": "dedicated",
  "description": "Security profile for CAP application",
  "scopes": [
    {
      "name": "$XSAPPNAME.Viewer",
      "description": "View access"
    },
    {
      "name": "$XSAPPNAME.Admin",
      "description": "Administrative access"
    }
  ],
  "role-templates": [
    {
      "name": "Viewer",
      "description": "Role for viewing data",
      "scope-references": [
        "$XSAPPNAME.Viewer"
      ]
    },
    {
      "name": "Admin",
      "description": "Role for administrators",
      "scope-references": [
        "$XSAPPNAME.Viewer",
        "$XSAPPNAME.Admin"
      ]
    }
  ],
  "role-collections": [
    {
      "name": "MyApp_Viewer",
      "description": "Viewer Role Collection",
      "role-template-references": [
        "$XSAPPNAME.Viewer"
      ]
    },
    {
      "name": "MyApp_Admin",
      "description": "Admin Role Collection",
      "role-template-references": [
        "$XSAPPNAME.Admin"
      ]
    }
  ],
  "oauth2-configuration": {
    "redirect-uris": [
      "https://*.cfapps.*.hana.ondemand.com/**"
    ]
  }
}
```

### Security Configuration Explained

- **scopes**: Technical permissions in the application
- **role-templates**: Logical groupings of scopes
- **role-collections**: User-assignable roles in BTP Cockpit
- **$XSAPPNAME**: Placeholder replaced with actual app name during deployment

---

## Step 3: Update Package Dependencies

Update your `package.json` to include authentication dependencies:

```json
{
  "name": "myapp",
  "version": "1.0.0",
  "dependencies": {
    "@sap/cds": "^7",
    "@sap/xssec": "^3",
    "@sap/xsenv": "^4",
    "express": "^4",
    "passport": "^0.6"
  },
  "devDependencies": {
    "@sap/cds-dk": "^7"
  },
  "scripts": {
    "start": "cds run"
  },
  "cds": {
    "requires": {
      "auth": {
        "kind": "xsuaa"
      },
      "db": {
        "kind": "hana"
      }
    }
  }
}
```

Install dependencies:

```bash
npm install
```

---

## Step 4: Configure MTA Deployment

Create `mta.yaml` in your project root:

```yaml
_schema-version: '3.1'
ID: myapp
version: 1.0.0
description: "CAP Application with XSUAA"

parameters:
  enable-parallel-deployments: true

modules:
  # --------------------- SERVER MODULE ---------------------
  - name: myapp-srv
    type: nodejs
    path: gen/srv
    parameters:
      buildpack: nodejs_buildpack
      memory: 256M
      disk-quota: 512M
    build-parameters:
      builder: npm
    provides:
      - name: srv-api
        properties:
          srv-url: ${default-url}
    requires:
      - name: myapp-db
      - name: myapp-auth

  # --------------------- DATABASE MODULE ---------------------
  - name: myapp-db-deployer
    type: hdb
    path: gen/db
    parameters:
      buildpack: nodejs_buildpack
      memory: 256M
      disk-quota: 512M
    requires:
      - name: myapp-db

  # --------------------- APPROUTER MODULE ---------------------
  - name: myapp-approuter
    type: approuter.nodejs
    path: approuter
    parameters:
      memory: 256M
      disk-quota: 512M
    requires:
      - name: srv-api
        group: destinations
        properties:
          name: srv-api
          url: ~{srv-url}
          forwardAuthToken: true
      - name: myapp-auth
      - name: myapp-destination
        group: destinations
        properties:
          name: ui5
          url: https://ui5.sap.com
          forwardAuthToken: false

resources:
  # --------------------- XSUAA SERVICE ---------------------
  - name: myapp-auth
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json
      config:
        xsappname: myapp-${org}-${space}
        tenant-mode: dedicated
        role-collections:
          - name: 'MyApp_Admin'
            description: 'Administrator'
            role-template-references:
              - '$XSAPPNAME.Admin'
          - name: 'MyApp_Viewer'
            description: 'Viewer'
            role-template-references:
              - '$XSAPPNAME.Viewer'

  # --------------------- HANA HDI CONTAINER ---------------------
  - name: myapp-db
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared

  # --------------------- DESTINATION SERVICE ---------------------
  - name: myapp-destination
    type: org.cloudfoundry.managed-service
    parameters:
      service: destination
      service-plan: lite
```

---

## Step 5: Create Application Router

### 5.1 Create AppRouter Directory Structure

```bash
mkdir -p approuter
cd approuter
```

### 5.2 Create `xs-app.json`

Create `approuter/xs-app.json`:

```json
{
  "welcomeFile": "/index.html",
  "authenticationMethod": "route",
  "sessionTimeout": 30,
  "routes": [
    {
      "source": "^/api/(.*)$",
      "target": "$1",
      "destination": "srv-api",
      "authenticationType": "xsuaa",
      "csrfProtection": true
    },
    {
      "source": "^/(.*)$",
      "target": "$1",
      "service": "html5-apps-repo-rt",
      "authenticationType": "xsuaa"
    }
  ],
  "logout": {
    "logoutEndpoint": "/do/logout",
    "logoutPage": "/logout.html"
  }
}
```

### 5.3 Create `package.json` for AppRouter

Create `approuter/package.json`:

```json
{
  "name": "myapp-approuter",
  "version": "1.0.0",
  "description": "Application Router for myapp",
  "scripts": {
    "start": "node node_modules/@sap/approuter/approuter.js"
  },
  "dependencies": {
    "@sap/approuter": "^14"
  }
}
```

---

## Step 6: Implement Custom Authorization Logic

Create service implementation in `srv/service.js`:

```javascript
const cds = require('@sap/cds');

module.exports = cds.service.impl(async function() {
  const { Books, Orders } = this.entities;

  // Check if user has Admin role
  this.before('CREATE', 'Books', async (req) => {
    if (!req.user.is('Admin')) {
      req.reject(403, 'Only administrators can create books');
    }
  });

  this.before('UPDATE', 'Books', async (req) => {
    if (!req.user.is('Admin')) {
      req.reject(403, 'Only administrators can update books');
    }
  });

  this.before('DELETE', 'Books', async (req) => {
    if (!req.user.is('Admin')) {
      req.reject(403, 'Only administrators can delete books');
    }
  });

  // Log user information for debugging
  this.on('READ', 'Books', async (req, next) => {
    console.log('User:', req.user.id);
    console.log('User attributes:', req.user.attr);
    console.log('Is Admin?', req.user.is('Admin'));
    console.log('Is Viewer?', req.user.is('Viewer'));
    
    return next();
  });

  // Custom logic for order creation
  this.before('CREATE', 'Orders', async (req) => {
    const book = await SELECT.one.from(Books).where({ ID: req.data.book_ID });
    
    if (!book) {
      req.reject(404, 'Book not found');
    }
    
    if (book.stock < req.data.quantity) {
      req.reject(400, 'Insufficient stock');
    }
    
    // Calculate total price
    req.data.totalPrice = book.price * req.data.quantity;
    req.data.status = 'Pending';
  });

  // Update stock after order creation
  this.after('CREATE', 'Orders', async (data, req) => {
    await UPDATE(Books)
      .where({ ID: data.book_ID })
      .with({ stock: { '-=': data.quantity } });
  });

  // Get current user information
  this.on('getUserInfo', async (req) => {
    return {
      id: req.user.id,
      roles: {
        isAdmin: req.user.is('Admin'),
        isViewer: req.user.is('Viewer')
      },
      attributes: req.user.attr
    };
  });
});
```

### Key Authorization Methods

- `req.user.is('RoleName')`: Check if user has a specific role
- `req.user.id`: Get current user ID
- `req.user.attr`: Access user attributes
- `req.reject(statusCode, message)`: Reject request with error

---

## Step 7: Deploy to SAP BTP

### 7.1 Login to Cloud Foundry

```bash
cf login -a https://api.cf.eu10.hana.ondemand.com
```

### 7.2 Build the MTA Project

```bash
mbt build
```

This creates a `.mtar` file in the `mta_archives` directory.

### 7.3 Deploy to SAP BTP

```bash
cf deploy mta_archives/myapp_1.0.0.mtar
```

### 7.4 Check Deployment Status

```bash
# List all applications
cf apps

# Check XSUAA service
cf service myapp-auth

# View application logs
cf logs myapp-srv --recent
```

---

## Step 8: Assign Roles to Users

### 8.1 Via SAP BTP Cockpit

1. Navigate to your **Subaccount**
2. Go to **Security** → **Role Collections**
3. Find your role collections (`MyApp_Admin`, `MyApp_Viewer`)
4. Click on a role collection
5. Click **Edit**
6. Add users by entering their email addresses
7. Click **Save**

### 8.2 Via CF CLI (Alternative)

```bash
# Create role collection
cf create-role-collection MyApp_Admin

# Add role to role collection
cf add-role-to-role-collection MyApp_Admin myapp-Admin

# Assign role collection to user
cf assign-role-collection MyApp_Admin user@example.com
```

---

## Step 9: Test Authentication

### 9.1 Create Test File

Create `test-auth.http`:

```http
### Get OAuth Token
# @name getToken
POST https://{{subdomain}}.authentication.{{region}}.hana.ondemand.com/oauth/token
Content-Type: application/x-www-form-urlencoded

grant_type=password
&username={{username}}
&password={{password}}
&client_id={{client_id}}
&client_secret={{client_secret}}

### Extract token
@token = {{getToken.response.body.access_token}}

### Test READ Books (Viewer role)
GET https://{{app-url}}/api/Books
Authorization: Bearer {{token}}

### Test CREATE Book (Admin role required)
POST https://{{app-url}}/api/Books
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "title": "SAP BTP Security Guide",
  "author": "John Doe",
  "stock": 50,
  "price": 29.99
}

### Test UPDATE Book (Admin role required)
PATCH https://{{app-url}}/api/Books({{book_id}})
Authorization: Bearer {{token}}
Content-Type: application/json

{
  "stock": 100
}

### Get User Info
GET https://{{app-url}}/api/getUserInfo
Authorization: Bearer {{token}}
```

### 9.2 Test with Browser

1. Open your application URL (AppRouter URL)
2. You'll be redirected to SAP login page
3. Login with credentials
4. Access protected resources
5. Check browser console for JWT token

---

## Additional Security Best Practices

### 1. Enable Audit Logging

Update `package.json`:

```json
{
  "cds": {
    "requires": {
      "auth": {
        "kind": "xsuaa"
      },
      "audit-log": {
        "kind": "audit-log-to-console"
      }
    }
  }
}
```

### 2. Implement Field-Level Authorization

```cds
// In schema.cds
annotate Books with @(restrict: [
  {
    grant: 'READ',
    to: 'Viewer',
    where: 'stock > 0'
  },
  {
    grant: '*',
    to: 'Admin'
  }
]) {
  price @readonly: { grant: 'UPDATE', to: 'Admin' }
};
```

### 3. Add Instance-Based Authorization

```javascript
// Check ownership
this.before('UPDATE', 'Orders', async (req) => {
  const order = await SELECT.one.from(Orders).where({ ID: req.data.ID });
  
  if (!req.user.is('Admin') && order.createdBy !== req.user.id) {
    req.reject(403, 'You can only update your own orders');
  }
});
```

### 4. Implement Rate Limiting

```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
});

// Apply to all requests
app.use(limiter);
```

### 5. Add CORS Configuration

```javascript
// In server.js
const cors = require('cors');

app.use(cors({
  origin: 'https://your-domain.com',
  credentials: true
}));
```

### 6. Secure Sensitive Data

```cds
// Mark fields as @cds.on.insert/@cds.on.update
entity Users {
  key ID: UUID;
  email: String @assert.format: '[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}$';
  password: String @cds.on.insert: $user @cds.on.update: $user;
}
```

---

## Troubleshooting

### Common Issues

#### 1. **401 Unauthorized Error**

**Problem**: User not authenticated

**Solution**:
- Check if XSUAA service is bound correctly
- Verify JWT token is being sent in Authorization header
- Check if AppRouter is configured correctly

```bash
# Check service bindings
cf env myapp-srv
```

#### 2. **403 Forbidden Error**

**Problem**: User authenticated but lacks required role

**Solution**:
- Verify role assignments in BTP Cockpit
- Check if role names match in `xs-security.json` and CDS annotations
- Review authorization logs

```javascript
// Add debug logging
console.log('User roles:', req.user.roles);
console.log('Required role: Admin');
```

#### 3. **Role Collections Not Appearing**

**Problem**: Role collections not created after deployment

**Solution**:
- Redeploy the application
- Update XSUAA service instance
- Check `xs-security.json` syntax

```bash
# Update XSUAA service
cf update-service myapp-auth -c xs-security.json
```

#### 4. **AppRouter Not Starting**

**Problem**: AppRouter fails to start

**Solution**:
- Check `xs-app.json` syntax
- Verify destinations are configured correctly
- Check environment variables

```bash
# View AppRouter logs
cf logs myapp-approuter --recent
```

#### 5. **Token Expiration Issues**

**Problem**: Token expires too quickly

**Solution**:
Update `xs-security.json`:

```json
{
  "oauth2-configuration": {
    "token-validity": 3600,
    "refresh-token-validity": 86400
  }
}
```

### Debug Mode

Enable debug logging:

```bash
# Set environment variable
cf set-env myapp-srv DEBUG *

# Restart application
cf restart myapp-srv
```

---

## Security Checklist

- [ ] XSUAA service configured with proper scopes
- [ ] Role templates defined for different user types
- [ ] Role collections created and assigned to users
- [ ] AppRouter configured with authentication
- [ ] CDS annotations applied to entities
- [ ] Custom authorization logic implemented
- [ ] Audit logging enabled
- [ ] CSRF protection enabled
- [ ] JWT tokens validated
- [ ] Sensitive data encrypted
- [ ] Rate limiting configured
- [ ] CORS properly configured
- [ ] Error messages don't leak sensitive info
- [ ] Regular security audits scheduled

---

## Additional Resources

- [SAP CAP Documentation](https://cap.cloud.sap/docs/)
- [XSUAA Documentation](https://help.sap.com/docs/CP_AUTHORIZ_TRUST_MNG)
- [Application Router Documentation](https://help.sap.com/docs/BTP/65de2977205c403bbc107264b8eccf4b/01c5f9ba7d6847aaaf069d153b981b51.html)
- [Multi-Target Application Documentation](https://help.sap.com/docs/BTP/65de2977205c403bbc107264b8eccf4b/d04fc0e2ad894545aebfd7126384307c.html)

---

## License

This guide is provided as-is for educational purposes.




















# 🔐 Beginner's Guide: Secure Your SAP CAP App with XSUAA

## 🎯 What Are We Trying to Do?

Imagine you're building a library management app. You want:
- **Visitors** to browse and read book information
- **Librarians** to add, update, and delete books

This guide teaches you how to create these "security gates" in your SAP application.

---

## 📚 Table of Contents

1. [Understanding the Basics](#-understanding-the-basics)
2. [The Big Picture](#-the-big-picture)
3. [Step-by-Step Implementation](#-step-by-step-implementation)
4. [Testing Your Security](#-testing-your-security)
5. [Common Mistakes & How to Fix Them](#-common-mistakes--how-to-fix-them)

---

## 🧠 Understanding the Basics

### What is XSUAA?

Think of XSUAA as a **security guard** for your application:
- It checks if someone is who they say they are (Authentication)
- It checks if they're allowed to do what they want (Authorization)

### Key Terms Explained Simply

| Term | Simple Explanation | Real-World Analogy |
|------|-------------------|-------------------|
| **Authentication** | Proving who you are | Showing your ID card |
| **Authorization** | What you're allowed to do | Your ID shows you're a student, so you can enter the library |
| **Scope** | A specific permission | "Can read books" |
| **Role** | A collection of permissions | "Librarian" can read, add, and delete books |
| **Token** | A digital key that proves who you are | A ticket that shows you paid to enter |

---

## 🗺️ The Big Picture

### How Security Works in Your App

```
User tries to access your app
        ↓
AppRouter (Security Gate)
        ↓
"Do you have a ticket?" → NO → Redirect to login page
        ↓ YES
"What's on your ticket?" (Check roles)
        ↓
Your CAP Application
        ↓
Check specific permissions
        ↓
Allow or Deny access
```

### The 3 Main Components

1. **XSUAA Service** 🔑
   - The security guard
   - Manages users and roles
   - Creates and validates tokens

2. **AppRouter** 🚪
   - The front door of your app
   - First thing users interact with
   - Redirects to login if needed

3. **Your CAP App** 🏢
   - The actual application
   - Checks detailed permissions
   - Does the actual work

---

## 🚀 Step-by-Step Implementation

### 🎬 Before We Start

Make sure you have:
- [ ] Node.js installed
- [ ] SAP BTP account (trial is fine)
- [ ] Basic CAP project created

```bash
# Create a new CAP project
cds init my-secure-app
cd my-secure-app
```

---

### Step 1: Create Your Data Model (The Blueprint) 📋

**What we're doing**: Defining what data exists and who can access it

**File**: `db/schema.cds`

```cds
namespace library;

// This is like creating tables in a database
entity Books {
  key ID    : UUID;
  title     : String(100);
  author    : String(100);
  stock     : Integer;
  price     : Decimal(10,2);
}

entity Orders {
  key ID         : UUID;
  book           : Association to Books;
  quantity       : Integer;
  totalPrice     : Decimal(10,2);
}
```

**Think of it as**: Creating a form with blank fields. We haven't added security yet.

---

### Step 2: Add Security Rules 🛡️

**What we're doing**: Adding rules about who can do what

**Update**: `db/schema.cds`

```cds
namespace library;

entity Books {
  key ID    : UUID;
  title     : String(100);
  author    : String(100);
  stock     : Integer;
  price     : Decimal(10,2);
}

// Add this annotation - it's like putting a lock on the entity
annotate Books with @(restrict: [
  // Rule 1: Viewers can only READ
  { 
    grant: 'READ',           // What they can do
    to: 'Viewer'             // Who can do it
  },
  // Rule 2: Admins can do everything
  { 
    grant: ['READ','WRITE'], // Multiple permissions
    to: 'Admin' 
  }
]);
```

**Analogy**: 
- `@restrict` = Putting a lock on a door
- `grant: 'READ'` = Giving someone a "read-only" key
- `to: 'Viewer'` = Specifying who gets the key

---

### Step 3: Define Your Security Configuration 🔧

**What we're doing**: Creating the roles and permissions

**File**: `xs-security.json` (create this in your project root)

```json
{
  "xsappname": "my-library-app",
  "tenant-mode": "dedicated",
  "description": "Security for my library app",
  
  "scopes": [
    {
      "name": "$XSAPPNAME.Viewer",
      "description": "Can view books"
    },
    {
      "name": "$XSAPPNAME.Admin",
      "description": "Can manage everything"
    }
  ],
  
  "role-templates": [
    {
      "name": "Viewer",
      "description": "Regular user who can view books",
      "scope-references": [
        "$XSAPPNAME.Viewer"
      ]
    },
    {
      "name": "Admin",
      "description": "Administrator with full access",
      "scope-references": [
        "$XSAPPNAME.Viewer",
        "$XSAPPNAME.Admin"
      ]
    }
  ]
}
```

**Let's break this down**:

```json
"scopes": [...]
```
↑ **Scopes** = Individual permissions (like "can open door", "can use computer")

```json
"role-templates": [...]
```
↑ **Role Templates** = Groups of permissions (like "Student" has "can open door" + "can use library")

```json
"$XSAPPNAME"
```
↑ This is automatically replaced with your app name. It's like writing "this app" instead of the actual name.

---

### Step 4: Create the Front Door (AppRouter) 🚪

**What we're doing**: Setting up the entry point that handles login

**Create folder**: `approuter/`

**File 1**: `approuter/package.json`

```json
{
  "name": "my-library-approuter",
  "scripts": {
    "start": "node node_modules/@sap/approuter/approuter.js"
  },
  "dependencies": {
    "@sap/approuter": "^14"
  }
}
```

**Simple explanation**: This installs the "security guard" software

**File 2**: `approuter/xs-app.json`

```json
{
  "authenticationMethod": "route",
  "routes": [
    {
      "source": "^/api/(.*)$",
      "target": "$1",
      "destination": "srv-api",
      "authenticationType": "xsuaa"
    }
  ]
}
```

**What this means**:
- `"source": "^/api/(.*)$"` = "If someone visits /api/anything..."
- `"authenticationType": "xsuaa"` = "...check if they're logged in first"

---

### Step 5: Write Your Business Logic 💼

**What we're doing**: Adding custom checks in your code

**File**: `srv/service.cds`

```cds
using library from '../db/schema';

service LibraryService {
  entity Books as projection on library.Books;
  entity Orders as projection on library.Orders;
  
  // Custom function to get user info
  function getUserInfo() returns String;
}
```

**File**: `srv/service.js`

```javascript
const cds = require('@sap/cds');

module.exports = cds.service.impl(async function() {
  const { Books, Orders } = this.entities;

  // BEFORE creating a book, check if user is Admin
  this.before('CREATE', 'Books', async (req) => {
    // req.user.is('Admin') checks if user has Admin role
    if (!req.user.is('Admin')) {
      // If not, reject the request
      req.reject(403, 'Only admins can add books!');
    }
  });

  // BEFORE updating a book, check if user is Admin
  this.before('UPDATE', 'Books', async (req) => {
    if (!req.user.is('Admin')) {
      req.reject(403, 'Only admins can update books!');
    }
  });

  // Get information about current user
  this.on('getUserInfo', async (req) => {
    return {
      userId: req.user.id,              // Who is this?
      isAdmin: req.user.is('Admin'),    // Are they an admin?
      isViewer: req.user.is('Viewer')   // Are they a viewer?
    };
  });
});
```

**Breaking it down**:

```javascript
this.before('CREATE', 'Books', ...)
```
↑ "Before someone creates a book, do this check"

```javascript
req.user.is('Admin')
```
↑ "Does this user have the Admin role?"

```javascript
req.reject(403, 'message')
```
↑ "Stop them and show this error message"

---

### Step 6: Configure Deployment 🚀

**What we're doing**: Telling SAP BTP how to deploy everything

**File**: `mta.yaml`

```yaml
_schema-version: '3.1'
ID: my-library-app
version: 1.0.0

modules:
  # Your main application
  - name: my-library-srv
    type: nodejs
    path: gen/srv
    requires:
      - name: my-library-auth    # Needs security service
      - name: my-library-db      # Needs database

  # Your front door (AppRouter)
  - name: my-library-approuter
    type: approuter.nodejs
    path: approuter
    requires:
      - name: my-library-auth    # Needs security service

resources:
  # Security service configuration
  - name: my-library-auth
    type: org.cloudfoundry.managed-service
    parameters:
      service: xsuaa
      service-plan: application
      path: ./xs-security.json

  # Database configuration
  - name: my-library-db
    type: com.sap.xs.hdi-container
    parameters:
      service: hana
      service-plan: hdi-shared
```

**Simple explanation**: This is like a recipe that tells SAP BTP:
1. Start the main app
2. Start the front door (AppRouter)
3. Create a security service
4. Create a database
5. Connect them all together

---

### Step 7: Deploy to SAP BTP ☁️

**What we're doing**: Putting your app on the internet!

```bash
# Step 1: Build everything
mbt build

# Step 2: Login to SAP BTP
cf login

# Step 3: Deploy
cf deploy mta_archives/my-library-app_1.0.0.mtar
```

**What happens**:
1. `mbt build` = Packages your app into a deployable file
2. `cf login` = Connects to your SAP BTP account
3. `cf deploy` = Uploads and starts your app

**Wait 5-10 minutes** ⏳ (Deployment takes time!)

---

### Step 8: Give Users Access 👥

**What we're doing**: Assigning roles to real people

#### Via SAP BTP Cockpit (Graphical Interface)

1. Open your **SAP BTP Cockpit**
2. Go to **Security** → **Role Collections**
3. You'll see:
   - `my-library-app-Viewer`
   - `my-library-app-Admin`

4. Click on `my-library-app-Admin`
5. Click **Edit**
6. Under **Users**, add email: `john.doe@example.com`
7. Click **Save**

**Now John Doe is an admin!** 🎉

---

## 🧪 Testing Your Security

### Test 1: Check if Login Works

1. Get your app URL from BTP Cockpit
2. Open it in a browser: `https://your-app.cfapps.eu10.hana.ondemand.com`
3. You should see a **login page**
4. Login with your SAP credentials

**Expected**: You should be redirected after login ✅

---

### Test 2: Check Viewer Access

**Login as a user with Viewer role**

Try to read books:
```bash
GET /api/Books
```
**Expected**: ✅ Works! You can see books

Try to create a book:
```bash
POST /api/Books
{
  "title": "New Book"
}
```
**Expected**: ❌ Error 403 - "Only admins can add books!"

---

### Test 3: Check Admin Access

**Login as a user with Admin role**

Try to create a book:
```bash
POST /api/Books
{
  "title": "Admin's Book"
}
```
**Expected**: ✅ Works! Book is created

---

## 🐛 Common Mistakes & How to Fix Them

### Mistake 1: "I get 401 Unauthorized"

**Problem**: You're not logged in

**Fix**:
```bash
# Check if XSUAA service is running
cf service my-library-auth

# Restart your app
cf restart my-library-approuter
```

---

### Mistake 2: "I get 403 Forbidden"

**Problem**: You're logged in but don't have the right role

**Fix**:
1. Go to BTP Cockpit
2. Check **Security** → **Role Collections**
3. Make sure your user email is added to the correct role
4. **Wait 5 minutes** for changes to take effect
5. Logout and login again

---

### Mistake 3: "Roles don't appear in BTP Cockpit"

**Problem**: xs-security.json wasn't deployed correctly

**Fix**:
```bash
# Update the XSUAA service
cf update-service my-library-auth -c xs-security.json

# Restart your app
cf restart my-library-srv
```

---

### Mistake 4: "AppRouter won't start"

**Problem**: Configuration error in xs-app.json

**Fix**:
```bash
# Check logs for errors
cf logs my-library-approuter --recent

# Common issue: JSON syntax error
# Use a JSON validator: https://jsonlint.com/
```

---

## 🎓 Learning Path

### What You've Learned

✅ What authentication and authorization mean  
✅ How XSUAA works as a security guard  
✅ How to define roles and permissions  
✅ How to protect your data with CDS annotations  
✅ How to add custom security checks in code  
✅ How to deploy a secure app to SAP BTP  

### Next Steps

1. **Add more roles**: Create a "Manager" role with different permissions
2. **Row-level security**: Let users only see their own orders
3. **Audit logging**: Track who did what and when
4. **API testing**: Use Postman to test your endpoints

---

## 📖 Simple Glossary

| Term | What It Means |
|------|---------------|
| **CAP** | Framework for building SAP apps |
| **CDS** | Language for defining data models |
| **XSUAA** | Security service in SAP BTP |
| **AppRouter** | Front door that handles login |
| **JWT Token** | Digital key that proves who you are |
| **Scope** | A single permission |
| **Role** | Collection of permissions |
| **MTA** | Way to package SAP apps for deployment |
| **HDI** | Database container in SAP HANA |

---

## 🆘 Need Help?

### Getting Error Messages?

1. **Read the error carefully** - It usually tells you what's wrong
2. **Check the logs**:
   ```bash
   cf logs my-library-srv --recent
   ```
3. **Google the error message** - Someone else probably had the same issue

### Still Stuck?

- Check SAP Community: https://community.sap.com/
- Read CAP documentation: https://cap.cloud.sap/docs/
- Ask on Stack Overflow with tag `sapui5` or `sap-cap`

---

## 🎯 Quick Reference Card

### Check Your Deployment

```bash
# See all your apps
cf apps

# See all services
cf services

# View logs
cf logs APP-NAME --recent
```

### Common Commands

```bash
# Build your app
mbt build

# Deploy to BTP
cf deploy mta_archives/FILENAME.mtar

# Restart an app
cf restart APP-NAME

# Delete an app
cf delete APP-NAME
```

### File Structure

```
my-library-app/
├── db/
│   └── schema.cds          # Your data model
├── srv/
│   ├── service.cds         # Service definition
│   └── service.js          # Your code
├── approuter/
│   ├── package.json        # AppRouter config
│   └── xs-app.json         # Routing rules
├── xs-security.json        # Security config
├── mta.yaml                # Deployment config
└── package.json            # Main config
```

---

## 💡 Pro Tips for Beginners

1. **Start Simple**: Get basic authentication working first, then add complex rules
2. **Test Locally First**: Use `cds watch` to test before deploying
3. **Use Console.log**: Add `console.log(req.user)` to see what's happening
4. **Read Error Messages**: They're your friends, not enemies!
5. **Version Control**: Use Git to save your work frequently
6. **Ask Questions**: No question is too basic - everyone started where you are

---

## 🎉 Congratulations!

You now understand how to secure a CAP application with XSUAA! 

Remember:
- **Authentication** = Who are you?
- **Authorization** = What can you do?
- **Roles** = Collections of permissions
- **XSUAA** = The security guard

Keep practicing, and soon this will become second nature! 🚀

---

**Made with ❤️ for junior developers**

*Last updated: November 2025*

---

**Last Updated**: November 2025
