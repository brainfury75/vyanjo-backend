# Vyanjo Backend - Meal + Curry Subscription System Implementation Plan

## Overview
Building a complete Node.js backend (JavaScript only, no TypeScript) for a meal and curry subscription service with Firebase phone authentication, PostgreSQL database, and comprehensive business logic for subscriptions, curry tokens, meal scheduling, pausing, upgrades, and delivery management.

## Current State
**Repository**: `/workspace/cmjpq7e8h02hqimr37qy606bs/vyanjo-backend`
**Status**: Greenfield project - empty repository with only .gitignore and README

## Tech Stack
- **Runtime**: Node.js v18+
- **Language**: JavaScript (ES6+) - **NO TypeScript**
- **Framework**: Express.js
- **Database**: PostgreSQL 14+
- **ORM**: Prisma (for migrations and schema management)
- **Authentication**: Firebase Admin SDK (phone auth token verification)
- **Validation**: express-validator
- **Timezone**: moment-timezone (Asia/Kolkata - IST)
- **CORS**: cors middleware

---

## Project Structure

```
vyanjo-backend/
├── src/
│   ├── controllers/       # Request/response handling
│   ├── services/          # Business logic
│   ├── routes/            # Route definitions
│   ├── middleware/        # Auth, error handling, validation
│   ├── utils/             # Helpers, timezone utilities
│   ├── config/            # Configuration (Firebase, database)
│   └── index.js           # Entry point
├── prisma/
│   ├── schema.prisma      # Database schema
│   ├── migrations/        # Migration files
│   └── seed.js            # Initial data seeding
├── package.json
├── .env.example
├── .env
├── .gitignore
└── README.md
```

**Architecture**: Layered architecture with clear separation of concerns
- **Controllers**: Handle HTTP request/response, validation
- **Services**: Business logic, database operations
- **Routes**: Route definitions with middleware
- **Middleware**: Auth, validation, error handling
- **Utils**: Reusable helpers (timezone, formatters)
- **Config**: Environment-specific configuration

---

## Database Schema

### Complete Prisma Schema

**File**: `prisma/schema.prisma`

The schema defines 17 tables for the complete meal and curry subscription system:

1. **users** - Customer accounts (Firebase phone auth)
2. **common_points** - Shared delivery locations (hostels, PGs, offices)
3. **addresses** - User delivery addresses
4. **meal_packages** - Purchasable subscription plans
5. **subscriptions** - User's active/historical subscriptions
6. **weekly_menus** - Weekly menu per diet & cuisine
7. **weekly_menu_items** - Menu items
8. **delivery_time_slots** - Reusable delivery windows
9. **delivery_groups** - Group multiple items for single delivery
10. **subscription_meals** - Generated meal schedule
11. **meal_pauses** - Audit log for paused meals
12. **curry_token_packages** - Purchasable curry token bundles
13. **user_curry_wallets** - User's curry token balance
14. **curry_orders** - Curry orders using tokens
15. **upgrade_prices** - Upgrade pricing rules
16. **subscription_upgrades** - Applied temporary upgrades
17. **notifications** - In-app notifications

**Complete schema**: Use the exact Prisma schema provided in the user's specification (17 models with all relations, constraints, and field types). This schema is production-ready and should be implemented as-is in `prisma/schema.prisma`.

### Database Connection

**Environment variable**: `DATABASE_URL`
**Format**: `postgresql://user:password@host:5432/database?schema=public`
**Example**: `postgresql://vyanjo_user:password@localhost:5432/vyanjo_db?schema=public`

### Migration Strategy
- Use Prisma Migrate for all schema changes
- Initial migration: `npx prisma migrate dev --name init`
- Subsequent changes: `npx prisma migrate dev --name describe_change`
- Production: `npx prisma migrate deploy`

### Data Integrity Rules

**Enforced at Database Level**:
- Unique constraint: `(diet_type, cuisine_type, week_start_date)` on `weekly_menus`
- Unique constraint: `(user_id, diet_type)` on `user_curry_wallets`
- Unique constraint: `(subscription_id, service_date, item_type)` on `subscription_meals`
- Partial unique index: `(user_id)` WHERE `status = 'active'` on `subscriptions` (one active subscription per user)

**Enforced at Application Level**:
- One primary address per user (checked in service layer)
- First address automatically becomes primary
- When setting `is_primary = true`, unset others

### Enum Values (Application-Level Validation)

```javascript
// File: src/utils/constants.js
const DIET_TYPES = ['veg', 'non_veg'];
const CUISINE_TYPES = ['south_indian', 'north_indian'];
const CONTAINER_TYPES = ['plastic', 'steel'];
const SUBSCRIPTION_STATUS = ['active', 'completed', 'cancelled'];
const MEAL_TYPES = ['breakfast', 'lunch', 'dinner', 'snacks'];
const CURRY_ITEM_TYPES = ['curry_lunch', 'curry_dinner'];
const CURRY_ORDER_STATUS = ['ordered', 'cancelled', 'fulfilled'];
const UPGRADE_TYPES = ['veg_to_nonveg', 'south_to_north'];
const UPGRADE_SCOPES = ['meal', 'day', 'week'];
const ADDRESS_TAGS = ['home', 'work', 'office', 'other'];
const PAUSE_MEAL_TYPES = ['breakfast', 'lunch', 'dinner', 'snacks', 'all'];

module.exports = {
  DIET_TYPES,
  CUISINE_TYPES,
  CONTAINER_TYPES,
  SUBSCRIPTION_STATUS,
  MEAL_TYPES,
  CURRY_ITEM_TYPES,
  CURRY_ORDER_STATUS,
  UPGRADE_TYPES,
  UPGRADE_SCOPES,
  ADDRESS_TAGS,
  PAUSE_MEAL_TYPES
};
```

---

## Firebase Authentication Setup

### Firebase Admin SDK Configuration

**File**: `src/config/firebase.js`

**Required Environment Variables**:
- `FIREBASE_PROJECT_ID`: Your Firebase project ID
- `FIREBASE_PRIVATE_KEY`: Private key from service account JSON (can be base64 encoded or direct)
- `FIREBASE_CLIENT_EMAIL`: Service account email

**Implementation**:
```javascript
const admin = require('firebase-admin');

// Initialize Firebase Admin SDK
admin.initializeApp({
  credential: admin.credential.cert({
    projectId: process.env.FIREBASE_PROJECT_ID,
    privateKey: process.env.FIREBASE_PRIVATE_KEY?.replace(/\\n/g, '\n'),
    clientEmail: process.env.FIREBASE_CLIENT_EMAIL,
  }),
});

module.exports = admin;
```

**Setup Instructions**:
1. Go to Firebase Console → Project Settings → Service Accounts
2. Click "Generate New Private Key"
3. Download JSON file
4. Extract `project_id`, `private_key`, `client_email` to `.env`

### Authentication Middleware

**File**: `src/middleware/auth.js`

**Purpose**: Verify Firebase ID token on every protected route and attach authenticated user to request

**Behavior**:
1. Extract token from `Authorization` header (format: `Bearer <token>`)
2. If no token: Return 401 with `{"error": {"message": "No authentication token provided", "code": "NO_TOKEN", "status": 401}}`
3. Verify token using `admin.auth().verifyIdToken(token)`
4. If verification fails: Return 401 with `{"error": {"message": "Invalid or expired token", "code": "INVALID_TOKEN", "status": 401}}`
5. Extract `phone_number` from decoded token
6. Query database for user with matching `phone_number`
7. If user doesn't exist: Create new user record with phone number
8. Attach user object to `req.user` with fields: `{id, phoneNumber, name}`
9. Call `next()` to proceed to route handler

**Error Handling**:
- Firebase token verification errors: Return 401
- Database errors: Return 500
- Catch all errors and format consistently

**Usage**: Applied to all routes except `GET /health`

**Express Request Extension**:
Attach `user` property to Express request object:
```javascript
// After verification
req.user = {
  id: user.id,
  phoneNumber: user.phoneNumber,
  name: user.name
};
```

---

## Timezone Handling (Critical for 8 PM Pause Deadline)

**Library**: moment-timezone
**Timezone**: Asia/Kolkata (IST)

### Timezone Utility Functions

**File**: `src/utils/timezone.js`

**Functions to implement**:

```javascript
const moment = require('moment-timezone');
const TIMEZONE = 'Asia/Kolkata';

// Get current time in IST
function getCurrentISTTime() {
  return moment().tz(TIMEZONE);
}

// Get current date in IST (YYYY-MM-DD)
function getTodayIST() {
  return moment().tz(TIMEZONE).format('YYYY-MM-DD');
}

// Get tomorrow's date in IST
function getTomorrowIST() {
  return moment().tz(TIMEZONE).add(1, 'day').format('YYYY-MM-DD');
}

// Check if current time is before 8 PM IST (for pause deadline)
function isBeforePauseDeadline() {
  const now = moment().tz(TIMEZONE);
  return now.hour() < 20; // Before 8:00 PM
}

// Convert any date to IST
function convertToIST(date) {
  return moment(date).tz(TIMEZONE);
}

// Calculate week start date (Monday) for given date
function calculateWeekStart(date) {
  return moment(date).tz(TIMEZONE).startOf('isoWeek').format('YYYY-MM-DD');
}

module.exports = {
  getCurrentISTTime,
  getTodayIST,
  getTomorrowIST,
  isBeforePauseDeadline,
  convertToIST,
  calculateWeekStart,
  TIMEZONE
};
```

**Usage Notes**:
- All date comparisons for business logic MUST use IST
- Database stores timestamps in UTC (PostgreSQL handles this automatically with TIMESTAMPTZ)
- API accepts dates in ISO format (YYYY-MM-DD)
- Critical for 8 PM pause deadline enforcement in pause meal endpoint

---

## Core Business Logic Rules

### 1. On-Demand Meal Generation

**Strategy**: Generate meals only when needed, not upfront

**When meals are generated**:
- User calls `GET /api/meals/schedule` (today + tomorrow)
- User attempts to pause/unpause a meal
- System needs to check meal existence

**How it works**:
1. Check if `subscription_meals` exist for requested date
2. If not exist, generate them:
   - Get subscription's meal package
   - For each included meal type (breakfast/lunch/dinner/snacks):
     - Create `subscription_meals` row with `service_date`, `item_type`
     - Assign default `delivery_slot_id` based on meal type
     - Set `is_paused = false`
3. Return generated meals

**Default Delivery Slot Assignments**:
- `breakfast` → Morning Slot (7:00-8:00 AM)
- `lunch` → Afternoon Slot (12:00-1:00 PM)
- `snacks` → Evening Snack Slot (5:00-6:00 PM)
- `dinner` → Dinner Slot (7:00-8:00 PM)

**Benefits**:
- No upfront computation overhead
- Only generates what's needed
- Flexible for future changes
- Works well with 48-hour scheduling window

**Implementation Location**: `src/services/mealService.js` - `generateMealsIfNeeded()` function

### 2. 48-Hour Scheduling Window (Today + Tomorrow Only)

**Rule**: Users can only view and manage meals for today and tomorrow

**Rationale**:
- Works seamlessly with on-demand generation
- Prevents excessive advance planning
- Simplifies system logic
- Meals beyond tomorrow don't exist yet

**Enforcement**:
- `GET /api/meals/schedule`: Returns only today and tomorrow
- `POST /api/meals/pause`: Only allows `date = today OR tomorrow`
- `POST /api/meals/unpause`: Only allows `date = today OR tomorrow`
- `POST /api/curry/order`: Only allows `orderDate = today OR tomorrow`

**Error Response**: If user tries to access meals beyond tomorrow:
`422 Unprocessable Entity` - `"You can only manage meals for today and tomorrow"`

### 3. 8 PM IST Pause Deadline

**Rule**: Users cannot pause/unpause meals for TODAY after 8:00 PM IST

**Enforcement**:
- Check current time in IST
- If current time >= 20:00 (8 PM) AND meal date is today:
  - Return 422: `"Cannot pause meals after 8 PM. Please pause before 8 PM IST."`
- Tomorrow's meals can be paused anytime

**Why Tomorrow is Exempt**: Tomorrow's meals haven't entered preparation yet

**Implementation**: Use `isBeforePauseDeadline()` from timezone utils

### 4. One Active Subscription Per User

**Enforcement at Two Levels**:

**Database Level** (backup protection):
- Partial unique index on `subscriptions(user_id)` WHERE `status = 'active'`
- Prisma schema: `@@unique([userId], map: "unique_active_subscription")`

**Application Level** (primary check):
- Before creating subscription, query for existing active subscription
- If found: Return 422 `"You already have an active subscription"`

**When subscription ends**:
- Status changed from 'active' to 'completed' or 'cancelled'
- User can then create new subscription

### 5. Primary Address Management

**Rule**: Each user has exactly one primary address

**Enforcement** (application-level only):
1. First address created for user: Automatically set `is_primary = true`
2. When creating/updating address with `is_primary = true`:
   - Update all other user addresses: Set `is_primary = false`
   - Then set this address: `is_primary = true`
3. If user has only one address, it must remain primary (enforce in delete logic)

**Delete Protection**: Cannot delete address if:
- It's the only address (user must have at least one)
- It's used in an active subscription

### 6. Curry Token Wallet Management

**Wallet Creation/Update**:
- One wallet per user per diet type (`unique(user_id, diet_type)`)
- Purchase package → Add tokens to `total_tokens`, extend `valid_until`
- Place order → Increment `used_tokens` by 1
- Cancel order → Decrement `used_tokens` by 1 (refund)

**Token Calculation**:
- `remaining_tokens = total_tokens - used_tokens`
- Check `valid_until >= current_date` before allowing orders

**Expiry Handling**:
- If `valid_until < current_date`: Return 422 `"Your curry tokens have expired"`
- Do not allow orders with expired wallets

### 7. Delivery Grouping

**Purpose**: Deliver multiple items together (e.g., tiffin + curry)

**How it works**:
1. User selects multiple meals/orders for same date
2. System creates `delivery_groups` record
3. All items assigned same `delivery_group_id` and `delivery_slot_id`
4. Delivered together in single trip

**Validation**:
- All items must be for same `service_date`
- Minimum 2 items required
- None can be paused
- All must belong to user

**Ungrouping**:
- Remove `delivery_group_id` from all items
- Keep their current `delivery_slot_id` (don't reset to default)
- Delete delivery group record

### 8. Temporary Upgrades

**Types**:
- `veg_to_nonveg`: Upgrade vegetarian meal to non-vegetarian
- `south_to_north`: Change from South Indian to North Indian cuisine

**Scopes**:
- `meal`: Single meal type for date range (e.g., only lunch)
- `day`: All meals for date range
- `week`: All meals for full weeks in date range

**Validation**:
- Check subscription's meal package allows the upgrade:
  - `veg_to_nonveg` requires `allows_diet_upgrade = true`
  - `south_to_north` requires `allows_cuisine_upgrade = true`
- Date range must be within subscription period
- Fetch price from `upgrade_prices` table matching type, scope, meal type

**Pricing Calculation**:
- `meal` scope: `price × number of days`
- `day` scope: `price × number of days`
- `week` scope: `price × number of weeks`

**Removal**:
- Can only remove upgrade if `start_date > current_date` (hasn't started yet)
- Once started, upgrade cannot be removed

---

## API Endpoints Specification

### API Response Format Standards

**Success Response Pattern**:
```javascript
{
  "data": { /* response payload */ },
  "message": "Operation successful" // optional
}
```

**Error Response Pattern**:
```javascript
{
  "error": {
    "message": "Human-readable error message",
    "code": "ERROR_CODE",
    "status": 400
  }
}
```

**Implementation**: Create `src/utils/responseFormatter.js` with helper functions:
```javascript
function successResponse(data, message = null) {
  const response = { data };
  if (message) response.message = message;
  return response;
}

function errorResponse(message, code, status) {
  return {
    error: { message, code, status }
  };
}

module.exports = { successResponse, errorResponse };
```

### Common HTTP Status Codes Used

- **200 OK**: Successful GET, PUT, DELETE
- **201 Created**: Successful POST (resource created)
- **400 Bad Request**: Validation errors
- **401 Unauthorized**: Authentication required/failed
- **403 Forbidden**: User doesn't have permission
- **404 Not Found**: Resource doesn't exist
- **422 Unprocessable Entity**: Business logic error
- **500 Internal Server Error**: Server/database error

---

## API Modules and Endpoints

### Module 1: Authentication

**Base Path**: `/api/auth`
**File**: `src/routes/authRoutes.js`
**Controller**: `src/controllers/authController.js`
**Service**: `src/services/authService.js`

#### POST /api/auth/login

**Purpose**: Sync Firebase-authenticated user with backend database

**Authentication**: Required (Firebase token in header)

**Request Body**: None (user identified from token)

**Business Logic**:
1. Middleware extracts phone number from verified Firebase token
2. Query database for user with matching phone_number
3. If not exists: Create new user record
4. Return user profile

**Response (200)**:
```json
{
  "data": {
    "user": {
      "id": "uuid",
      "phoneNumber": "+919876543210",
      "name": null,
      "createdAt": "2025-01-15T10:30:00Z",
      "updatedAt": "2025-01-15T10:30:00Z"
    }
  }
}
```

**Errors**:
- 401 if token invalid/expired (handled by auth middleware)

#### PUT /api/auth/profile

**Purpose**: Update user's name

**Authentication**: Required

**Request Body**:
```json
{
  "name": "John Doe"
}
```

**Validation** (use express-validator):
- `name`: Required, string, 2-100 characters, trim whitespace
- Validation location: `src/middleware/validators/authValidators.js`

**Business Logic**:
1. Get user_id from req.user (attached by auth middleware)
2. Update user record with new name
3. Return updated user

**Response (200)**:
```json
{
  "data": {
    "user": {
      "id": "uuid",
      "phoneNumber": "+919876543210",
      "name": "John Doe",
      "updatedAt": "2025-01-15T11:00:00Z"
    }
  },
  "message": "Profile updated successfully"
}
```

**Errors**:
- 400 if validation fails
- 401 if not authenticated

---

### Module 2: Addresses & Common Points

**Base Path**: `/api/addresses` and `/api/common-points`
**Files**: `src/routes/addressRoutes.js`
**Controller**: `src/controllers/addressController.js`
**Service**: `src/services/addressService.js`

#### GET /api/common-points

**Purpose**: Search common delivery points (hostels, PGs, offices)

**Authentication**: Required

**Query Parameters**:
- `search` (optional): Search term for name/city/pincode
- `city` (optional): Filter by city

**Business Logic**:
- If `search` provided: Case-insensitive search on name, city, or pincode (use ILIKE in Prisma)
- If `city` provided: Exact match on city
- Only return where `isActive = true`
- Order by name ascending
- Limit to 50 results

**Response (200)**:
```json
{
  "data": {
    "commonPoints": [
      {
        "id": "uuid",
        "name": "ABC Hostel",
        "addressLine1": "Street 123",
        "city": "Hyderabad",
        "state": "Telangana",
        "pincode": "500081",
        "latitude": 17.385,
        "longitude": 78.486
      }
    ]
  }
}
```

#### GET /api/addresses

**Purpose**: Get all addresses for authenticated user

**Authentication**: Required

**Business Logic**:
- Return all addresses where `user_id = req.user.id`
- Include related commonPoint data if linked
- Order by `isPrimary DESC`, then `createdAt DESC`

**Response (200)**:
```json
{
  "data": {
    "addresses": [
      {
        "id": "uuid",
        "tag": "home",
        "addressLine1": "Flat 301, Tower A",
        "addressLine2": "Green Avenue",
        "landmark": "Near Metro Station",
        "city": "Hyderabad",
        "state": "Telangana",
        "pincode": "500081",
        "latitude": 17.385,
        "longitude": 78.486,
        "phoneNumber": "+919876543210",
        "isPrimary": true,
        "commonPoint": null,
        "createdAt": "2025-01-10T09:00:00Z"
      }
    ]
  }
}
```

#### POST /api/addresses

**Purpose**: Create new address for user

**Authentication**: Required

**Request Body**:
```json
{
  "tag": "home",
  "addressLine1": "Flat 301, Tower A",
  "addressLine2": "Green Avenue",
  "landmark": "Near Metro Station",
  "city": "Hyderabad",
  "state": "Telangana",
  "pincode": "500081",
  "latitude": 17.385,
  "longitude": 78.486,
  "commonPointId": null,
  "phoneNumber": "+919876543210",
  "isPrimary": false
}
```

**Validation**:
- `tag`: Required, one of ADDRESS_TAGS
- `addressLine1`: Required, max 500 chars
- `addressLine2`: Optional, max 500 chars
- `landmark`: Optional, max 200 chars
- `city`: Required, max 100 chars
- `state`: Required, max 100 chars
- `pincode`: Required, 6 digits
- `latitude`: Optional, decimal (-90 to 90)
- `longitude`: Optional, decimal (-180 to 180)
- `commonPointId`: Optional, must exist if provided
- `phoneNumber`: Optional, 10-15 digits
- `isPrimary`: Optional boolean

**Business Logic**:
1. Count existing addresses for user
2. If count = 0: Force `isPrimary = true` (first address is always primary)
3. If `isPrimary = true` in request: Update all other user addresses to `isPrimary = false`
4. If `commonPointId` provided: Verify it exists
5. Create address record with `user_id = req.user.id`

**Response (201)**:
```json
{
  "data": {
    "address": { /* created address object */ }
  },
  "message": "Address created successfully"
}
```

**Errors**:
- 400 if validation fails
- 404 if commonPointId doesn't exist

#### PUT /api/addresses/:id

**Purpose**: Update existing address

**Authentication**: Required

**Authorization**: User can only update their own addresses

**Request Body**: Same fields as POST (all optional)

**Validation**: Same as POST

**Business Logic**:
1. Verify address exists and belongs to req.user.id (403 if not)
2. If setting `isPrimary = true`: Update all other user addresses to false
3. Update address fields

**Response (200)**:
```json
{
  "data": {
    "address": { /* updated address */ }
  },
  "message": "Address updated successfully"
}
```

**Errors**:
- 403 if address doesn't belong to user
- 404 if address not found

#### DELETE /api/addresses/:id

**Purpose**: Delete address

**Authentication**: Required

**Authorization**: User can only delete their own addresses

**Business Logic**:
1. Verify address belongs to user (403 if not)
2. Check if address is used in any active subscription
   - Query: `subscriptions` where `addressId = :id` AND `status = 'active'`
   - If found: Return 422 "Cannot delete address used in active subscription"
3. Count user's addresses
   - If count = 1: Return 422 "Cannot delete your only address"
4. Delete address

**Response (200)**:
```json
{
  "message": "Address deleted successfully"
}
```

**Errors**:
- 403 if doesn't belong to user
- 404 if not found
- 422 if used in active subscription or only address

---

### Module 3: Meal Packages

**Base Path**: `/api/meal-packages`
**Files**: `src/routes/mealPackageRoutes.js`
**Controller**: `src/controllers/mealPackageController.js`
**Service**: `src/services/mealPackageService.js`

#### GET /api/meal-packages

**Purpose**: Browse available meal subscription plans

**Authentication**: Required

**Query Parameters**:
- `diet` (optional): Filter by DIET_TYPES
- `cuisine` (optional): Filter by CUISINE_TYPES
- `duration` (optional): Filter by duration days (7 or 30)

**Business Logic**:
- Return packages where `isActive = true`
- Apply filters if provided (exact match)
- Order by price ascending
- Include all package details

**Response (200)**:
```json
{
  "data": {
    "packages": [
      {
        "id": "uuid",
        "name": "South Indian Veg Weekly",
        "dietType": "veg",
        "cuisineType": "south_indian",
        "includesBreakfast": true,
        "includesLunch": true,
        "includesDinner": true,
        "includesSnacks": false,
        "durationDays": 7,
        "defaultContainer": "steel",
        "allowsContainerChoice": true,
        "allowsDietUpgrade": false,
        "allowsCuisineUpgrade": true,
        "price": 1200.00
      }
    ]
  }
}
```

#### GET /api/meal-packages/:id

**Purpose**: Get detailed information about specific package

**Authentication**: Required

**Business Logic**:
- Find package by id where `isActive = true`
- If not found or inactive: 404

**Response (200)**:
```json
{
  "data": {
    "package": { /* full package details */ }
  }
}
```

**Errors**:
- 404 if package not found or inactive

---

### Module 4: Weekly Menus

**Base Path**: `/api/menus`
**Files**: `src/routes/menuRoutes.js`
**Controller**: `src/controllers/menuController.js`
**Service**: `src/services/menuService.js`

#### GET /api/menus/current

**Purpose**: Get this week's menu for user's subscription diet/cuisine

**Authentication**: Required

**Business Logic**:
1. Get user's active subscription (include meal package)
2. If no active subscription: Return 422 "No active subscription found"
3. Extract `dietType` and `cuisineType` from meal package
4. Calculate current week start date using `calculateWeekStart(getTodayIST())`
5. Find weekly_menus matching diet, cuisine, and week_start_date
6. Include related weekly_menu_items
7. If not found: Return 404 "Menu not available for current week"

**Response (200)**:
```json
{
  "data": {
    "menu": {
      "id": "uuid",
      "dietType": "veg",
      "cuisineType": "south_indian",
      "weekStartDate": "2025-01-13",
      "items": [
        {
          "id": "uuid",
          "itemType": "breakfast",
          "itemName": "Idli, Sambar, Chutney"
        },
        {
          "id": "uuid",
          "itemType": "lunch",
          "itemName": "Rice, Dal, Rasam, Vegetable Curry"
        }
      ]
    }
  }
}
```

**Errors**:
- 422 if no active subscription
- 404 if menu not found

#### GET /api/menus/week

**Purpose**: Get menu for specific week by date

**Authentication**: Required

**Query Parameters** (all required):
- `date`: Any date within the week (YYYY-MM-DD)
- `diet`: DIET_TYPES value
- `cuisine`: CUISINE_TYPES value

**Validation**:
- `date`: Valid ISO date format
- `diet`: Must be in DIET_TYPES
- `cuisine`: Must be in CUISINE_TYPES

**Business Logic**:
1. Calculate week start date from provided date
2. Find menu matching diet, cuisine, and week start
3. Include menu items

**Response (200)**: Same as `/current`

**Errors**:
- 400 if parameters invalid
- 404 if menu not found

---

### Module 5: Subscriptions

**Base Path**: `/api/subscriptions`
**Files**: `src/routes/subscriptionRoutes.js`
**Controller**: `src/controllers/subscriptionController.js`
**Service**: `src/services/subscriptionService.js`

#### POST /api/subscriptions

**Purpose**: Create new meal subscription

**Authentication**: Required

**Request Body**:
```json
{
  "mealPackageId": "uuid",
  "addressId": "uuid",
  "containerType": "steel",
  "startDate": "2025-01-20"
}
```

**Validation**:
- `mealPackageId`: Required UUID, must exist and be active
- `addressId`: Required UUID, must exist and belong to user
- `containerType`: Required, one of CONTAINER_TYPES
- `startDate`: Required ISO date, must be >= today (IST)

**Business Logic**:
1. Check user has no active subscription:
   - Query: `subscriptions` where `userId = req.user.id` AND `status = 'active'`
   - If found: Return 422 "You already have an active subscription"
2. Fetch meal package (verify exists and active)
3. Verify address belongs to user
4. If package.allowsContainerChoice = false:
   - Use package.defaultContainer, ignore request value
5. Calculate endDate = startDate + mealPackage.durationDays - 1
6. Create subscription with status = 'active'
7. **Do NOT generate subscription_meals here** (on-demand generation)
8. Create notification: "Your subscription has been activated successfully"
9. Return subscription with included mealPackage and address data

**Response (201)**:
```json
{
  "data": {
    "subscription": {
      "id": "uuid",
      "mealPackage": {
        "id": "uuid",
        "name": "South Indian Veg Weekly",
        "dietType": "veg",
        "cuisineType": "south_indian"
      },
      "address": {
        "id": "uuid",
        "tag": "home",
        "addressLine1": "Flat 301"
      },
      "containerType": "steel",
      "startDate": "2025-01-20",
      "endDate": "2025-01-26",
      "status": "active",
      "createdAt": "2025-01-15T12:00:00Z"
    }
  },
  "message": "Subscription created successfully"
}
```

**Errors**:
- 422 if user has active subscription
- 404 if package or address not found
- 403 if address doesn't belong to user
- 400 if validation fails

#### GET /api/subscriptions/active

**Purpose**: Get user's active subscription details

**Authentication**: Required

**Business Logic**:
1. Find subscription where `userId = req.user.id` AND `status = 'active'`
2. Include mealPackage and address relations
3. Calculate daysRemaining = endDate - getTodayIST()
4. If not found: Return 404 "No active subscription found"

**Response (200)**:
```json
{
  "data": {
    "subscription": {
      "id": "uuid",
      "mealPackage": { /* package details */ },
      "address": { /* full address */ },
      "containerType": "steel",
      "startDate": "2025-01-20",
      "endDate": "2025-01-26",
      "status": "active",
      "daysRemaining": 6
    }
  }
}
```

**Errors**:
- 404 if no active subscription

#### GET /api/subscriptions/history

**Purpose**: Get past subscriptions

**Authentication**: Required

**Business Logic**:
- Find subscriptions where `userId = req.user.id` AND `status IN ['completed', 'cancelled']`
- Include mealPackage data
- Order by endDate DESC
- Limit to 10 most recent

**Response (200)**:
```json
{
  "data": {
    "subscriptions": [
      {
        "id": "uuid",
        "mealPackage": { /* package details */ },
        "startDate": "2024-12-01",
        "endDate": "2024-12-30",
        "status": "completed",
        "createdAt": "2024-11-28T10:00:00Z"
      }
    ]
  }
}
```

---

### Remaining API Modules Summary

**Note**: Complete specifications for all endpoints follow the same pattern as above. Full details provided in user's original specification document. Implementation should include:

**Module 6-11 Endpoint List** (use exact same specification format):

- **GET /api/meals/schedule** - Get today + tomorrow meals (with on-demand generation)
- **POST /api/meals/pause** - Pause meal (8 PM deadline check)
- **POST /api/meals/unpause** - Resume paused meal
- **GET /api/delivery/slots** - List available time slots
- **POST /api/meals/:mealId/delivery-slot** - Change delivery time
- **POST /api/meals/group** - Group items for single delivery
- **DELETE /api/meals/group/:groupId** - Ungroup delivery
- **GET /api/curry/packages** - Browse curry token packages
- **POST /api/curry/purchase** - Buy curry tokens
- **GET /api/curry/wallet** - Check token balance
- **POST /api/curry/order** - Place curry order (with grouping option)
- **GET /api/curry/orders** - Get order history
- **DELETE /api/curry/orders/:orderId** - Cancel curry order (refund token)
- **GET /api/upgrades/prices** - Get upgrade pricing
- **POST /api/upgrades/apply** - Apply temporary upgrade
- **GET /api/upgrades/active** - Get active upgrades
- **DELETE /api/upgrades/:upgradeId** - Remove upgrade
- **GET /api/notifications** - Get user notifications
- **POST /api/notifications/:notificationId/read** - Mark as read
- **POST /api/notifications/read-all** - Mark all as read

**Total Endpoints**: ~40 endpoints across 11 modules

**Implementation Pattern for Each Endpoint**:
1. Define route with auth middleware
2. Add validation middleware using express-validator
3. Controller extracts request data, calls service
4. Service implements business logic using Prisma
5. Return formatted response using responseFormatter
6. Errors caught by centralized error handler

---

## Error Handling

### Centralized Error Middleware

**File**: `src/middleware/errorHandler.js`

**Purpose**: Catch all errors and return consistent JSON responses

**Implementation Pattern**:
```javascript
function errorHandler(err, req, res, next) {
  // Log error for debugging
  console.error('Error:', err);

  // Handle different error types

  // Prisma errors
  if (err.code === 'P2002') {
    // Unique constraint violation
    return res.status(422).json({
      error: {
        message: 'A record with this value already exists',
        code: 'DUPLICATE_ENTRY',
        status: 422
      }
    });
  }

  if (err.code === 'P2025') {
    // Record not found
    return res.status(404).json({
      error: {
        message: 'Record not found',
        code: 'NOT_FOUND',
        status: 404
      }
    });
  }

  // Express-validator errors
  if (err.errors && Array.isArray(err.errors)) {
    return res.status(400).json({
      error: {
        message: err.errors[0].msg,
        code: 'VALIDATION_ERROR',
        status: 400
      }
    });
  }

  // Custom business logic errors
  if (err.status && err.code) {
    return res.status(err.status).json({
      error: {
        message: err.message,
        code: err.code,
        status: err.status
      }
    });
  }

  // Default server error
  return res.status(500).json({
    error: {
      message: 'Internal server error',
      code: 'INTERNAL_ERROR',
      status: 500
    }
  });
}

module.exports = { errorHandler };
```

**Custom Error Class**:

**File**: `src/utils/AppError.js`

```javascript
class AppError extends Error {
  constructor(message, code, status) {
    super(message);
    this.code = code;
    this.status = status;
  }
}

module.exports = AppError;
```

**Usage in Services**:
```javascript
const AppError = require('../utils/AppError');

// Example
if (!subscription) {
  throw new AppError(
    'No active subscription found',
    'NO_ACTIVE_SUBSCRIPTION',
    422
  );
}
```

### Validation Middleware

**File**: `src/middleware/validate.js`

```javascript
const { validationResult } = require('express-validator');

function validate(req, res, next) {
  const errors = validationResult(req);
  if (!errors.isEmpty()) {
    return res.status(400).json({
      error: {
        message: errors.array()[0].msg,
        code: 'VALIDATION_ERROR',
        status: 400,
        details: errors.array()
      }
    });
  }
  next();
}

module.exports = { validate };
```

**Validator Example**:

**File**: `src/middleware/validators/subscriptionValidators.js`

```javascript
const { body } = require('express-validator');
const { CONTAINER_TYPES } = require('../../utils/constants');

const createSubscriptionValidation = [
  body('mealPackageId').isUUID().withMessage('Invalid meal package ID'),
  body('addressId').isUUID().withMessage('Invalid address ID'),
  body('containerType').isIn(CONTAINER_TYPES).withMessage('Invalid container type'),
  body('startDate').isISO8601().withMessage('Invalid start date format'),
];

module.exports = { createSubscriptionValidation };
```

---

## Prisma Client Setup

**File**: `src/utils/prisma.js`

```javascript
const { PrismaClient } = require('@prisma/client');

const prisma = new PrismaClient({
  log: process.env.NODE_ENV === 'development' ? ['query', 'error', 'warn'] : ['error'],
});

module.exports = prisma;
```

**Usage in Services**:
```javascript
const prisma = require('../utils/prisma');

async function getActiveSubscription(userId) {
  return await prisma.subscription.findFirst({
    where: {
      userId,
      status: 'active'
    },
    include: {
      mealPackage: true,
      address: true
    }
  });
}
```

---

## Package Configuration

### package.json

**File**: `package.json`

```json
{
  "name": "vyanjo-backend",
  "version": "1.0.0",
  "description": "Meal and curry subscription backend API",
  "main": "src/index.js",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js",
    "prisma:generate": "prisma generate",
    "prisma:migrate": "prisma migrate dev",
    "prisma:migrate:prod": "prisma migrate deploy",
    "prisma:studio": "prisma studio",
    "seed": "node prisma/seed.js"
  },
  "keywords": ["meal", "subscription", "api"],
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "@prisma/client": "^5.7.0",
    "firebase-admin": "^12.0.0",
    "express-validator": "^7.0.1",
    "moment-timezone": "^0.5.44",
    "dotenv": "^16.3.1"
  },
  "devDependencies": {
    "prisma": "^5.7.0",
    "nodemon": "^3.0.2"
  }
}
```

**Key Dependencies**:
- `express` - Web framework
- `@prisma/client` - Database ORM (generated)
- `firebase-admin` - Firebase authentication
- `express-validator` - Request validation
- `moment-timezone` - IST timezone handling
- `cors` - CORS middleware
- `dotenv` - Environment variable management

**Dev Dependencies**:
- `prisma` - Prisma CLI
- `nodemon` - Development hot reload

---

## Environment Variables

### .env.example

**File**: `.env.example`

```bash
# Server Configuration
NODE_ENV=development
PORT=3000

# Database
DATABASE_URL="postgresql://user:password@localhost:5432/vyanjo_db?schema=public"

# Firebase Admin SDK
FIREBASE_PROJECT_ID=your-project-id
FIREBASE_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\nYOUR_KEY_HERE\n-----END PRIVATE KEY-----\n"
FIREBASE_CLIENT_EMAIL=firebase-adminsdk-xxxxx@your-project.iam.gserviceaccount.com
```

**Setup Instructions**:
1. Copy `.env.example` to `.env`
2. Update `DATABASE_URL` with PostgreSQL connection string
3. Get Firebase service account:
   - Firebase Console → Project Settings → Service Accounts
   - Generate New Private Key (downloads JSON)
   - Extract `project_id`, `private_key`, `client_email`
4. For `FIREBASE_PRIVATE_KEY`: Keep newlines escaped as `\n` or encode as base64

### .gitignore

Ensure `.env` is in `.gitignore`:
```
node_modules/
.env
dist/
*.log
.DS_Store
```

---

## Entry Point (Server Initialization)

### Main Server File

**File**: `src/index.js`

```javascript
require('dotenv').config();
const express = require('express');
const cors = require('cors');
const { errorHandler } = require('./middleware/errorHandler');

// Import routes
const authRoutes = require('./routes/authRoutes');
const addressRoutes = require('./routes/addressRoutes');
const mealPackageRoutes = require('./routes/mealPackageRoutes');
const menuRoutes = require('./routes/menuRoutes');
const subscriptionRoutes = require('./routes/subscriptionRoutes');
const mealRoutes = require('./routes/mealRoutes');
const deliveryRoutes = require('./routes/deliveryRoutes');
const curryRoutes = require('./routes/curryRoutes');
const upgradeRoutes = require('./routes/upgradeRoutes');
const notificationRoutes = require('./routes/notificationRoutes');

const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Health check endpoint (no auth required)
app.get('/health', (req, res) => {
  res.json({
    status: 'ok',
    timestamp: new Date().toISOString(),
    environment: process.env.NODE_ENV
  });
});

// Mount API routes
app.use('/api/auth', authRoutes);
app.use('/api', addressRoutes);
app.use('/api', mealPackageRoutes);
app.use('/api', menuRoutes);
app.use('/api', subscriptionRoutes);
app.use('/api', mealRoutes);
app.use('/api', deliveryRoutes);
app.use('/api', curryRoutes);
app.use('/api', upgradeRoutes);
app.use('/api', notificationRoutes);

// 404 handler
app.use((req, res) => {
  res.status(404).json({
    error: {
      message: 'Endpoint not found',
      code: 'NOT_FOUND',
      status: 404
    }
  });
});

// Error handling middleware (must be last)
app.use(errorHandler);

// Start server
app.listen(PORT, () => {
  console.log(`🚀 Server running on port ${PORT}`);
  console.log(`📌 Environment: ${process.env.NODE_ENV}`);
  console.log(`🏥 Health check: http://localhost:${PORT}/health`);
});
```

**Startup Flow**:
1. Load environment variables
2. Initialize Express app
3. Apply middleware (CORS, JSON parsing)
4. Mount routes
5. Add error handlers
6. Start listening on PORT

---

## Database Seeding

### Seed Script

**File**: `prisma/seed.js`

**Purpose**: Populate initial data required for system to function

**Required Seed Data**:
1. Delivery time slots (4 default slots)
2. At least 1-2 meal packages for testing
3. Curry token packages (veg and non-veg)
4. Upgrade prices
5. Optional: Weekly menu for current week

```javascript
const { PrismaClient } = require('@prisma/client');
const moment = require('moment-timezone');

const prisma = new PrismaClient();

async function main() {
  console.log('🌱 Starting database seeding...');

  // 1. Delivery Time Slots
  console.log('Creating delivery time slots...');
  await prisma.deliveryTimeSlot.createMany({
    data: [
      { name: 'Morning Slot', startTime: '07:00:00', endTime: '08:00:00', isActive: true },
      { name: 'Afternoon Slot', startTime: '12:00:00', endTime: '13:00:00', isActive: true },
      { name: 'Evening Snack Slot', startTime: '17:00:00', endTime: '18:00:00', isActive: true },
      { name: 'Dinner Slot', startTime: '19:00:00', endTime: '20:00:00', isActive: true },
    ],
    skipDuplicates: true
  });

  // 2. Meal Packages
  console.log('Creating meal packages...');
  await prisma.mealPackage.createMany({
    data: [
      {
        name: 'South Indian Veg Weekly',
        dietType: 'veg',
        cuisineType: 'south_indian',
        includesBreakfast: true,
        includesLunch: true,
        includesDinner: true,
        includesSnacks: false,
        durationDays: 7,
        defaultContainer: 'steel',
        allowsContainerChoice: true,
        allowsDietUpgrade: false,
        allowsCuisineUpgrade: true,
        price: 1200.00,
        isActive: true
      },
      {
        name: 'North Indian Veg Monthly',
        dietType: 'veg',
        cuisineType: 'north_indian',
        includesBreakfast: true,
        includesLunch: true,
        includesDinner: true,
        includesSnacks: true,
        durationDays: 30,
        defaultContainer: 'plastic',
        allowsContainerChoice: true,
        allowsDietUpgrade: true,
        allowsCuisineUpgrade: false,
        price: 4500.00,
        isActive: true
      }
    ],
    skipDuplicates: true
  });

  // 3. Curry Token Packages
  console.log('Creating curry token packages...');
  await prisma.curryTokenPackage.createMany({
    data: [
      { name: 'Veg Curry 10-Pack', dietType: 'veg', tokenCount: 10, validityDays: 30, price: 500.00, isActive: true },
      { name: 'Veg Curry 20-Pack', dietType: 'veg', tokenCount: 20, validityDays: 60, price: 900.00, isActive: true },
      { name: 'Non-Veg Curry 10-Pack', dietType: 'non_veg', tokenCount: 10, validityDays: 30, price: 700.00, isActive: true },
    ],
    skipDuplicates: true
  });

  // 4. Upgrade Prices
  console.log('Creating upgrade prices...');
  await prisma.upgradePrice.createMany({
    data: [
      { upgradeType: 'veg_to_nonveg', scope: 'meal', mealType: 'lunch', price: 50.00, isActive: true },
      { upgradeType: 'veg_to_nonveg', scope: 'meal', mealType: 'dinner', price: 50.00, isActive: true },
      { upgradeType: 'veg_to_nonveg', scope: 'day', mealType: null, price: 100.00, isActive: true },
      { upgradeType: 'south_to_north', scope: 'day', mealType: null, price: 80.00, isActive: true },
      { upgradeType: 'south_to_north', scope: 'week', mealType: null, price: 500.00, isActive: true },
    ],
    skipDuplicates: true
  });

  console.log('✅ Seeding completed successfully!');
}

main()
  .catch((e) => {
    console.error('❌ Error during seeding:', e);
    process.exit(1);
  })
  .finally(async () => {
    await prisma.$disconnect();
  });
```

**Run Seed**:
```bash
npm run seed
```

---

## Development Workflow

### Initial Setup Steps

1. **Clone repository**
   ```bash
   cd vyanjo-backend
   ```

2. **Install dependencies**
   ```bash
   npm install
   ```

3. **Setup environment variables**
   ```bash
   cp .env.example .env
   # Edit .env with actual values
   ```

4. **Setup PostgreSQL database**
   - Create database: `vyanjo_db`
   - Note connection details for DATABASE_URL

5. **Run database migrations**
   ```bash
   npm run prisma:migrate
   ```

6. **Generate Prisma client**
   ```bash
   npm run prisma:generate
   ```

7. **Seed initial data**
   ```bash
   npm run seed
   ```

8. **Start development server**
   ```bash
   npm run dev
   ```

9. **Test health endpoint**
   ```bash
   curl http://localhost:3000/health
   ```

### Development Commands

- `npm run dev` - Start with nodemon (hot reload)
- `npm run prisma:studio` - Open Prisma Studio (database GUI)
- `npm run prisma:migrate` - Create new migration
- `npm run prisma:generate` - Regenerate Prisma client
- `npm start` - Start production server

### Project Initialization Sequence

**When starting from scratch** (empty vyanjo-backend folder):

1. Initialize npm:
   ```bash
   npm init -y
   ```

2. Install dependencies:
   ```bash
   npm install express cors @prisma/client firebase-admin express-validator moment-timezone dotenv
   npm install -D prisma nodemon
   ```

3. Initialize Prisma:
   ```bash
   npx prisma init
   ```

4. Copy complete Prisma schema from user's specification to `prisma/schema.prisma`

5. Create all directory structure:
   ```bash
   mkdir -p src/{controllers,services,routes,middleware,utils,config}
   mkdir -p src/middleware/validators
   mkdir -p prisma
   ```

6. Create all files as specified in this plan

7. Follow setup steps above

---

## Testing Approach

### Manual Testing with Postman/Thunder Client

**Create test collection with these sequences**:

**1. Authentication Flow**:
- Authenticate with Firebase (external tool/frontend)
- Copy ID token
- POST /api/auth/login with token
- Verify user created

**2. Address Setup**:
- POST /api/addresses (create first address)
- Verify isPrimary = true automatically
- POST /api/addresses (create second)
- GET /api/addresses (verify list)

**3. Browse and Subscribe**:
- GET /api/meal-packages
- POST /api/subscriptions
- GET /api/subscriptions/active

**4. Meal Schedule**:
- GET /api/meals/schedule
- Verify meals generated on-demand
- Check default delivery slots assigned

**5. Pause Functionality**:
- POST /api/meals/pause (tomorrow's lunch)
- Verify success
- Try pausing today after 8 PM (should fail)

**6. Curry Tokens**:
- GET /api/curry/packages
- POST /api/curry/purchase
- GET /api/curry/wallet (check balance)
- POST /api/curry/order
- Verify token deducted

**7. Delivery Grouping**:
- POST /api/meals/group (group tiffin + curry)
- Verify same delivery_group_id
- DELETE /api/meals/group/:id (ungroup)

**8. Upgrades**:
- GET /api/upgrades/prices
- POST /api/upgrades/apply
- GET /api/upgrades/active

**9. Notifications**:
- GET /api/notifications
- Verify notifications created for all actions
- POST /api/notifications/:id/read

### Test Environment Variables

**For local testing**: Use `.env` with local PostgreSQL and Firebase test project

---

## Production Deployment Considerations

### Environment Setup

- **NODE_ENV**: Set to `production`
- **Database**: PostgreSQL instance (Supabase, Railway, AWS RDS, etc.)
- **Firebase**: Production Firebase project with service account

### Pre-Deployment Steps

1. **Run production migration**:
   ```bash
   npm run prisma:migrate:prod
   ```

2. **Generate Prisma client**:
   ```bash
   npm run prisma:generate
   ```

3. **Seed production data**:
   ```bash
   npm run seed
   ```

4. **Start server**:
   ```bash
   npm start
   ```

### Security Considerations

- Ensure `.env` is never committed (in `.gitignore`)
- Use strong DATABASE_URL password
- Rotate Firebase service account keys periodically
- Enable CORS only for trusted origins in production
- Add rate limiting middleware (optional future enhancement)
- Use HTTPS in production

### Monitoring

- Health check endpoint: `GET /health`
- Log errors to external service (future: Sentry, LogRocket)
- Monitor database connections
- Track API response times

---

## File-by-File Implementation Checklist

### Configuration Files
- [ ] `package.json` - Dependencies and scripts
- [ ] `.env.example` - Environment variable template
- [ ] `.gitignore` - Ignore node_modules, .env, etc.
- [ ] `prisma/schema.prisma` - Complete database schema (17 models)

### Source Code Structure
- [ ] `src/index.js` - Main server entry point
- [ ] `src/config/firebase.js` - Firebase Admin SDK init
- [ ] `src/utils/prisma.js` - Prisma client singleton
- [ ] `src/utils/constants.js` - Enum definitions
- [ ] `src/utils/timezone.js` - IST timezone helpers
- [ ] `src/utils/responseFormatter.js` - Response helpers
- [ ] `src/utils/AppError.js` - Custom error class
- [ ] `src/middleware/auth.js` - Firebase token verification
- [ ] `src/middleware/errorHandler.js` - Centralized error handler
- [ ] `src/middleware/validate.js` - Validation middleware wrapper

### Validators (express-validator schemas)
- [ ] `src/middleware/validators/authValidators.js`
- [ ] `src/middleware/validators/addressValidators.js`
- [ ] `src/middleware/validators/subscriptionValidators.js`
- [ ] `src/middleware/validators/mealValidators.js`
- [ ] `src/middleware/validators/curryValidators.js`
- [ ] `src/middleware/validators/upgradeValidators.js`

### Routes
- [ ] `src/routes/authRoutes.js`
- [ ] `src/routes/addressRoutes.js`
- [ ] `src/routes/mealPackageRoutes.js`
- [ ] `src/routes/menuRoutes.js`
- [ ] `src/routes/subscriptionRoutes.js`
- [ ] `src/routes/mealRoutes.js`
- [ ] `src/routes/deliveryRoutes.js`
- [ ] `src/routes/curryRoutes.js`
- [ ] `src/routes/upgradeRoutes.js`
- [ ] `src/routes/notificationRoutes.js`

### Controllers (10 controllers)
- [ ] `src/controllers/authController.js`
- [ ] `src/controllers/addressController.js`
- [ ] `src/controllers/mealPackageController.js`
- [ ] `src/controllers/menuController.js`
- [ ] `src/controllers/subscriptionController.js`
- [ ] `src/controllers/mealController.js`
- [ ] `src/controllers/deliveryController.js`
- [ ] `src/controllers/curryController.js`
- [ ] `src/controllers/upgradeController.js`
- [ ] `src/controllers/notificationController.js`

### Services (10 services with business logic)
- [ ] `src/services/authService.js`
- [ ] `src/services/addressService.js`
- [ ] `src/services/mealPackageService.js`
- [ ] `src/services/menuService.js`
- [ ] `src/services/subscriptionService.js`
- [ ] `src/services/mealService.js` - **Critical: On-demand meal generation**
- [ ] `src/services/deliveryService.js`
- [ ] `src/services/curryService.js`
- [ ] `src/services/upgradeService.js`
- [ ] `src/services/notificationService.js`

### Database
- [ ] `prisma/seed.js` - Initial data seeding script

### Total Files to Create
**~50 files** across all layers

---

## Summary

This implementation plan provides:

✅ **Complete backend system** for meal + curry subscription service
✅ **Node.js with JavaScript only** (no TypeScript)
✅ **Express.js** REST API with ~40 endpoints
✅ **PostgreSQL + Prisma ORM** with 17-table schema
✅ **Firebase phone authentication**
✅ **IST timezone support** with 8 PM pause deadline
✅ **On-demand meal generation** (optimized performance)
✅ **48-hour scheduling window** (today + tomorrow)
✅ **Curry token wallet system**
✅ **Temporary upgrades** (diet/cuisine)
✅ **Delivery grouping** (tiffin + curry together)
✅ **Business logic enforcement** (one active subscription, primary address, etc.)
✅ **Comprehensive error handling**
✅ **Production-ready architecture**

### Key Implementation Priorities

1. **Core Infrastructure** (Day 1-2):
   - Setup project structure, install dependencies
   - Configure Prisma schema and run migrations
   - Setup Firebase authentication
   - Implement auth middleware and error handling

2. **Essential APIs** (Day 3-5):
   - Authentication endpoints
   - Address management
   - Meal packages and subscriptions
   - Meal schedule with on-demand generation

3. **Advanced Features** (Day 6-8):
   - Pause/unpause with 8 PM deadline
   - Curry token system
   - Delivery grouping
   - Upgrades
   - Notifications

4. **Testing & Polish** (Day 9-10):
   - Seed data
   - Manual testing
   - Bug fixes
   - Documentation

### Critical Success Factors

- **On-demand meal generation**: Meals created only when user views schedule or pauses
- **Timezone handling**: All date logic uses IST, especially 8 PM pause deadline
- **One active subscription**: Enforced at DB + application level
- **Token management**: Proper increment/decrement of curry tokens
- **Error handling**: Consistent error responses across all endpoints
- **Validation**: All inputs validated with express-validator

---

**End of Implementation Plan**

This document provides complete specifications for implementing the Vyanjo meal + curry subscription backend. All API endpoints, business logic rules, database schema, and implementation patterns are fully specified. Implementation agent can proceed with mechanical implementation following this plan exactly.
