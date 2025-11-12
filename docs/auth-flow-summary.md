# Authentication Flow - Quick Reference

## High-Level Overview

```
┌─────────────────────────────────────────────────────────────────┐
│                    IDMS_CME Authentication                      │
│                  Firebase + Django Dual Layer                   │
└─────────────────────────────────────────────────────────────────┘
```

## Login Process (Step-by-Step)

```
1. USER ACTION
   │
   ├─ User enters email & password
   └─ Clicks "Sign In" button
   
2. FRONTEND (SignIn.jsx)
   │
   ├─ axios.post('http://localhost:8000/api/login', {email, password})
   └─ Sends credentials to Django backend
   
3. BACKEND (user_functions.py - login())
   │
   ├─ Receives email & password
   ├─ Validates inputs exist
   └─ Calls Firebase authentication
   
4. FIREBASE AUTHENTICATION
   │
   ├─ auth.sign_in_with_email_and_password(email, password)
   ├─ Validates credentials against Firebase Auth
   └─ Returns user object with:
       • idToken (JWT token)
       • localId (Firebase UID)
       • email
       • refreshToken
       • expiresIn
   
5. BACKEND VALIDATION (Django)
   │
   ├─ Queries Django database: User.objects.get(email=email)
   ├─ Verifies firebase_uid matches Firebase's localId
   └─ Returns success or error
   
6. FRONTEND RESPONSE HANDLING
   │
   ├─ On Success:
   │   ├─ localStorage.setItem("token", idToken)
   │   ├─ localStorage.setItem("email", email)
   │   ├─ localStorage.setItem("firebase_id", localId)
   │   └─ navigate("/homepage")
   │
   └─ On Error:
       └─ Display error message
```

## Sign Up Process (Step-by-Step)

```
1. USER ACTION
   │
   ├─ User enters: email, password, first name, last name
   └─ Clicks "Sign Up" button
   
2. FRONTEND (SignUp.jsx)
   │
   ├─ axios.post('http://localhost:8000/api/signup', userData)
   └─ Sends registration data to Django backend
   
3. BACKEND (user_functions.py - signup())
   │
   ├─ Validates data with UserSerializer
   ├─ Checks email format, password requirements
   └─ Calls Firebase to create user
   
4. FIREBASE USER CREATION
   │
   ├─ auth.create_user_with_email_and_password(email, password)
   ├─ Creates user in Firebase Auth
   └─ Returns user object with localId and idToken
   
5. DJANGO USER CREATION
   │
   ├─ User.objects.create_user(
   │       email=email,
   │       password=password,  # Django hashes this
   │       firebase_uid=localId,
   │       first_name=first_name,
   │       last_name=last_name
   │   )
   └─ Stores user in SQLite database
   
6. FRONTEND RESPONSE HANDLING
   │
   ├─ localStorage.setItem("token", idToken)
   └─ navigate("/homepage")
```

## Session Management

```
┌─────────────────────────────────────────────────────────────┐
│                    LocalStorage Contents                    │
├─────────────────────────────────────────────────────────────┤
│  • token: Firebase JWT idToken (expires in 1 hour)         │
│  • email: User's email address                             │
│  • firebase_id: Firebase user localId (UID)                │
└─────────────────────────────────────────────────────────────┘

Protected Route Check (PrivateRoute.jsx):
    ↓
if (currentUser OR localStorage.token exists)
    → ALLOW access to protected route
else
    → REDIRECT to "/" (login page)
```

## Logout Process

```
1. USER CLICKS LOGOUT
   │
2. LogoutButton.jsx
   │
   ├─ Calls logout() from AuthContext
   └─ Triggers cleanup
   
3. AuthContext.logout()
   │
   ├─ localStorage.clear()  // Removes all tokens
   └─ setCurrentUser(null)  // Clears app state
   
4. NAVIGATION
   │
   └─ navigate("/")  // Redirect to login page
```

## Component Responsibilities

```
┌──────────────────────┬────────────────────────────────────────┐
│     Component        │            Responsibility              │
├──────────────────────┼────────────────────────────────────────┤
│ SignIn.jsx           │ Login form, API call, token storage   │
│ SignUp.jsx           │ Registration form, API call           │
│ AuthContext.jsx      │ Global auth state, logout function    │
│ PrivateRoute.jsx     │ Route protection, redirect logic      │
│ LogoutButton.jsx     │ Logout UI and trigger                 │
│ Firebase.js          │ Firebase initialization               │
├──────────────────────┼────────────────────────────────────────┤
│ user_functions.py    │ Login/signup API endpoints            │
│ models.py (User)     │ User model with firebase_uid          │
│ serializers.py       │ Input validation                      │
│ urls.py              │ Route /api/login, /api/signup         │
└──────────────────────┴────────────────────────────────────────┘
```

## Key Security Points

```
✅ IMPLEMENTED
├─ Password hashing (Django PBKDF2)
├─ Dual validation (Firebase + Django)
├─ Email uniqueness
└─ Token-based authentication

⚠️  MISSING/RECOMMENDED
├─ Token refresh mechanism (tokens expire in 1 hour)
├─ Rate limiting (prevent brute force)
├─ HTTPS enforcement (use in production)
├─ HttpOnly cookies (instead of localStorage)
├─ Email verification
└─ Two-factor authentication
```

## Database Schema

```
┌─────────────────────────────────────────────────────────────┐
│                      User Model (SQLite)                    │
├─────────────────────┬───────────────────────────────────────┤
│ Field               │ Description                           │
├─────────────────────┼───────────────────────────────────────┤
│ id                  │ UUID (Primary Key)                    │
│ email               │ EmailField (Unique, USERNAME_FIELD)   │
│ password            │ Hashed password (Django hash)         │
│ firebase_uid        │ CharField (Unique, from Firebase)     │
│ first_name          │ CharField                             │
│ last_name           │ CharField                             │
│ is_staff            │ BooleanField                          │
│ is_superuser        │ BooleanField                          │
│ date_joined         │ DateTimeField                         │
└─────────────────────┴───────────────────────────────────────┘
```

## Error Scenarios

```
Login Errors:
├─ Missing email/password → 400 "Email and password are required"
├─ Invalid credentials → 400 "Invalid email or password"
├─ User not in Django DB → 400 "User does not exist in Django backend"
└─ UID mismatch → 400 "User ID mismatch"

Signup Errors:
├─ Email already exists → 400 "Email already in use"
├─ Invalid data → 400 with validation errors
└─ Firebase error → 400 with error details
```

## Quick Testing Checklist

```
□ Sign up with new email → Success, redirects to /homepage
□ Sign up with existing email → Shows error "Email already in use"
□ Login with correct credentials → Success, redirects to /homepage
□ Login with wrong password → Shows error "Invalid email or password"
□ Access /homepage without login → Redirects to /
□ Access /homepage after login → Shows homepage
□ Logout → Clears storage, redirects to /
□ Access /homepage after logout → Redirects to /
```

---

For complete details, see [AUTHENTICATION.md](../AUTHENTICATION.md)
