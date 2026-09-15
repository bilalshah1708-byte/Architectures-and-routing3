{
  "name": "express-routing-demo",
  "version": "1.0.0",
  "description": "Express server demonstrating organized routing directories and query parameter handling",
  "main": "server.js",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "keywords": ["express", "routing", "query-parameters"],
  "license": "MIT",
  "dependencies": {
    "express": "^4.19.2"
  },
  "devDependencies": {
    "nodemon": "^3.1.4"
  }
}

const express = require('express');
const path = require('path');

// Route modules (each file lives in its own "routing directory")
const indexRoutes = require('./routes/index');
const usersRoutes = require('./routes/users');
const productsRoutes = require('./routes/products');

const app = express();
const PORT = process.env.PORT || 3000;

// ---------- Global middleware ----------
app.use(express.json());
app.use(express.urlencoded({ extended: true }));

// Simple request logger so you can see incoming query params in the console
app.use((req, res, next) => {
  const queryString = Object.keys(req.query).length
    ? ` | query: ${JSON.stringify(req.query)}`
    : '';
  console.log(`[${new Date().toISOString()}] ${req.method} ${req.path}${queryString}`);
  next();
});

// ---------- Mount routers under their own path prefixes ----------
// Each mounted router acts as its own "directory" of routes.
app.use('/', indexRoutes);
app.use('/api/users', usersRoutes);
app.use('/api/products', productsRoutes);

// ---------- 404 handler ----------
app.use((req, res) => {
  res.status(404).json({ error: 'Not found', path: req.originalUrl });
});

// ---------- Centralized error handler ----------
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.status || 500).json({ error: err.message || 'Internal server error' });
});

app.listen(PORT, () => {
  console.log(`Server running at http://localhost:${PORT}`);
});

module.exports = app;


const express = require('express');
const router = express.Router();

// GET /
// Simple landing route so you can confirm the server is up.
router.get('/', (req, res) => {
  res.json({
    message: 'Express routing + query parameter demo API',
    endpoints: {
      users: '/api/users?role=admin&limit=5&page=1',
      userById: '/api/users/1',
      products: '/api/products?category=electronics&minPrice=10&maxPrice=100&sort=price&order=asc',
    },
  });
});

// GET /health
// Common pattern: a lightweight health-check route.
router.get('/health', (req, res) => {
  res.json({ status: 'ok', uptime: process.uptime() });
});

module.exports = router;

const express = require('express');
const router = express.Router();

// In-memory sample data. Swap this out for a real database in a real app.
const USERS = [
  { id: 1, name: 'Alice Khan', role: 'admin', active: true },
  { id: 2, name: 'Bilal Ahmed', role: 'editor', active: true },
  { id: 3, name: 'Sana Malik', role: 'viewer', active: false },
  { id: 4, name: 'Usman Raza', role: 'editor', active: true },
  { id: 5, name: 'Hira Farooq', role: 'admin', active: false },
  { id: 6, name: 'Omar Sheikh', role: 'viewer', active: true },
];

// GET /api/users
// Supported query params:
//   role     - filter by exact role match      e.g. ?role=admin
//   active   - filter by active status          e.g. ?active=true
//   search   - case-insensitive name search      e.g. ?search=ali
//   page     - page number, default 1            e.g. ?page=2
//   limit    - results per page, default 10       e.g. ?limit=2
router.get('/', (req, res) => {
  let results = [...USERS];

  const { role, active, search } = req.query;

  if (role) {
    results = results.filter((u) => u.role.toLowerCase() === String(role).toLowerCase());
  }

  if (active !== undefined) {
    const wantActive = active === 'true';
    results = results.filter((u) => u.active === wantActive);
  }

  if (search) {
    const term = String(search).toLowerCase();
    results = results.filter((u) => u.name.toLowerCase().includes(term));
  }

  // Pagination
  const page = Math.max(parseInt(req.query.page, 10) || 1, 1);
  const limit = Math.max(parseInt(req.query.limit, 10) || 10, 1);
  const startIndex = (page - 1) * limit;
  const endIndex = startIndex + limit;
  const paginated = results.slice(startIndex, endIndex);

  res.json({
    total: results.length,
    page,
    limit,
    totalPages: Math.ceil(results.length / limit) || 1,
    data: paginated,
  });
});

// GET /api/users/:id
// Route parameter example, shown alongside query param routes for contrast.
router.get('/:id', (req, res) => {
  const user = USERS.find((u) => u.id === parseInt(req.params.id, 10));
  if (!user) {
    return res.status(404).json({ error: `User ${req.params.id} not found` });
  }
  res.json(user);
});

module.exports = router;

const express = require('express');
const router = express.Router();

const PRODUCTS = [
  { id: 1, name: 'Wireless Mouse', category: 'electronics', price: 25 },
  { id: 2, name: 'Mechanical Keyboard', category: 'electronics', price: 80 },
  { id: 3, name: 'Coffee Mug', category: 'kitchen', price: 12 },
  { id: 4, name: 'Desk Lamp', category: 'furniture', price: 45 },
  { id: 5, name: 'Bluetooth Speaker', category: 'electronics', price: 60 },
  { id: 6, name: 'Blender', category: 'kitchen', price: 90 },
];

// GET /api/products
// Supported query params:
//   category   - filter by category               e.g. ?category=electronics
//   minPrice   - minimum price (inclusive)          e.g. ?minPrice=20
//   maxPrice   - maximum price (inclusive)          e.g. ?maxPrice=70
//   sort       - field to sort by (name|price)       e.g. ?sort=price
//   order      - asc|desc, default asc                e.g. ?order=desc
router.get('/', (req, res) => {
  let results = [...PRODUCTS];
  const { category, minPrice, maxPrice, sort, order } = req.query;

  if (category) {
    results = results.filter((p) => p.category.toLowerCase() === String(category).toLowerCase());
  }

  if (minPrice !== undefined) {
    const min = Number(minPrice);
    if (!Number.isNaN(min)) results = results.filter((p) => p.price >= min);
  }

  if (maxPrice !== undefined) {
    const max = Number(maxPrice);
    if (!Number.isNaN(max)) results = results.filter((p) => p.price <= max);
  }

  if (sort === 'name' || sort === 'price') {
    const direction = order === 'desc' ? -1 : 1;
    results.sort((a, b) => (a[sort] > b[sort] ? direction : a[sort] < b[sort] ? -direction : 0));
  }

  res.json({ total: results.length, data: results });
});

// GET /api/products/:id
router.get('/:id', (req, res) => {
  const product = PRODUCTS.find((p) => p.id === parseInt(req.params.id, 10));
  if (!product) {
    return res.status(404).json({ error: `Product ${req.params.id} not found` });
  }
  res.json(product);
});

module.exports = router;

