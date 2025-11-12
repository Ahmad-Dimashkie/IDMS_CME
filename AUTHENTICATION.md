# Authentication System Documentation - IDMS_CME

## Overview

The IDMS_CME (Intelligent Document Management System) uses a **dual authentication system** combining **Firebase Authentication** and **Django backend validation**. This hybrid approach provides both the simplicity of Firebase's authentication services and the security of server-side validation.

## Architecture

The authentication system consists of:
- **Frontend**: React application with Firebase SDK
- **Backend**: Django REST API with Firebase Admin integration via Pyrebase
- **Storage**: SQLite database for user data with Firebase UID mapping

## Authentication Flow

### 1. Sign Up Process

#### Frontend (SignUp.jsx)
```
User enters credentials → POST to /api/signup
├── Email
├── Password  
├── First Name
└── Last Name
```

#### Backend (user_functions.py)
```python
1. Validates user data using UserSerializer
2. Creates user in Firebase using pyrebase:
   - auth.create_user_with_email_and_password(email, password)
3. Stores user in Django database:
   - Email (unique identifier)
   - Hashed password
   - Firebase UID (from Firebase response)
   - First/Last name
4. Returns Firebase user object with idToken
```

#### Frontend Response Handling
```
Success → Store idToken in localStorage → Navigate to /homepage
Error → Display error message to user
```

### 2. Login Process

#### Frontend (SignIn.jsx)
```
User enters credentials → POST to /api/login
├── Email
└── Password
```

#### Backend (user_functions.py)
```python
1. Validates email and password are provided
2. Authenticates with Firebase:
   - auth.sign_in_with_email_and_password(email, password)
   - Receives Firebase user object with localId
3. Validates against Django database:
   - Looks up User by email
   - Verifies firebase_uid matches
   - Returns error if user doesn't exist or UIDs don't match
4. Returns success with user data and firebase_uid
```

#### Frontend Response Handling
```javascript
Success:
├── Store idToken in localStorage
├── Store email in localStorage
├── Store firebase_id in localStorage
└── Navigate to /homepage

Error:
└── Display error message
```

### 3. Session Management

#### AuthContext (AuthContext.jsx)
- Uses React Context API for state management
- Stores currentUser in component state
- Loads user from localStorage on mount
- Does NOT use Firebase's onAuthStateChanged

#### Session Persistence
```javascript
localStorage stores:
├── token: Firebase idToken
├── email: User email
└── firebase_id: Firebase localId
```

### 4. Protected Routes (PrivateRoute.jsx)

```javascript
Route Protection Logic:
if (currentUser || localStorage.getItem("token") != null) {
  → Allow access to protected route
} else {
  → Redirect to "/" (login page)
}
```

Protected routes include:
- `/homepage` - Main dashboard
- `/view-pdf` - PDF viewer
- `/profile` - User profile
- `/cases` - Cases listing
- `/cases/:name` - Case details

### 5. Logout Process

#### Frontend (LogoutButton.jsx)
```javascript
1. Calls logout() from AuthContext
2. Clears all localStorage data
3. Sets currentUser to null
4. Navigates to "/" (login page)
```

## Component Architecture

### Frontend Components

1. **SignIn.jsx**
   - Email/password input fields
   - Calls `/api/login` endpoint
   - Stores authentication tokens
   - Handles error display

2. **SignUp.jsx**
   - User registration form
   - Calls `/api/signup` endpoint
   - Stores authentication tokens
   - Handles error display

3. **AuthContext.jsx**
   - Provides authentication state
   - Manages currentUser
   - Provides logout function
   - Loads user from localStorage

4. **PrivateRoute.jsx**
   - HOC for route protection
   - Checks for token or currentUser
   - Redirects to login if unauthenticated

5. **LogoutButton.jsx**
   - Logout UI component
   - Clears session data
   - Redirects to login

### Backend Components

1. **user_functions.py**
   - `login()` - Handles login requests
   - `signup()` - Handles registration
   - Uses Pyrebase for Firebase integration
   - Validates against Django User model

2. **models.py - User Model**
   ```python
   - id: UUID (primary key)
   - email: EmailField (unique, USERNAME_FIELD)
   - password: Hashed password
   - firebase_uid: CharField (unique)
   - first_name, last_name: User details
   - Custom UserManager for user creation
   ```

3. **serializers.py - UserSerializer**
   - Validates user data on signup
   - Ensures email uniqueness
   - Validates password requirements

## Firebase Configuration

### Frontend (Firebase.js)
```javascript
Firebase Config:
├── apiKey: "AIzaSyBPDVCBr4qzEmEMuq9U_SWoCCXzBI4UmIc"
├── authDomain: "idms-a1c57.firebaseapp.com"
├── projectId: "idms-a1c57"
├── storageBucket: "idms-a1c57.appspot.com"
├── messagingSenderId: "287855361713"
└── appId: "1:287855361713:web:771954928f92d21a976c90"
```

### Backend
- Uses Pyrebase library
- Requires firebaseConfig (imported but missing in repo)
- Should contain same Firebase project credentials

## Security Features

### Current Implementation

1. **Password Hashing**
   - Django's `set_password()` hashes passwords
   - Uses PBKDF2 algorithm by default

2. **Dual Validation**
   - Firebase validates credentials
   - Django verifies firebase_uid matches stored value
   - Prevents unauthorized access even with Firebase token

3. **Email Uniqueness**
   - Enforced at both Firebase and Django levels
   - Prevents duplicate accounts

4. **Token-Based Sessions**
   - Uses Firebase idToken
   - Stored in localStorage
   - Checked on each protected route access

### Security Considerations

⚠️ **Current Vulnerabilities:**

1. **Firebase Config Exposure**
   - Firebase API keys are publicly visible in source code
   - This is normal for Firebase (API keys are not secret)
   - However, Firebase security rules should be properly configured

2. **Missing firebase_config.py**
   - Backend imports non-existent firebase_config
   - This file needs to be created or environment variables used

3. **No Token Expiration Handling**
   - Firebase tokens expire after 1 hour
   - No refresh token mechanism implemented
   - Users must re-login after token expiration

4. **No HTTPS Enforcement**
   - API calls use http://localhost:8000
   - Production should use HTTPS

5. **CORS Configuration**
   - Must be properly configured for production
   - Currently allows localhost

6. **No Rate Limiting**
   - Login endpoint has no rate limiting
   - Vulnerable to brute force attacks

7. **localStorage Security**
   - Tokens stored in localStorage (vulnerable to XSS)
   - Consider using httpOnly cookies for production

## API Endpoints

### POST /api/signup
**Request:**
```json
{
  "email": "user@example.com",
  "password": "securepassword",
  "first_name": "John",
  "last_name": "Doe"
}
```

**Success Response (201):**
```json
{
  "message": "Successfully created an account!",
  "user": {
    "idToken": "...",
    "email": "user@example.com",
    "localId": "firebase_uid",
    ...
  }
}
```

**Error Responses:**
- 400: Email already in use
- 400: Validation errors
- 400: Firebase error

### POST /api/login
**Request:**
```json
{
  "email": "user@example.com",
  "password": "securepassword"
}
```

**Success Response (200):**
```json
{
  "message": "Successfully logged in!",
  "user": {
    "idToken": "...",
    "email": "user@example.com",
    "localId": "firebase_uid",
    ...
  },
  "firebase_uid": "firebase_uid"
}
```

**Error Responses:**
- 400: Email and password are required
- 400: Invalid email or password
- 400: User does not exist in Django backend
- 400: User ID mismatch

## Data Flow Diagram

```
┌──────────────┐
│   Browser    │
│  (React App) │
└──────┬───────┘
       │
       │ 1. POST /api/login {email, password}
       │
       ▼
┌──────────────────┐
│  Django Backend  │
│  user_functions  │
└──────┬───────────┘
       │
       │ 2. auth.sign_in_with_email_and_password()
       │
       ▼
┌──────────────┐
│   Firebase   │
│  Auth Service│
└──────┬───────┘
       │
       │ 3. Returns {idToken, localId, ...}
       │
       ▼
┌──────────────────┐
│  Django Backend  │
│  Validates User  │
└──────┬───────────┘
       │
       │ 4. Query User.objects.get(email=email)
       │
       ▼
┌──────────────┐
│   SQLite DB  │
│  User Table  │
└──────┬───────┘
       │
       │ 5. Verify firebase_uid matches
       │
       ▼
┌──────────────┐
│   Browser    │
│ Store tokens │
│  in storage  │
└──────────────┘
```

## Recommendations for Improvement

1. **Create firebase_config.py**
   ```python
   firebaseConfig = {
       "apiKey": "...",
       "authDomain": "...",
       "databaseURL": "...",
       "projectId": "...",
       "storageBucket": "...",
       "messagingSenderId": "...",
       "appId": "..."
   }
   ```

2. **Implement Token Refresh**
   - Add refresh token mechanism
   - Handle token expiration gracefully
   - Auto-refresh before expiration

3. **Add Rate Limiting**
   ```python
   from rest_framework.throttling import AnonRateThrottle
   # Apply to login/signup endpoints
   ```

4. **Enhance Error Handling**
   - More specific error messages for debugging
   - Log authentication attempts
   - Implement account lockout after failed attempts

5. **Use Environment Variables**
   - Move Firebase config to .env file
   - Use different configs for dev/prod
   - Never commit secrets to repository

6. **Implement Password Requirements**
   - Minimum length
   - Complexity requirements
   - Password strength indicator

7. **Add Email Verification**
   - Firebase supports email verification
   - Require verification before access

8. **Implement Two-Factor Authentication**
   - Firebase supports 2FA
   - Add optional 2FA for enhanced security

9. **Session Timeout**
   - Implement automatic logout
   - Warning before session expires
   - Activity-based session extension

10. **Audit Logging**
    - Log all authentication events
    - Track failed login attempts
    - Monitor suspicious activities

## Testing Authentication

### Manual Testing Steps

1. **Test Signup:**
   ```
   - Navigate to /signup
   - Enter valid email and password
   - Verify account creation
   - Check localStorage for tokens
   - Verify redirect to /homepage
   ```

2. **Test Login:**
   ```
   - Navigate to /
   - Enter credentials
   - Verify successful login
   - Check localStorage for tokens
   - Verify redirect to /homepage
   ```

3. **Test Protected Routes:**
   ```
   - Try accessing /homepage without login
   - Verify redirect to /
   - Login and access /homepage
   - Verify successful access
   ```

4. **Test Logout:**
   ```
   - Login successfully
   - Click logout button
   - Verify localStorage cleared
   - Verify redirect to /
   - Try accessing protected route
   - Verify redirect back to /
   ```

5. **Test Error Handling:**
   ```
   - Try login with wrong password
   - Try signup with existing email
   - Try login with non-existent email
   - Verify appropriate error messages
   ```

## Conclusion

The IDMS_CME authentication system provides a robust foundation with Firebase and Django integration. The dual-layer validation ensures that even if Firebase authentication is compromised, the Django backend provides an additional security layer. However, several improvements are recommended for production deployment, particularly around token management, security hardening, and error handling.
