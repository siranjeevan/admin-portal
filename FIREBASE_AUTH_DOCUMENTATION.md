# Firebase Role-Based Authentication Documentation

## Table of Contents
1. [Overview](#overview)
2. [Firebase Setup](#firebase-setup)
3. [Project Structure](#project-structure)
4. [Implementation Guide](#implementation-guide)
5. [User Roles](#user-roles)
6. [Security Rules](#security-rules)
7. [Usage Examples](#usage-examples)
8. [Testing](#testing)

---

## Overview

This document provides a complete guide for implementing Firebase role-based authentication in the Admin Portal application. The system supports multiple user roles with different access levels.

### Features
- Email/Password authentication
- Role-based access control (RBAC)
- Protected routes
- Custom claims for roles
- Firestore security rules
- Session management

---

## Firebase Setup

### 1. Create Firebase Project

1. Go to [Firebase Console](https://console.firebase.google.com/)
2. Click "Add Project"
3. Enter project name: `admin-portal-hope3`
4. Enable Google Analytics (optional)
5. Create project

### 2. Enable Authentication

1. In Firebase Console, go to **Authentication**
2. Click **Get Started**
3. Enable **Email/Password** sign-in method
4. Click **Save**

### 3. Create Firestore Database

1. Go to **Firestore Database**
2. Click **Create Database**
3. Start in **Production Mode**
4. Choose location (e.g., `us-central`)
5. Click **Enable**

### 4. Get Firebase Configuration

1. Go to **Project Settings** (gear icon)
2. Scroll to **Your apps**
3. Click **Web** icon (`</>`)
4. Register app name: `admin-portal`
5. Copy the configuration object

---

## Project Structure

```
admin-portal/
├── src/
│   ├── config/
│   │   └── firebase.js              # Firebase configuration
│   ├── contexts/
│   │   └── AuthContext.jsx          # Authentication context
│   ├── hooks/
│   │   └── useAuth.js               # Custom auth hook
│   ├── components/
│   │   ├── ProtectedRoute.jsx       # Route protection
│   │   ├── Login.jsx                # Login component
│   │   └── RoleBasedComponent.jsx   # Role-based rendering
│   ├── utils/
│   │   └── roleUtils.js             # Role utility functions
│   └── App.jsx
├── firestore.rules                   # Firestore security rules
└── FIREBASE_AUTH_DOCUMENTATION.md
```

---

## Implementation Guide

### Step 1: Install Firebase SDK

```bash
npm install firebase
```

### Step 2: Firebase Configuration

Create `src/config/firebase.js`:

```javascript
import { initializeApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';
import { getFirestore } from 'firebase/firestore';

const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_PROJECT_ID.firebaseapp.com",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_PROJECT_ID.appspot.com",
  messagingSenderId: "YOUR_MESSAGING_SENDER_ID",
  appId: "YOUR_APP_ID"
};

// Initialize Firebase
const app = initializeApp(firebaseConfig);

// Initialize services
export const auth = getAuth(app);
export const db = getFirestore(app);

export default app;
```

### Step 3: Authentication Context

Create `src/contexts/AuthContext.jsx`:

```javascript
import { createContext, useContext, useState, useEffect } from 'react';
import { 
  signInWithEmailAndPassword,
  signOut,
  onAuthStateChanged,
  createUserWithEmailAndPassword
} from 'firebase/auth';
import { doc, getDoc, setDoc } from 'firebase/firestore';
import { auth, db } from '../config/firebase';

const AuthContext = createContext();

export const useAuth = () => {
  const context = useContext(AuthContext);
  if (!context) {
    throw new Error('useAuth must be used within AuthProvider');
  }
  return context;
};

export const AuthProvider = ({ children }) => {
  const [currentUser, setCurrentUser] = useState(null);
  const [userRole, setUserRole] = useState(null);
  const [loading, setLoading] = useState(true);

  // Sign up new user with role
  const signup = async (email, password, role = 'viewer') => {
    const userCredential = await createUserWithEmailAndPassword(auth, email, password);
    const user = userCredential.user;
    
    // Store user role in Firestore
    await setDoc(doc(db, 'users', user.uid), {
      email: user.email,
      role: role,
      createdAt: new Date().toISOString()
    });
    
    return userCredential;
  };

  // Sign in user
  const login = async (email, password) => {
    return signInWithEmailAndPassword(auth, email, password);
  };

  // Sign out user
  const logout = () => {
    return signOut(auth);
  };

  // Get user role from Firestore
  const getUserRole = async (uid) => {
    try {
      const userDoc = await getDoc(doc(db, 'users', uid));
      if (userDoc.exists()) {
        return userDoc.data().role;
      }
      return null;
    } catch (error) {
      console.error('Error getting user role:', error);
      return null;
    }
  };

  // Check if user has required role
  const hasRole = (requiredRole) => {
    const roleHierarchy = {
      'super_admin': 4,
      'admin': 3,
      'editor': 2,
      'viewer': 1
    };
    
    return roleHierarchy[userRole] >= roleHierarchy[requiredRole];
  };

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, async (user) => {
      setCurrentUser(user);
      
      if (user) {
        const role = await getUserRole(user.uid);
        setUserRole(role);
      } else {
        setUserRole(null);
      }
      
      setLoading(false);
    });

    return unsubscribe;
  }, []);

  const value = {
    currentUser,
    userRole,
    signup,
    login,
    logout,
    hasRole,
    loading
  };

  return (
    <AuthContext.Provider value={value}>
      {!loading && children}
    </AuthContext.Provider>
  );
};
```

### Step 4: Protected Route Component

Create `src/components/ProtectedRoute.jsx`:

```javascript
import { Navigate } from 'react-router-dom';
import { useAuth } from '../contexts/AuthContext';

const ProtectedRoute = ({ children, requiredRole }) => {
  const { currentUser, userRole, hasRole } = useAuth();

  if (!currentUser) {
    return <Navigate to="/login" replace />;
  }

  if (requiredRole && !hasRole(requiredRole)) {
    return <Navigate to="/unauthorized" replace />;
  }

  return children;
};

export default ProtectedRoute;
```

### Step 5: Login Component

Create `src/components/Login.jsx`:

```javascript
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { useAuth } from '../contexts/AuthContext';

const Login = () => {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [loading, setLoading] = useState(false);
  
  const { login } = useAuth();
  const navigate = useNavigate();

  const handleSubmit = async (e) => {
    e.preventDefault();
    
    try {
      setError('');
      setLoading(true);
      await login(email, password);
      navigate('/dashboard');
    } catch (error) {
      setError('Failed to log in: ' + error.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="min-h-screen flex items-center justify-center bg-gray-100">
      <div className="bg-white p-8 rounded-lg shadow-md w-96">
        <h2 className="text-2xl font-bold mb-6 text-center">Admin Login</h2>
        
        {error && (
          <div className="bg-red-100 border border-red-400 text-red-700 px-4 py-3 rounded mb-4">
            {error}
          </div>
        )}
        
        <form onSubmit={handleSubmit}>
          <div className="mb-4">
            <label className="block text-gray-700 text-sm font-bold mb-2">
              Email
            </label>
            <input
              type="email"
              value={email}
              onChange={(e) => setEmail(e.target.value)}
              className="w-full px-3 py-2 border rounded-lg focus:outline-none focus:border-blue-500"
              required
            />
          </div>
          
          <div className="mb-6">
            <label className="block text-gray-700 text-sm font-bold mb-2">
              Password
            </label>
            <input
              type="password"
              value={password}
              onChange={(e) => setPassword(e.target.value)}
              className="w-full px-3 py-2 border rounded-lg focus:outline-none focus:border-blue-500"
              required
            />
          </div>
          
          <button
            type="submit"
            disabled={loading}
            className="w-full bg-blue-500 text-white py-2 rounded-lg hover:bg-blue-600 disabled:opacity-50"
          >
            {loading ? 'Logging in...' : 'Login'}
          </button>
        </form>
      </div>
    </div>
  );
};

export default Login;
```

### Step 6: Update App.jsx

```javascript
import { BrowserRouter, Routes, Route, Navigate } from 'react-router-dom';
import { AuthProvider } from './contexts/AuthContext';
import ProtectedRoute from './components/ProtectedRoute';
import Login from './components/Login';
import Dashboard from './components/Dashboard';
import ParentAccess from './components/ParentAccess';

function App() {
  return (
    <BrowserRouter>
      <AuthProvider>
        <Routes>
          <Route path="/login" element={<Login />} />
          
          <Route path="/dashboard" element={
            <ProtectedRoute>
              <Dashboard />
            </ProtectedRoute>
          } />
          
          <Route path="/parent-management" element={
            <ProtectedRoute requiredRole="editor">
              <ParentAccess />
            </ProtectedRoute>
          } />
          
          <Route path="/" element={<Navigate to="/dashboard" replace />} />
          <Route path="/unauthorized" element={<div>Unauthorized Access</div>} />
        </Routes>
      </AuthProvider>
    </BrowserRouter>
  );
}

export default App;
```

---

## User Roles

### Role Hierarchy

| Role | Level | Permissions |
|------|-------|-------------|
| **super_admin** | 4 | Full access to all features, user management |
| **admin** | 3 | Manage content, view analytics, limited user management |
| **editor** | 2 | Create, edit, delete content |
| **viewer** | 1 | Read-only access |

### Role Definitions

#### Super Admin
- Create/delete users
- Assign roles
- Access all modules
- System configuration

#### Admin
- Manage parent records
- View analytics
- Export data
- Moderate content

#### Editor
- Add/edit/delete parents
- Update locations
- Manage child information

#### Viewer
- View parent list
- View details
- Search functionality
- No edit permissions

---

## Security Rules

### Firestore Security Rules

Create `firestore.rules`:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    
    // Helper function to check if user is authenticated
    function isAuthenticated() {
      return request.auth != null;
    }
    
    // Helper function to get user role
    function getUserRole() {
      return get(/databases/$(database)/documents/users/$(request.auth.uid)).data.role;
    }
    
    // Helper function to check role level
    function hasRole(role) {
      let roles = {
        'viewer': 1,
        'editor': 2,
        'admin': 3,
        'super_admin': 4
      };
      let userRole = getUserRole();
      return roles[userRole] >= roles[role];
    }
    
    // Users collection
    match /users/{userId} {
      // Users can read their own document
      allow read: if isAuthenticated() && request.auth.uid == userId;
      
      // Only super_admin can create/update/delete users
      allow create, update, delete: if isAuthenticated() && hasRole('super_admin');
    }
    
    // Parents collection
    match /parents/{parentId} {
      // All authenticated users can read
      allow read: if isAuthenticated();
      
      // Editors and above can create/update/delete
      allow create, update, delete: if isAuthenticated() && hasRole('editor');
    }
    
    // Analytics collection (example)
    match /analytics/{docId} {
      // Admins and above can read
      allow read: if isAuthenticated() && hasRole('admin');
      
      // Only super_admin can write
      allow write: if isAuthenticated() && hasRole('super_admin');
    }
  }
}
```

---

## Usage Examples

### Example 1: Role-Based Component Rendering

```javascript
import { useAuth } from '../contexts/AuthContext';

const ParentManagement = () => {
  const { hasRole } = useAuth();

  return (
    <div>
      <h1>Parent Management</h1>
      
      {/* Show to all authenticated users */}
      <ParentList />
      
      {/* Show only to editors and above */}
      {hasRole('editor') && (
        <button>Add New Parent</button>
      )}
      
      {/* Show only to admins and above */}
      {hasRole('admin') && (
        <button>Export Data</button>
      )}
      
      {/* Show only to super_admin */}
      {hasRole('super_admin') && (
        <button>Delete All</button>
      )}
    </div>
  );
};
```

### Example 2: Creating Users with Roles

```javascript
// Admin panel - Create new user
const createNewUser = async (email, password, role) => {
  try {
    const { signup } = useAuth();
    await signup(email, password, role);
    console.log('User created successfully');
  } catch (error) {
    console.error('Error creating user:', error);
  }
};

// Usage
createNewUser('editor@example.com', 'password123', 'editor');
```

### Example 3: Conditional Navigation

```javascript
import { useAuth } from '../contexts/AuthContext';

const Sidebar = () => {
  const { hasRole } = useAuth();

  return (
    <nav>
      <Link to="/dashboard">Dashboard</Link>
      
      {hasRole('viewer') && (
        <Link to="/search">Search</Link>
      )}
      
      {hasRole('editor') && (
        <Link to="/parent-management">Parent Management</Link>
      )}
      
      {hasRole('admin') && (
        <Link to="/analytics">Analytics</Link>
      )}
      
      {hasRole('super_admin') && (
        <Link to="/user-management">User Management</Link>
      )}
    </nav>
  );
};
```

---

## Testing

### Test Users Setup

Create test users in Firebase Console:

```javascript
// Test users for different roles
const testUsers = [
  {
    email: 'superadmin@test.com',
    password: 'Test@123',
    role: 'super_admin'
  },
  {
    email: 'admin@test.com',
    password: 'Test@123',
    role: 'admin'
  },
  {
    email: 'editor@test.com',
    password: 'Test@123',
    role: 'editor'
  },
  {
    email: 'viewer@test.com',
    password: 'Test@123',
    role: 'viewer'
  }
];
```

### Testing Checklist

- [ ] Super Admin can access all routes
- [ ] Admin cannot access user management
- [ ] Editor can create/edit/delete parents
- [ ] Viewer can only view data
- [ ] Unauthorized users redirected to login
- [ ] Role-based components render correctly
- [ ] Firestore rules prevent unauthorized access
- [ ] Session persists on page refresh
- [ ] Logout clears session properly

### Manual Testing Steps

1. **Login Test**
   - Try logging in with each test user
   - Verify correct redirect after login

2. **Route Protection Test**
   - Try accessing protected routes without login
   - Verify redirect to login page

3. **Role Permission Test**
   - Login as viewer, try to edit (should fail)
   - Login as editor, try to edit (should succeed)

4. **Firestore Rules Test**
   - Try to read/write data with different roles
   - Verify rules block unauthorized access

---

## Environment Variables

Create `.env` file:

```env
VITE_FIREBASE_API_KEY=your_api_key
VITE_FIREBASE_AUTH_DOMAIN=your_auth_domain
VITE_FIREBASE_PROJECT_ID=your_project_id
VITE_FIREBASE_STORAGE_BUCKET=your_storage_bucket
VITE_FIREBASE_MESSAGING_SENDER_ID=your_sender_id
VITE_FIREBASE_APP_ID=your_app_id
```

Update `firebase.js`:

```javascript
const firebaseConfig = {
  apiKey: import.meta.env.VITE_FIREBASE_API_KEY,
  authDomain: import.meta.env.VITE_FIREBASE_AUTH_DOMAIN,
  projectId: import.meta.env.VITE_FIREBASE_PROJECT_ID,
  storageBucket: import.meta.env.VITE_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: import.meta.env.VITE_FIREBASE_MESSAGING_SENDER_ID,
  appId: import.meta.env.VITE_FIREBASE_APP_ID
};
```

---

## Best Practices

1. **Never store sensitive data in client-side code**
2. **Always validate roles on the backend (Firestore rules)**
3. **Use environment variables for Firebase config**
4. **Implement proper error handling**
5. **Add loading states during authentication**
6. **Clear sensitive data on logout**
7. **Use HTTPS in production**
8. **Implement rate limiting for login attempts**
9. **Add email verification for new users**
10. **Regular security audits**

---

## Troubleshooting

### Common Issues

**Issue: "Firebase not initialized"**
- Solution: Ensure firebase.js is imported before use

**Issue: "Permission denied"**
- Solution: Check Firestore security rules

**Issue: "User role not loading"**
- Solution: Verify user document exists in Firestore

**Issue: "Redirect loop"**
- Solution: Check ProtectedRoute logic and default routes

---

## Additional Resources

- [Firebase Authentication Docs](https://firebase.google.com/docs/auth)
- [Firestore Security Rules](https://firebase.google.com/docs/firestore/security/get-started)
- [React Context API](https://react.dev/reference/react/useContext)
- [React Router Protected Routes](https://reactrouter.com/en/main)

---

## Support

For issues or questions:
- Email: support@hope3.com
- Documentation: [Internal Wiki]
- Slack: #admin-portal-support

---

**Last Updated:** 2024
**Version:** 1.0.0
**Author:** Development Team
