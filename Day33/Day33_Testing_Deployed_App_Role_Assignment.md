# Day 33: Testing Deployed Application & Role Assignment

---

## Day Schedule (8 Hours)

| Time | Session | Duration |
|------|---------|----------|
| 09:00 - 09:15 | Day 32 Recap & Quick Fire | 15 min |
| 09:15 - 10:30 | Session 1: Accessing Your Deployed App & BTP Cockpit Navigation | 75 min |
| 10:30 - 10:45 | Break | 15 min |
| 10:45 - 12:00 | Session 2: Role Collections — Creation & Assignment | 75 min |
| 12:00 - 13:00 | Session 3: Hands-on — Create Roles & Assign to Users | 60 min |
| 13:00 - 13:45 | Lunch Break | 45 min |
| 13:45 - 14:45 | Session 4: Application Scaling, Environment Variables & Logs | 60 min |
| 14:45 - 15:00 | Break | 15 min |
| 15:00 - 16:00 | Session 5: Hands-on — Test Authorization & Analyze Logs | 60 min |
| 16:00 - 16:45 | Session 6: End-to-End Deployment Documentation | 45 min |
| 16:45 - 17:00 | Assessment & Assignment | 15 min |

---

## What You'll Learn Today

By the end of this session, you will be able to:
- Access your deployed CAP application through its production URL
- Navigate the BTP Cockpit to manage applications and services
- Create Role Collections in the BTP Cockpit
- Assign Role Collections to users (including your trial user)
- Test real authentication and authorization in the deployed app
- Scale applications (instances and memory) based on needs
- Read and analyze application logs through BTP Cockpit and CLI
- Manage environment variables for deployed applications

---

## Day 32 Recap — Quick Fire (09:00 - 09:15)

1. `mbt build` creates what file? → _____
2. `cf login` authenticates you to _____? → _____
3. `cf deploy *.mtar` does what? → _____
4. Deployment order: _____ → _____ → _____ → _____? → _____
5. `cf logs <app> --recent` shows _____? → _____
6. `VCAP_SERVICES` contains _____? → _____
7. App showing `0/1` instances means _____? → _____
8. Exit code 137 means _____? → _____

<details>
<summary>Answers</summary>

1. An **.mtar archive** (deployable package)
2. **Cloud Foundry** runtime on SAP BTP
3. Creates services, deploys modules, binds everything — **full deployment**
4. **Resources** → **DB Deployer** → **Backend** → **Approuter**
5. Recent application **logs** (stdout, errors, HTTP requests)
6. **Service credentials** (HANA URL/password, XSUAA secrets) as env vars
7. App **crashed** — 0 running instances out of 1 requested
8. **Out of Memory** — app killed by the OS for exceeding RAM limit

</details>

---

## Session 1: Accessing Your Deployed App & BTP Cockpit Navigation (09:15 - 10:30)

### Finding Your Deployed Application

After `cf deploy` succeeds, your application is live. Here's how to find it:

**Method 1: From the Terminal**
```bash
# List all running apps with URLs
cf apps

# Output:
# name                        state    instances  memory  urls
# po-management-srv           started  1/1        256M    po-management-srv.cfapps.us10.hana.ondemand.com
# po-management-approuter     started  1/1        256M    po-management.cfapps.us10.hana.ondemand.com
```

The **approuter URL** is your application's entry point — that's what users open in their browser.

**Method 2: From BTP Cockpit**
1. Open https://cockpit.hanatrial.ondemand.com
2. Navigate: Subaccount → Cloud Foundry → Spaces → your space (e.g., "dev")
3. Click on "Applications" → see all deployed apps with their URLs

---

### BTP Cockpit Navigation Map

```
BTP Cockpit
├── Global Account
│   └── Subaccount (e.g., "trial")
│       ├── Overview (subaccount details, CF API endpoint)
│       ├── Cloud Foundry
│       │   └── Spaces
│       │       └── dev
│       │           ├── Applications          ← Your apps + URLs
│       │           ├── Service Instances      ← HANA, XSUAA
│       │           └── Service Bindings       ← App ↔ Service connections
│       ├── Security
│       │   ├── Role Collections              ← 🆕 Create roles here!
│       │   ├── Users                         ← Assign roles to users
│       │   └── Trust Configuration           ← IdP settings
│       ├── SAP HANA Cloud                    ← HANA instance management
│       └── Connectivity
│           └── Destinations                   ← URL registry
```

---

### Understanding the Application URLs

```
YOUR APPLICATION HAS TWO URLs:

1. APPROUTER URL (for users):
   https://po-management.cfapps.us10.hana.ondemand.com
   │
   ├── This is the ENTRY POINT for users
   ├── Handles login (redirects to XSUAA)
   ├── Routes requests to backend
   └── Users should ONLY use this URL

2. BACKEND URL (internal):
   https://po-management-srv.cfapps.us10.hana.ondemand.com
   │
   ├── Direct access to CAP OData services
   ├── SHOULD require authentication (returns 401 without token)
   ├── Used by approuter internally
   └── Users should NOT use this URL directly
```

---

### Testing Access to Your App

```bash
# Test 1: Open approuter URL in browser
# → Should redirect to login page (XSUAA)
# → After login → see your Fiori app!

# Test 2: Try backend directly (without token)
curl https://po-management-srv.cfapps.us10.hana.ondemand.com/catalog/Products
# Expected: 401 Unauthorized (good! security is working!)

# Test 3: Service index (if not secured)
curl https://po-management-srv.cfapps.us10.hana.ondemand.com/
# May show service endpoints or 401
```

---

### What Users See When Opening the App

```
FIRST TIME (not logged in):
┌──────────────────────────────────────────────────────┐
│                                                      │
│       SAP BTP Identity Authentication                │
│                                                      │
│       Email:    [_________________________]          │
│       Password: [_________________________]          │
│                                                      │
│                    [Log On]                           │
│                                                      │
│  ─── or sign in with ───                             │
│  [Corporate IdP]  [SAP Universal ID]                 │
│                                                      │
└──────────────────────────────────────────────────────┘

AFTER LOGIN (authenticated):
┌──────────────────────────────────────────────────────┐
│  Purchase Order Management                [👤 User]  │
│                                                      │
│  IF user has correct roles:                          │
│    → Sees the Fiori app (List Report, etc.)          │
│                                                      │
│  IF user has NO roles assigned:                      │
│    → May see empty page or 403 Forbidden             │
│    → "You do not have authorization"                 │
│                                                      │
└──────────────────────────────────────────────────────┘
```

**Key point:** Login (authentication) succeeds even WITHOUT roles. But the APP returns 403 because the user has no authorization. This is why role assignment is critical!

---

## Session 2: Role Collections — Creation & Assignment (10:45 - 12:00)

### The Security Chain in Production

```
xs-security.json          BTP Cockpit              BTP Cockpit
(defines scopes &     →   (creates role        →   (assigns role
 role templates)           collections)             collections to users)

DONE IN CODE              DONE IN COCKPIT          DONE IN COCKPIT
(during deployment)       (admin setup)            (admin setup)
```

**Important:** Deploying your app creates the **role templates** in XSUAA. But you must MANUALLY:
1. Create **Role Collections** in BTP Cockpit
2. Assign those collections to **users**

Without this, users can login but can't DO anything!

---

### Step-by-Step: Creating Role Collections

#### Step 1: Navigate to Role Collections

```
BTP Cockpit → Subaccount → Security → Role Collections
```

You'll see a list (possibly empty on first setup):
```
┌─────────────────────────────────────────────────────────┐
│  Role Collections                          [+ Create]   │
├─────────────────────────────────────────────────────────┤
│  Name               │ Description           │ # Roles   │
│  (empty — you need to create these!)                    │
└─────────────────────────────────────────────────────────┘
```

---

#### Step 2: Create a Role Collection

1. Click **"Create"** (or "+" button)
2. Fill in:
   - **Name:** `PO_Manager` (or whatever matches your xs-security.json)
   - **Description:** "Can create and approve Purchase Orders"
3. Click **"Create"**

---

#### Step 3: Add Roles to the Collection

After creating the collection, click on it → click **"Edit"**:

1. In the "Roles" section, click **"Add Role"**
2. You'll see roles from your deployed XSUAA instance:
   ```
   Application: po-management!t12345
   ├── Viewer
   ├── PurchaseManager
   └── Administrator
   ```
3. Select the role template you want (e.g., "PurchaseManager")
4. Click **"Add"**
5. Click **"Save"**

---

#### Step 4: Assign Users to the Role Collection

Still in the Role Collection page:

1. Switch to the **"Users"** tab
2. Click **"Add User"**  (or "Assign User")
3. Enter:
   - **Identity Provider:** Default identity provider
   - **User ID:** your-email@trial.com (your BTP trial email)
4. Click **"Assign"**

---

### Visual Walkthrough

```
┌─────────────────────────────────────────────────────────────┐
│  Role Collection: PO_Manager                                 │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  Tab: [Roles]  [Users]                                      │
│                                                             │
│  ROLES:                                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Application ID      │ Role Template    │ Role         │ │
│  │ po-management!t123  │ PurchaseManager  │ PurchaseM... │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
│  USERS:                                                     │
│  ┌────────────────────────────────────────────────────────┐ │
│  │ Identity Provider       │ User ID                      │ │
│  │ Default (SAP ID)        │ your-email@trial.com         │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

### Role Collections for PO Management App

Create these 3 Role Collections:

| Role Collection Name | Roles Included | Assign To |
|---------------------|---------------|-----------|
| `PO_ViewerRC` | Viewer | Users who only need to browse POs |
| `PO_ManagerRC` | PurchaseManager | Users who create and approve POs |
| `PO_AdminRC` | Administrator | IT admins with full access |

**For trial accounts:** Assign ALL role collections to YOUR user (so you can test everything).

---

### When Do Role Templates Appear in BTP Cockpit?

Role templates appear ONLY after deployment:

```
BEFORE DEPLOY:
  BTP Cockpit → Roles → (empty — nothing from your app)

AFTER DEPLOY (cf deploy creates XSUAA instance):
  BTP Cockpit → Roles → Now shows:
    po-management!t12345.Viewer
    po-management!t12345.PurchaseManager
    po-management!t12345.Administrator
```

**If you don't see your roles:** Check that:
1. `xs-security.json` is correct
2. XSUAA service was created (`cf services` shows it)
3. Your role templates are defined in xs-security.json

---

### Automatic Role Collection Assignment (via xs-security.json)

You can PRE-DEFINE role collections in `xs-security.json` so they're created at deploy time:

```json
{
  "role-collections": [
    {
      "name": "PO_ViewerRC",
      "description": "Read-only access",
      "role-template-references": ["$XSAPPNAME.Viewer"]
    },
    {
      "name": "PO_ManagerRC",
      "description": "Create and approve POs",
      "role-template-references": ["$XSAPPNAME.PurchaseManager"]
    },
    {
      "name": "PO_AdminRC",
      "description": "Full admin access",
      "role-template-references": ["$XSAPPNAME.Administrator"]
    }
  ]
}
```

**With this:** Role Collections are created automatically during deployment! You still need to manually assign USERS to them.

---

### Testing After Role Assignment

```
BEFORE assigning roles:
  User logs in → sees empty app or 403 error
  "Insufficient privileges" or "Forbidden"

AFTER assigning PO_ManagerRC:
  User logs in → sees full app!
  Can create POs ✅
  Can approve POs ✅
  Cannot delete ❌ (not admin)

AFTER assigning PO_AdminRC:
  User can do EVERYTHING
  Delete, manage suppliers, full access ✅
```

---

## Session 3: Hands-on — Create Roles & Assign to Users (12:00 - 13:00)

### Exercise 1: Create Role Collections in BTP Cockpit (20 minutes)

1. **Login** to BTP Cockpit (https://cockpit.hanatrial.ondemand.com)
2. Navigate: **Subaccount → Security → Role Collections**
3. **Create** "PO_ViewerRC":
   - Add role: `Viewer` from your `po-management` application
4. **Create** "PO_ManagerRC":
   - Add role: `PurchaseManager`
5. **Create** "PO_AdminRC":
   - Add role: `Administrator`

---

### Exercise 2: Assign Roles to Your User (10 minutes)

1. Click on **"PO_AdminRC"** (start with admin so you can test everything)
2. Go to **"Users"** tab
3. Click **"Assign User"**
4. Enter YOUR trial email address
5. Click **"Assign"**
6. Repeat for **"PO_ManagerRC"** (assign to same user for testing)

---

### Exercise 3: Test in the Browser (30 minutes)

1. **Open your app URL** (approuter):
   ```
   https://po-management.cfapps.us10.hana.ondemand.com
   ```

2. **Login** with your trial credentials

3. **Verify access:**
   - Can you see the Product list? ✅ (if viewer role is working)
   - Can you create a Purchase Order? ✅ (if manager role is working)
   - Can you access Suppliers? ✅ (if admin role is working)

4. **Test role restriction:**
   - Remove PO_AdminRC from your user (Security → Role Collections → PO_AdminRC → Users → remove)
   - Refresh the app
   - Try to access Suppliers → should get 403!
   - Re-add PO_AdminRC when done testing

---

### Troubleshooting: "I assigned the role but still get 403"

| Issue | Fix |
|-------|-----|
| Role assigned but still 403 | Clear browser cache / use incognito / wait 5 minutes for propagation |
| Can't find my app's roles | Check `cf services` → is XSUAA service created? |
| Role template not showing | Verify xs-security.json has correct role-templates section |
| User ID doesn't match | Use the EXACT email shown in BTP Cockpit → Users section |
| Wrong Identity Provider selected | Select "Default identity provider" (SAP ID Service) |

---

## Session 4: Application Scaling, Environment Variables & Logs (13:45 - 14:45)

### Application Scaling

**Why scale?** When more users access your app, one instance might not handle the load.

```
SINGLE INSTANCE (default):
┌──────────┐
│  App     │ ← All requests go here
│  (256MB) │    If it crashes, app is DOWN!
└──────────┘

MULTIPLE INSTANCES (scaled):
┌──────────┐  ┌──────────┐  ┌──────────┐
│  App #1  │  │  App #2  │  │  App #3  │
│  (256MB) │  │  (256MB) │  │  (256MB) │
└──────────┘  └──────────┘  └──────────┘
      ↑              ↑              ↑
      └──── Load balanced ──────────┘
      Requests spread evenly. If one crashes, others continue!
```

---

### Scaling Commands

```bash
# Scale UP — more instances (horizontal scaling)
cf scale po-management-srv -i 2     # Run 2 instances
cf scale po-management-srv -i 3     # Run 3 instances

# Scale UP — more memory (vertical scaling)
cf scale po-management-srv -m 512M  # Increase RAM to 512MB

# Scale DOWN — fewer instances
cf scale po-management-srv -i 1     # Back to 1 instance

# Check current scaling
cf app po-management-srv
# instances: 2/2  ← Running 2 of 2 requested
# memory: 512M
```

---

### When to Scale

| Symptom | Solution |
|---------|----------|
| App is slow under load | Scale UP instances (more parallelism) |
| App crashes with exit 137 (OOM) | Scale UP memory |
| Trial quota running out | Scale DOWN instances to 1 |
| App rarely used | Keep at 1 instance, low memory |
| Production with many users | 2-3 instances + adequate memory |

**Trial accounts:** Keep at 1 instance with 256M (conserve quota).

---

### Environment Variables

Cloud Foundry passes configuration to your app via environment variables:

```bash
# View ALL environment variables
cf env po-management-srv

# Key sections in the output:
# VCAP_SERVICES → service credentials (HANA, XSUAA)
# VCAP_APPLICATION → app metadata (name, URL, memory)
# User-Provided → your custom variables
```

---

### Setting Custom Environment Variables

```bash
# Set a custom variable
cf set-env po-management-srv LOG_LEVEL debug
cf set-env po-management-srv FEATURE_FLAG_NEW_UI true

# Restart to pick up changes
cf restart po-management-srv

# Remove a variable
cf unset-env po-management-srv FEATURE_FLAG_NEW_UI
cf restart po-management-srv
```

**Use cases for custom env vars:**
- `LOG_LEVEL=debug` → More detailed logging for troubleshooting
- `NODE_ENV=production` → Optimize Node.js for production
- Feature flags → Enable/disable features without redeployment

---

### Viewing Logs in BTP Cockpit

**Method 1: CLI (most common)**
```bash
# Recent logs (last few hundred lines)
cf logs po-management-srv --recent

# Live streaming (watch in real-time)
cf logs po-management-srv
# → Make requests in the browser → see them appear here!
# Press Ctrl+C to stop
```

**Method 2: BTP Cockpit**
1. Navigate: Subaccount → Cloud Foundry → Spaces → dev → Applications
2. Click on your application name
3. Click **"Logs"** tab
4. View recent logs with filtering options

---

### Reading Logs — What Each Line Means

```
2026-06-05T10:30:01.234Z [RTR/0] OUT po-management-srv... "GET /catalog/Products HTTP/1.1" 200
2026-06-05T10:30:01.235Z [APP/PROC/WEB/0] OUT [cds] - serving CatalogService
2026-06-05T10:30:02.100Z [RTR/0] OUT po-management-srv... "POST /purchasing/POs HTTP/1.1" 403
2026-06-05T10:30:05.000Z [APP/PROC/WEB/0] ERR Error: Insufficient privileges
```

| Prefix | Meaning |
|--------|---------|
| `[RTR/0]` | Router — HTTP request/response (URL, status code) |
| `[APP/PROC/WEB/0]` | Your application code (console.log output) |
| `[CELL/0]` | Container lifecycle (start, stop, crash) |
| `[API/0]` | Cloud Foundry API events |
| `OUT` | Standard output (normal logs) |
| `ERR` | Standard error (errors!) |

**Key patterns:**
- `200` in RTR = request succeeded
- `401` in RTR = not authenticated
- `403` in RTR = not authorized (wrong role)
- `500` in RTR = server error (check APP logs for details)
- `ERR` lines = something went wrong (investigate!)

---

### Log Analysis for Common Issues

```bash
# Find all errors
cf logs po-management-srv --recent | grep "ERR"

# Find authorization failures
cf logs po-management-srv --recent | grep "403"

# Find slow requests (> 1 second)
cf logs po-management-srv --recent | grep "RTR" | awk -F'"' '{print $2, $3}'

# Find crashes
cf logs po-management-srv --recent | grep -i "crash\|killed\|error"
```

---

## Session 5: Hands-on — Test Authorization & Analyze Logs (15:00 - 16:00)

### Exercise 1: Test Different Authorization Levels (25 minutes)

**Setup:** Make sure your app is deployed and you have Role Collections created.

---

**Test A: Full Admin Access**
1. Assign `PO_AdminRC` to your user
2. Open the app → login
3. Verify:
   - ✅ Can read Products
   - ✅ Can read Purchase Orders
   - ✅ Can create Purchase Orders
   - ✅ Can access Suppliers
   - ✅ Can delete POs
4. Document: "Admin sees everything"

---

**Test B: Manager Access**
1. Remove `PO_AdminRC` from your user
2. Assign only `PO_ManagerRC`
3. Clear browser cache / open incognito
4. Login again → verify:
   - ✅ Can read Products
   - ✅ Can create POs
   - ✅ Can approve POs
   - ❌ Cannot access Suppliers (403)
   - ❌ Cannot delete (403)
5. Document: "Manager can manage POs but not admin functions"

---

**Test C: Viewer Access**
1. Remove `PO_ManagerRC`, assign only `PO_ViewerRC`
2. Clear browser / incognito
3. Login → verify:
   - ✅ Can read Products
   - ✅ Can read approved POs
   - ❌ Cannot create POs (403)
   - ❌ Cannot edit anything
   - ❌ Cannot access Suppliers
4. Document: "Viewer is strictly read-only"

---

**Test D: No Roles**
1. Remove ALL role collections from your user
2. Clear browser / incognito
3. Login → verify:
   - Login succeeds (authentication works!)
   - ❌ App shows 403 or empty (no authorization!)
4. Document: "Authentication without authorization = useless"

**Remember to RE-ASSIGN PO_AdminRC after testing!**

---

### Exercise 2: Monitor Logs During Testing (15 minutes)

```bash
# Terminal 1: Stream logs live
cf logs po-management-srv

# Terminal 2 (or browser): Make requests to the app
# Watch Terminal 1 — you'll see each request logged!
```

**Things to look for:**
1. Successful requests: `200` status
2. Authorization failures: `403` status → which endpoint? which user?
3. Errors: `ERR` lines → what went wrong?

---

### Exercise 3: Scale and Monitor (20 minutes)

```bash
# Check current state
cf app po-management-srv

# Scale to 2 instances
cf scale po-management-srv -i 2

# Check that both instances are running
cf app po-management-srv
# instances: 2/2

# Make several requests — see them distributed across instances
cf logs po-management-srv --recent | grep "RTR" | tail -10
# Notice: requests go to instance 0 AND instance 1

# Scale back to 1 (save quota)
cf scale po-management-srv -i 1
```

---

## Session 6: End-to-End Deployment Documentation (16:00 - 16:45)

### Exercise: Document Your Deployment Process

Create a deployment runbook that someone else could follow to deploy your app.

**File: `docs/deployment-guide.md`**

```markdown
# PO Management — Deployment Guide

## Prerequisites
- SAP BTP Trial account with Cloud Foundry enabled
- HANA Cloud instance running
- Node.js 18+ installed
- CF CLI installed
- MBT installed (`npm install -g mbt`)

## Step 1: Build
```bash
mbt build
# Creates: mta_archives/po-management_1.0.0.mtar
```

## Step 2: Login to Cloud Foundry
```bash
cf login -a https://api.cf.us10-001.hana.ondemand.com
# Select org and space
```

## Step 3: Ensure HANA is Running
- BTP Cockpit → SAP HANA Cloud → Status: Running ✅

## Step 4: Deploy
```bash
cf deploy mta_archives/po-management_1.0.0.mtar
# Wait 5-10 minutes
```

## Step 5: Verify Deployment
```bash
cf apps      # All apps "started"
cf services  # All services created
```

## Step 6: Create Role Collections
1. BTP Cockpit → Security → Role Collections
2. Create: PO_ViewerRC (add Viewer role)
3. Create: PO_ManagerRC (add PurchaseManager role)
4. Create: PO_AdminRC (add Administrator role)

## Step 7: Assign Roles to Users
1. Role Collection → Users tab → Assign User
2. Assign appropriate role collection to each user

## Step 8: Access the App
- URL: https://po-management.cfapps.us10.hana.ondemand.com
- Login with assigned credentials

## Troubleshooting
- App not starting? → `cf logs po-management-srv --recent`
- 403 after login? → Check role assignment in BTP Cockpit
- Empty data? → Check HANA is running, CSV data loaded
- 502 Bad Gateway? → Backend may have crashed, check logs

## Cleanup
```bash
cf undeploy po-management --delete-services --delete-service-keys
```
```

---

## Assessment: MCQ — 15 Questions on Cloud Testing & Roles

**Q1.** After deployment, users access the application via:

- A) The backend service URL directly
- B) The approuter URL (single entry point with authentication)
- C) The HANA database URL
- D) The BTP Cockpit URL

**Answer: B** — Users always access through the approuter, which handles login and routing.

---

**Q2.** A user can login but sees "403 Forbidden" in the app. The cause is:

- A) Wrong password
- B) User is authenticated but has NO Role Collections assigned (no authorization)
- C) The app is not deployed
- D) The database is empty

**Answer: B** — 403 = authenticated but not authorized. The user needs Role Collections assigned in BTP Cockpit.

---

**Q3.** Role Collections are created in:

- A) xs-security.json only (automatic)
- B) BTP Cockpit → Security → Role Collections (manual or via xs-security.json)
- C) Cloud Foundry CLI
- D) The Fiori app itself

**Answer: B** — Role Collections can be defined in xs-security.json (auto-created at deploy) or created manually in BTP Cockpit.

---

**Q4.** The correct assignment hierarchy is:

- A) Users → Scopes → Roles
- B) Scopes → Role Templates → Role Collections → Users
- C) Users → Applications → Services
- D) Roles → Users → Scopes

**Answer: B** — Scopes (atomic permissions) → Role Templates (bundles) → Role Collections (assigned to users).

---

**Q5.** After assigning a new Role Collection to a user, they might need to:

- A) Restart the server
- B) Clear browser cache / open incognito / wait a few minutes for propagation
- C) Re-deploy the application
- D) Change their password

**Answer: B** — Token propagation may take a few minutes. Clearing cache or incognito window forces a fresh token.

---

**Q6.** `cf scale po-management-srv -i 3` does what?

- A) Sets memory to 3 MB
- B) Runs 3 instances of the application (horizontal scaling for load balancing)
- C) Creates 3 databases
- D) Deploys to 3 spaces

**Answer: B** — `-i 3` = 3 instances. Requests are load-balanced across them.

---

**Q7.** `cf scale po-management-srv -m 512M` does what?

- A) Adds 512 MB disk
- B) Increases the application's RAM allocation to 512 MB (vertical scaling)
- C) Limits download speed
- D) Sets timeout to 512 ms

**Answer: B** — `-m 512M` increases memory. Useful if app crashes with OOM (exit 137).

---

**Q8.** `cf env po-management-srv` shows:

- A) Only user-defined variables
- B) ALL environment variables including VCAP_SERVICES (service credentials)
- C) The source code
- D) The deployment log

**Answer: B** — Shows everything: VCAP_SERVICES, VCAP_APPLICATION, and user-defined vars.

---

**Q9.** `cf set-env po-management-srv LOG_LEVEL debug` requires what after?

- A) Nothing — takes effect immediately
- B) `cf restart po-management-srv` (app must restart to read new env vars)
- C) Redeployment
- D) BTP Cockpit approval

**Answer: B** — Environment variable changes require an app restart to take effect.

---

**Q10.** In application logs, `[RTR/0]` with status `200` means:

- A) An error occurred
- B) A successful HTTP request was routed to the application
- C) The app crashed
- D) A user logged out

**Answer: B** — RTR = Router. 200 = success. This is a normal, successful API request.

---

**Q11.** In logs, `[APP/PROC/WEB/0] ERR Error: Insufficient privileges` indicates:

- A) The app needs more memory
- B) A user tried to access something they don't have the role for (403)
- C) The database is down
- D) The build failed

**Answer: B** — "Insufficient privileges" = authorization failure. User doesn't have the required scope/role.

---

**Q12.** Role Templates from your app appear in BTP Cockpit ONLY after:

- A) Writing xs-security.json
- B) Successful deployment (XSUAA service instance is created during deploy)
- C) Creating a HANA database
- D) Assigning users

**Answer: B** — Role Templates are registered with BTP when the XSUAA service instance is created during `cf deploy`.

---

**Q13.** To remove everything deployed (apps + services) in one command:

- A) `cf delete-all`
- B) `cf undeploy po-management --delete-services --delete-service-keys`
- C) `cf reset`
- D) `cf logout`

**Answer: B** — `cf undeploy` with `--delete-services` removes the entire MTA (apps + bound services).

---

**Q14.** The `VCAP_SERVICES` environment variable is:

- A) Something you write manually
- B) Automatically injected by Cloud Foundry with credentials for bound services
- C) A configuration file
- D) A BTP Cockpit setting

**Answer: B** — Cloud Foundry auto-injects VCAP_SERVICES with credentials when services are bound to apps.

---

**Q15.** If a user reports "I can login but can't create POs", you should check:

- A) If HANA is running
- B) If the user has a Role Collection that includes the "Create" permission (PO_ManagerRC or PO_AdminRC)
- C) If the CSS is loaded
- D) If the user's browser is updated

**Answer: B** — Login works (authentication OK), but can't create = missing authorization. Check role assignment.

---

## Assignment: Deploy PO Management End-to-End & Document

### Task

Deploy your complete PO Management application to SAP BTP and document the entire process with evidence.

### Deliverables

**1. Deployment Evidence**

| Step | Command / Action | Result / Screenshot |
|------|-----------------|-------------------|
| Build | `mbt build` | .mtar file created (show file size) |
| Login | `cf login` | Show target org/space |
| Deploy | `cf deploy *.mtar` | Show success message |
| Apps running | `cf apps` | Show all apps started |
| Services created | `cf services` | Show HANA + XSUAA |
| App URL | `cf app <approuter>` | Show the route URL |

**2. Role Setup Evidence**

| Role Collection | Roles Included | Users Assigned |
|----------------|---------------|----------------|
| PO_ViewerRC | Viewer | (your email) |
| PO_ManagerRC | PurchaseManager | (your email) |
| PO_AdminRC | Administrator | (your email) |

**3. Authorization Test Results**

| Test Case | Role | Action Tested | Expected | Actual |
|-----------|------|---------------|----------|--------|
| 1 | Admin | Read Products | 200 ✅ | |
| 2 | Admin | Create PO | 201 ✅ | |
| 3 | Admin | Delete PO | 204 ✅ | |
| 4 | Admin | Access Suppliers | 200 ✅ | |
| 5 | Manager | Read Products | 200 ✅ | |
| 6 | Manager | Create PO | 201 ✅ | |
| 7 | Manager | Delete PO | 403 ❌ | |
| 8 | Manager | Access Suppliers | 403 ❌ | |
| 9 | Viewer | Read Products | 200 ✅ | |
| 10 | Viewer | Create PO | 403 ❌ | |
| 11 | No role | Any action | 403 ❌ | |
| 12 | No login | Any action | 401 ❌ | |

**4. Deployment Runbook** (see Session 6 template)

---

## End of Day Summary

### What We Learned

| Topic | Key Takeaway |
|-------|-------------|
| App URL | Approuter URL is the user-facing entry point |
| BTP Cockpit | Navigate: Security → Role Collections for role management |
| Role Collections | Created in Cockpit (or xs-security.json), assigned to users |
| Role assignment | Users get Role Collections → which contain Role Templates → which contain Scopes |
| Testing roles | Login works without roles (auth OK), but app blocks access (403) without proper roles |
| Scaling | `-i N` for instances, `-m XM` for memory |
| Environment vars | `cf set-env` + `cf restart` to apply changes |
| Logs | `cf logs --recent` for debugging, RTR for requests, ERR for errors |
| 401 vs 403 | 401 = no login; 403 = logged in but wrong role |
| Cleanup | `cf undeploy --delete-services` removes everything |

### The Production Readiness Checklist

```
□ App deployed and all modules running (cf apps → all "started")
□ HANA connected and tables created (data visible)
□ XSUAA configured and working (login redirects correctly)
□ Role Collections created (Viewer, Manager, Admin)
□ Roles assigned to at least one test user
□ Authorization tested: correct access for each role
□ Authorization tested: denied access for missing roles
□ Logs show no errors during normal usage
□ App is stable (not crashing/restarting)
□ Documentation written (deployment guide)
```

### The One-Liner

> **Deployment is only half the job — assigning Role Collections in BTP Cockpit is what makes your security actually WORK for real users.**

---

### Looking Ahead: Day 34

Tomorrow: **Week 7 Review & Integration Practice**
- Complete review of Weeks 6-7 (Git, HANA, Auth, Deployment)
- Weekly quiz
- Mini project: Full deployment pipeline from scratch

---

*End of Day 33 — Your app is deployed, secured, and tested with real authentication! 🎉*
