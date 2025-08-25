# ProtoDB Client/Server Refactoring Guide

This guide shows how to refactor your protodb application to build client and server independently, with the client using React + Vite and the server using Node.js without frontend dependencies.

## New Project Structure

```
protodb/
├── client/                 # React + Vite frontend
│   ├── src/
│   │   ├── components/
│   │   ├── hooks/
│   │   ├── services/
│   │   ├── App.jsx
│   │   └── main.jsx
│   ├── public/
│   ├── index.html
│   ├── package.json
│   ├── vite.config.js
│   └── .env
├── server/                 # Node.js backend
│   ├── src/
│   │   ├── controllers/
│   │   ├── models/
│   │   ├── routes/
│   │   ├── middleware/
│   │   ├── services/
│   │   └── app.js
│   ├── package.json
│   └── .env
├── shared/                 # Shared utilities/types
│   ├── types/
│   └── utils/
└── package.json           # Root package.json for scripts
```

## Client Package Configuration

### `client/package.json`
```json
{
  "name": "protodb-client",
  "private": true,
  "version": "0.0.1",
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "preview": "vite preview",
    "lint": "eslint . --ext js,jsx --report-unused-disable-directives --max-warnings 0"
  },
  "dependencies": {
    "react": "^18.2.0",
    "react-dom": "^18.2.0",
    "axios": "^1.6.0",
    "react-router-dom": "^6.8.0"
  },
  "devDependencies": {
    "@types/react": "^18.2.43",
    "@types/react-dom": "^18.2.17",
    "@vitejs/plugin-react": "^4.2.1",
    "eslint": "^8.55.0",
    "eslint-plugin-react": "^7.33.2",
    "eslint-plugin-react-hooks": "^4.6.0",
    "eslint-plugin-react-refresh": "^0.4.5",
    "vite": "^5.0.8"
  }
}
```

### `client/vite.config.js`
```javascript
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'

export default defineConfig({
  plugins: [react()],
  server: {
    port: 3000,
    proxy: {
      '/api': {
        target: 'http://localhost:5000',
        changeOrigin: true
      }
    }
  },
  build: {
    outDir: 'dist',
    sourcemap: true
  }
})
```

### `client/src/main.jsx`
```javascript
import React from 'react'
import ReactDOM from 'react-dom/client'
import App from './App.jsx'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>,
)
```

### `client/src/services/api.js`
```javascript
import axios from 'axios'

const API_BASE_URL = import.meta.env.VITE_API_URL || '/api'

const api = axios.create({
  baseURL: API_BASE_URL,
  headers: {
    'Content-Type': 'application/json',
  },
})

// Request interceptor
api.interceptors.request.use(
  (config) => {
    // Add auth token if available
    const token = localStorage.getItem('authToken')
    if (token) {
      config.headers.Authorization = `Bearer ${token}`
    }
    return config
  },
  (error) => Promise.reject(error)
)

// Response interceptor
api.interceptors.response.use(
  (response) => response,
  (error) => {
    if (error.response?.status === 401) {
      localStorage.removeItem('authToken')
      window.location.href = '/login'
    }
    return Promise.reject(error)
  }
)

export default api
```

## Server Package Configuration

### `server/package.json`
```json
{
  "name": "protodb-server",
  "version": "1.0.0",
  "type": "module",
  "main": "src/app.js",
  "scripts": {
    "dev": "nodemon src/app.js",
    "start": "node src/app.js",
    "test": "jest"
  },
  "dependencies": {
    "express": "^4.18.2",
    "cors": "^2.8.5",
    "helmet": "^7.1.0",
    "morgan": "^1.10.0",
    "dotenv": "^16.3.1",
    "express-rate-limit": "^7.1.5"
  },
  "devDependencies": {
    "nodemon": "^3.0.2",
    "jest": "^29.7.0",
    "supertest": "^6.3.3"
  }
}
```

### `server/src/app.js`
```javascript
import express from 'express'
import cors from 'cors'
import helmet from 'helmet'
import morgan from 'morgan'
import rateLimit from 'express-rate-limit'
import dotenv from 'dotenv'

dotenv.config()

const app = express()
const PORT = process.env.PORT || 5000

// Security middleware
app.use(helmet())
app.use(cors({
  origin: process.env.CLIENT_URL || 'http://localhost:3000',
  credentials: true
}))

// Rate limiting
const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100 // limit each IP to 100 requests per windowMs
})
app.use('/api', limiter)

// Logging
app.use(morgan('combined'))

// Body parsing
app.use(express.json({ limit: '10mb' }))
app.use(express.urlencoded({ extended: true }))

// Routes
import apiRoutes from './routes/index.js'
app.use('/api', apiRoutes)

// Error handling middleware
app.use((err, req, res, next) => {
  console.error(err.stack)
  res.status(500).json({
    error: 'Something went wrong!',
    ...(process.env.NODE_ENV === 'development' && { stack: err.stack })
  })
})

// 404 handler
app.use('*', (req, res) => {
  res.status(404).json({ error: 'Route not found' })
})

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`)
})

export default app
```

### `server/src/routes/index.js`
```javascript
import express from 'express'

const router = express.Router()

// Health check
router.get('/health', (req, res) => {
  res.json({ status: 'OK', timestamp: new Date().toISOString() })
})

// Your API routes here
router.get('/data', (req, res) => {
  // Your data logic
  res.json({ message: 'Data endpoint' })
})

router.post('/data', (req, res) => {
  // Your create logic
  res.json({ message: 'Data created' })
})

export default router
```

## Root Package Configuration

### `package.json` (root)
```json
{
  "name": "protodb",
  "version": "1.0.0",
  "private": true,
  "scripts": {
    "dev": "concurrently \"npm run dev:server\" \"npm run dev:client\"",
    "dev:client": "cd client && npm run dev",
    "dev:server": "cd server && npm run dev",
    "build": "npm run build:client && npm run build:server",
    "build:client": "cd client && npm run build",
    "build:server": "cd server && npm run build",
    "start": "cd server && npm start",
    "install:all": "npm install && cd client && npm install && cd ../server && npm install",
    "clean": "rm -rf client/dist server/dist client/node_modules server/node_modules node_modules"
  },
  "devDependencies": {
    "concurrently": "^8.2.2"
  }
}
```

## Migration Steps

### 1. **Separate Dependencies**
Move React, Vite, and frontend-specific dependencies to `client/package.json`
Move Express, database, and backend-specific dependencies to `server/package.json`

### 2. **Split Source Code**
```bash
# Create new structure
mkdir -p client/src server/src shared

# Move frontend code
mv src/components client/src/
mv src/App.* client/src/
mv public client/

# Move backend code  
mv src/routes server/src/
mv src/models server/src/
mv src/controllers server/src/
```

### 3. **Update Import Paths**
In client code, update API calls:
```javascript
// Before: direct function calls
import { getData } from '../server/api'

// After: HTTP requests
import api from './services/api'
const data = await api.get('/data')
```

### 4. **Environment Configuration**
**`client/.env`**
```
VITE_API_URL=http://localhost:5000/api
```

**`server/.env`**
```
PORT=5000
CLIENT_URL=http://localhost:3000
NODE_ENV=development
```

### 5. **Build Scripts**
The root `package.json` provides unified commands:
- `npm run dev` - Runs both client and server in development
- `npm run build` - Builds both applications
- `npm run install:all` - Installs all dependencies

### 6. **Production Deployment**
For production, build the client and serve it as static files from the server:
```javascript
// In server/src/app.js (production only)
if (process.env.NODE_ENV === 'production') {
  app.use(express.static(path.join(__dirname, '../../client/dist')))
  
  app.get('*', (req, res) => {
    res.sendFile(path.join(__dirname, '../../client/dist/index.html'))
  })
}
```

## Benefits of This Structure

1. **Independent Builds** - Client and server can be built and deployed separately
2. **Technology Isolation** - Frontend dependencies don't affect backend performance
3. **Scalability** - Each part can be scaled independently
4. **Development Flexibility** - Different teams can work on client/server independently
5. **Deployment Options** - Can deploy to different platforms (client to CDN, server to containers)

This structure maintains separation of concerns while providing convenient development scripts for local development.
