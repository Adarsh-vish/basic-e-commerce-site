# Luxe & Co. - Node.js / Express & SQL Backend Integration

This guide provides the complete backend code to power the Luxe & Co. e-commerce store with an **Express.js** REST API and a **SQLite** database (using `sqlite3` or `better-sqlite3`, or easily adaptable to PostgreSQL/MySQL via Sequelize or Prisma).

---

## Project Setup

1. Create a new folder on your computer named `luxe-backend`.
2. Open terminal inside that folder and run:
   ```bash
   npm init -y
   npm install express cors bcryptjs jsonwebtoken sqlite3 dotenv
   ```
3. Create a file named **`server.js`** and paste the code below into it.

---

## Backend Source Code (`server.js`)

```javascript
const express = require('express');
const cors = require('cors');
const sqlite3 = require('sqlite3').verbose();
const bcrypt = require('bcryptjs');
const jwt = require('jsonwebtoken');

const app = express();
const PORT = process.env.PORT || 5000;
const JWT_SECRET = 'luxe_super_secret_key_2026';

// Middleware
app.use(cors());
app.use(express.json());

// Initialize SQLite Database
const db = new sqlite3.Database('./luxe.db', (err) => {
    if (err) {
        console.error('Error opening database', err.message);
    } else {
        console.log('Connected to the SQLite database.');
        createTables();
    }
});

// Create Database Tables
function createTables() {
    db.run(`CREATE TABLE IF NOT EXISTS users (
        id INTEGER PRIMARY KEY AUTOINCREMENT,
        name TEXT NOT NULL,
        email TEXT UNIQUE NOT NULL,
        password TEXT NOT NULL,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )`);

    db.run(`CREATE TABLE IF NOT EXISTS products (
        id TEXT PRIMARY KEY,
        name TEXT NOT NULL,
        category TEXT NOT NULL,
        price REAL NOT NULL,
        image TEXT NOT NULL,
        description TEXT NOT NULL
    )`, () => {
        seedProducts();
    });

    db.run(`CREATE TABLE IF NOT EXISTS orders (
        id TEXT PRIMARY KEY,
        user_email TEXT NOT NULL,
        items TEXT NOT NULL,
        total REAL NOT NULL,
        status TEXT NOT NULL,
        created_at DATETIME DEFAULT CURRENT_TIMESTAMP
    )`);
}

// Seed initial products if table is empty
function seedProducts() {
    db.get("SELECT COUNT(*) as count FROM products", (err, row) => {
        if (row.count === 0) {
            const initialProducts = [
                ['p1', 'Apex Wireless ANC Headphones', 'electronics', 299.99, 'https://images.unsplash.com/photo-1505740420928-5e560c06d30e?w=600&auto=format&fit=crop&q=80', 'Immersive active noise cancellation with 40-hour battery life.'],
                ['p2', 'Chronos Minimalist Chronograph', 'accessories', 189.50, 'https://images.unsplash.com/photo-1523275335684-37898b6baf30?w=600&auto=format&fit=crop&q=80', 'Italian leather strap paired with surgical stainless steel.'],
                ['p3', 'Nomad Canvas Backpack', 'apparel', 120.00, 'https://images.unsplash.com/photo-1553062407-98eeb64c6a62?w=600&auto=format&fit=crop&q=80', 'Water-resistant waxed canvas with padded 16-inch laptop compartment.'],
                ['p4', 'Zenith Mechanical Smart Keyboard', 'electronics', 165.00, 'https://images.unsplash.com/photo-1587829741301-dc798b83add3?w=600&auto=format&fit=crop&q=80', 'Hot-swappable linear switches with custom RGB backlighting.'],
                ['p5', 'Cashmere Blend Minimalist Hoodie', 'apparel', 95.00, 'https://images.unsplash.com/photo-1556905055-8f358a7a47b2?w=600&auto=format&fit=crop&q=80', 'Ultra-soft ethically sourced cashmere blend.'],
                ['p6', 'Titanium Minimalist Cardholder', 'accessories', 45.00, 'https://images.unsplash.com/photo-1627123424574-724758594e93?w=600&auto=format&fit=crop&q=80', 'RFID-blocking aerospace grade titanium wallet.']
            ];

            const stmt = db.prepare("INSERT INTO products VALUES (?, ?, ?, ?, ?, ?)");
            initialProducts.forEach(p => stmt.run(p));
            stmt.finalize();
            console.log('Initial products seeded.');
        }
    });
}

// --- API ROUTES ---

// 1. Get All Products
app.get('/api/products', (req, res) => {
    db.all("SELECT * FROM products", [], (err, rows) => {
        if (err) return res.status(500).json({ error: err.message });
        res.json(rows);
    });
});

// 2. User Registration
app.post('/api/auth/register', async (req, res) => {
    const { name, email, password } = req.body;
    try {
        const hashedPassword = await bcrypt.hash(password, 10);
        db.run(
            `INSERT INTO users (name, email, password) VALUES (?, ?, ?)`,
            [name, email, hashedPassword],
            function(err) {
                if (err) {
                    return res.status(400).json({ error: 'Email already registered or invalid data.' });
                }
                const token = jwt.sign({ email, name }, JWT_SECRET, { expiresIn: '7d' });
                res.json({ token, user: { name, email } });
            }
        );
    } catch (e) {
        res.status(500).json({ error: 'Server error during registration.' });
    }
});

// 3. User Login
app.post('/api/auth/login', (req, res) => {
    const { email, password } = req.body;
    db.get(`SELECT * FROM users WHERE email = ?`, [email], async (err, user) => {
        if (err || !user) return res.status(400).json({ error: 'Invalid email or password.' });

        const isMatch = await bcrypt.compare(password, user.password);
        if (!isMatch) return res.status(400).json({ error: 'Invalid email or password.' });

        const token = jwt.sign({ email: user.email, name: user.name }, JWT_SECRET, { expiresIn: '7d' });
        res.json({ token, user: { name: user.name, email: user.email } });
    });
});

// 4. Create Order
app.post('/api/orders', (req, res) => {
    const { userEmail, items, total } = req.body;
    const orderId = 'ORD-' + Math.floor(100000 + Math.random() * 900000);
    const itemsJson = JSON.stringify(items);

    db.run(
        `INSERT INTO orders (id, user_email, items, total, status) VALUES (?, ?, ?, ?, ?)`,
        [orderId, userEmail, itemsJson, total, 'Processing'],
        function(err) {
            if (err) return res.status(500).json({ error: err.message });
            res.json({ success: true, orderId });
        }
    );
});

// 5. Get User Orders
app.get('/api/orders/:email', (req, res) => {
    const email = req.params.email;
    db.all(`SELECT * FROM orders WHERE user_email = ? ORDER BY created_at DESC`, [email], (err, rows) => {
        if (err) return res.status(500).json({ error: err.message });
        
        // Parse items JSON back to object
        const formattedOrders = rows.map(o => ({
            ...o,
            items: JSON.parse(o.items)
        }));
        res.json(formattedOrders);
    });
});

// Start Server
app.listen(PORT, () => {
    console.log(`Luxe & Co. backend running on http://localhost:${PORT}`);
});
```

---

## How to Connect Frontend to Backend

In your `index.html` frontend file, replace the local state functions (`loadFromLocalStorage`, `processOrder`, `handleAuthSubmit`, etc.) with `fetch()` calls pointing to `http://localhost:5000/api/...`.

For example, loading products on startup:
```javascript
async function fetchProducts() {
    try {
        const response = await fetch('http://localhost:5000/api/products');
        state.products = await response.json();
        renderProducts();
    } catch (e) {
        console.error('Failed to load products from backend', e);
    }
}