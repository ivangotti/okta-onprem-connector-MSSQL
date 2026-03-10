# Okta On-Premise Provisioning to Microsoft SQL Server

> **Warning**
> This is experimental code intended for demonstration and learning purposes. Use at your own risk in production environments.

A comprehensive solution for bidirectional user provisioning between **Okta** and an on-premise **Microsoft SQL Server** database using the SCIM (System for Cross-domain Identity Management) protocol.

---

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Installation Guide](#installation-guide)
  - [Part A: SQL Server Setup](#part-a-sql-server-setup)
  - [Part B: SCIM Gateway Configuration](#part-b-scim-gateway-configuration)
  - [Part C: Okta OPP Agent Installation](#part-c-okta-opp-agent-installation)
  - [Part D: Okta Application Configuration](#part-d-okta-application-configuration)
- [User Attribute Mapping](#user-attribute-mapping)
- [Troubleshooting](#troubleshooting)
- [Author](#author)

---

## Overview

**Okta On-Premise Provisioning (OPP)** enables organizations to extend Okta's identity lifecycle management capabilities to on-premise applications that cannot be reached directly from the cloud. This project demonstrates how to:

- Provision users from Okta to a SQL Server database
- Import existing users from SQL Server into Okta
- Synchronize user attributes bidirectionally
- Manage user lifecycle (create, update, deactivate, delete)

### How It Works

The Okta On-Premise Provisioning Agent acts as a secure bridge between Okta's cloud service and your internal network. It establishes an outbound connection from your network to Okta, eliminating the need to open inbound firewall ports.

```
┌─────────────────┐     HTTPS      ┌─────────────────┐     SCIM/HTTPS     ┌─────────────────┐
│                 │ ◄────────────► │                 │ ◄────────────────► │                 │
│   Okta Cloud    │                │   OPP Agent     │                    │  SCIM Gateway   │
│                 │                │  (On-Premise)   │                    │   (Node.js)     │
└─────────────────┘                └─────────────────┘                    └────────┬────────┘
                                                                                   │
                                                                              SQL/TDS
                                                                                   │
                                                                                   ▼
                                                                         ┌─────────────────┐
                                                                         │                 │
                                                                         │  MS SQL Server  │
                                                                         │   (Database)    │
                                                                         └─────────────────┘
```

---

## Architecture

| Component | Description |
|-----------|-------------|
| **Okta Cloud** | Identity provider that manages users and provisioning policies |
| **OPP Agent** | Windows service that securely connects Okta to on-premise resources |
| **SCIM Gateway** | Node.js application that translates SCIM requests to SQL operations |
| **MS SQL Server** | On-premise database storing user accounts |

---

## Features

- **Full CRUD Operations**: Create, Read, Update, and Delete users
- **Bidirectional Sync**: Import users from SQL to Okta and provision from Okta to SQL
- **Password Synchronization**: Sync passwords from Okta to the SQL database
- **Secure Communication**: TLS/HTTPS encryption for all data in transit
- **SCIM 1.1 Compliance**: Standards-based identity management protocol
- **Attribute Mapping**: Flexible mapping between Okta and SQL user attributes

---

## Prerequisites

- Windows Server 2019/2022
- Microsoft SQL Server 2019/2022
- Node.js (v16 or higher)
- Java JDK (for certificate import)
- Valid SSL/TLS certificate for your domain
- Okta tenant with Lifecycle Management capabilities

---

## Installation Guide

### Part A: SQL Server Setup

#### 1. Provision Windows Server with SQL Server

**Option 1 - AWS EC2:**
- Launch an EC2 instance using the AMI: `Microsoft Windows Server 2022 with SQL Server 2022 Standard`
- Instance type: `t3.xlarge` (minimum recommended)

**Option 2 - Manual Installation:**
- Install Windows Server and download/install MS SQL Server

#### 2. Create the User Database

1. Open **Microsoft SQL Server Management Studio**
2. Connect using your server credentials
3. Right-click **Databases** → **New Database** → Name it (e.g., `dbo`)
4. Execute the following SQL to create the User table:

```sql
CREATE TABLE [dbo].[User](
    UserID varchar(50) NOT NULL,
    Enabled varchar(50) NULL,
    Password varchar(50) NULL,
    FirstName varchar(50) NULL,
    MiddleName varchar(50) NULL,
    LastName varchar(50) NULL,
    Email varchar(50) NULL,
    MobilePhone varchar(50) NULL
);

-- Optional: Insert sample data
INSERT INTO [User] (UserID, Enabled, Password, FirstName, MiddleName, LastName, Email, MobilePhone)
VALUES
    ('julia.doe@example.com', 'true', '********', 'Julia', 'R.', 'Doe', 'julia.doe@example.com', '123-456-789'),
    ('john.smith@example.com', 'true', '********', 'John', 'W.', 'Smith', 'john.smith@example.com', '789-456-123');
```

#### 3. Enable SQL Server Authentication

1. In Object Explorer, right-click your server → **Properties**
2. Navigate to **Security** → Select **SQL Server and Windows Authentication mode**
3. Click **OK**
4. Open Windows **Services** → Restart **SQL Server (MSSQLSERVER)**

#### 4. Configure the SA Account

1. In Object Explorer: **Security** → **Logins** → Double-click **sa**
2. **General tab**: Set a password, uncheck "Enforce password policy"
3. **Status tab**: Set Login to **Enabled**
4. Click **OK**

---

### Part B: SCIM Gateway Configuration

#### 1. Install Node.js

Download and install [Node.js](https://nodejs.org/) (LTS version recommended)

#### 2. Install SCIM Gateway

```bash
npm install scimgateway
```

Follow the [official installation guide](https://github.com/jelhub/scimgateway) for detailed setup.

#### 3. Configure the Entry Point

Edit `index.js` to enable only the MSSQL plugin:

```javascript
#!/usr/bin/env node

// const loki = require('./lib/plugin-loki')
// const mongodb = require('./lib/plugin-mongodb')
// const scim = require('./lib/plugin-scim')
// const forwardinc = require('./lib/plugin-forwardinc')
const mssql = require('./lib/plugin-mssql')
// const saphana = require('./lib/plugin-saphana')
// const azureAD = require('./lib/plugin-azure-ad')
// const ldap = require('./lib/plugin-ldap')
// const api = require('./lib/plugin-api')
```

#### 4. Configure the MSSQL Plugin

Edit `config/plugin-mssql.json`:

```json
{
  "scimgateway": {
    "port": 443,
    "scim": {
      "version": "1.1",
      "customSchema": "customSchema.json"
    }
  },
  "endpoint": {
    "connection": {
      "server": "127.0.0.1",
      "authentication": {
        "type": "default",
        "options": {
          "userName": "sa",
          "password": "your-password"
        }
      },
      "options": {
        "port": 1433,
        "database": "dbo"
      }
    }
  }
}
```

#### 5. Configure SSL Certificates

1. Obtain a valid SSL certificate for your domain (e.g., using [Certbot](https://certbot.eff.org/) or [ZeroSSL](https://zerossl.com/))
2. Place `certificate.pem` and `private.pem` in `config/certs/`
3. Update the certificate paths in `plugin-mssql.json`:

```json
"certificate": {
  "key": "private.pem",
  "cert": "certificate.pem"
}
```

#### 6. Configure DNS

Create an A record pointing your domain (e.g., `scim.yourdomain.com`) to your server's IP address.

#### 7. Start the SCIM Gateway

```bash
node index.js
```

**Expected output:** `now listening on TLS port 443`

#### 8. Verify the Installation

Test these endpoints in your browser:

| Endpoint | Expected Response |
|----------|-------------------|
| `https://scim.yourdomain.com/ping` | `hello` |
| `https://scim.yourdomain.com/ServiceProviderConfig` | SCIM configuration JSON |
| `https://scim.yourdomain.com/Users` | List of users from SQL database |

Default credentials: `gwadmin` / `password`

---

### Part C: Okta OPP Agent Installation

#### 1. Install the Okta On-Premise Provisioning Agent

Follow Okta's [official deployment guide](https://help.okta.com/en-us/content/topics/provisioning/opp/opp-main.htm) to download and install the agent.

#### 2. Install Java JDK

Download and install [Java JDK](https://www.oracle.com/java/technologies/downloads/) (Windows x64 Installer)

#### 3. Import the SSL Certificate

The OPP Agent needs to trust your SCIM server's certificate:

```cmd
cd "C:\Program Files\Java\jdk-20\bin"
keytool -importcert -file "C:\path\to\scimgateway\config\certs\certificate.pem" -keystore "C:\Program Files\Okta\On-Premises Provisioning Agent\current\jre\lib\security\cacerts"
```

> **Note:** Default keystore password is `changeit`

---

### Part D: Okta Application Configuration

#### 1. Create the Application in Okta

1. Navigate to **Applications** → **Create App Integration**
2. Select **SWA (Secure Web Authentication)**
3. Configure:
   - **App name**: Your SQL application name
   - **Login URL**: `https://www.placeholder.com` (placeholder)
4. Click **Finish**

#### 2. Enable On-Premise Provisioning

1. Go to **General** tab → **App Settings** → **Edit**
2. Set **Provisioning** to **On-Premises Provisioning**
3. Save

#### 3. Configure the SCIM Connector

1. Go to **Provisioning** tab → **Configure SCIM Connector**
2. Enter your SCIM server details:
   - **SCIM connector base URL**: `https://scim.yourdomain.com`
   - **Authentication mode**: Basic Auth
   - **Username/Password**: Your SCIM Gateway credentials

3. Click **Test Connector Configuration** to verify connectivity

#### 4. Configure Provisioning Settings

**To App Settings:**
- Enable: Create Users, Update User Attributes, Deactivate Users, Sync Password

**Profile Mapping (Okta User → Application):**
- Configure attribute mappings as needed
- Set email mapping to "Do not map" if encountering sync issues

#### 5. Plugin Customization (Optional)

If you encounter attribute mapping issues, edit `lib/plugin-mssql.js` and add these to `validScimAttr`:

```javascript
const validScimAttr = [
  'userName',
  'active',
  'password',
  'name.givenName',
  'name.middleName',
  'name.familyName',
  'name.formatted',
  'emails',
  'emails.work',
  'phoneNumbers',
  'phoneNumbers.work'
]
```

Restart the SCIM Gateway after making changes.

---

## User Attribute Mapping

| Okta Attribute | SCIM Attribute | SQL Column |
|----------------|----------------|------------|
| User name | `userName` | UserID |
| Status | `active` | Enabled |
| Password | `password` | Password |
| First Name | `name.givenName` | FirstName |
| Middle Name | `name.middleName` | MiddleName |
| Last Name | `name.familyName` | LastName |
| Email | `emails.work` | Email |
| Phone | `phoneNumbers.work` | MobilePhone |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Connection timeout | Increase timeout in Okta Provisioning settings |
| Certificate errors | Ensure certificate is imported into OPP Agent keystore |
| User sync fails | Check SCIM Gateway logs in `logs/plugin-mssql.log` |
| Authentication fails | Verify credentials in `plugin-mssql.json` |

---

## Author

**Ivan Gotti**

- GitHub: [@ivangotti](https://github.com/ivangotti)

---

## License

This project is provided as-is for educational and demonstration purposes.
