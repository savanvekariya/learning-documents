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

---

**Last Updated**: November 2025
