// ===== Backend: server.js =====
const express = require('express');
const jwt = require('jsonwebtoken');
const bodyParser = require('body-parser');
const cors = require('cors');

const app = express();
app.use(cors());
app.use(bodyParser.json());

const SECRET = "supersecret";

// Mock DB
let users = [
  { id: 1, email: "admin@saas.com", password: "admin123", role: "admin", plan: "enterprise" },
  { id: 2, email: "user@saas.com", password: "user123", role: "user", plan: "basic" }
];
let invoices = [
  { id: 1, userId: 2, amount: 10, status: "paid" },
  { id: 2, userId: 2, amount: 10, status: "due" }
];

// Auth middleware
function auth(role) {
  return (req, res, next) => {
    const token = req.headers.authorization?.split(" ")[1];
    if (!token) return res.status(401).send("No token");
    try {
      const decoded = jwt.verify(token, SECRET);
      req.user = decoded;
      if (role && decoded.role !== role) return res.status(403).send("Forbidden");
      next();
    } catch {
      res.status(401).send("Invalid token");
    }
  };
}

// Login
app.post("/login", (req, res) => {
  const { email, password } = req.body;
  const user = users.find(u => u.email === email && u.password === password);
  if (!user) return res.status(401).send("Invalid credentials");
  const token = jwt.sign({ id: user.id, role: user.role }, SECRET);
  res.json({ token, role: user.role, plan: user.plan });
});

// User invoices
app.get("/invoices", auth("user"), (req, res) => {
  const userInvoices = invoices.filter(i => i.userId === req.user.id);
  res.json(userInvoices);
});

// Admin: view all users
app.get("/users", auth("admin"), (req, res) => {
  res.json(users);
});

// Admin: add invoice
app.post("/invoice", auth("admin"), (req, res) => {
  const { userId, amount } = req.body;
  invoices.push({ id: invoices.length + 1, userId, amount, status: "due" });
  res.json({ message: "Invoice added" });
});

app.listen(4000, () => console.log("Server running on 4000"));


// ===== Frontend: App.js (React) =====
import React, { useState } from "react";
import axios from "axios";

function App() {
  const [token, setToken] = useState(null);
  const [role, setRole] = useState(null);
  const [data, setData] = useState([]);

  const login = async (email, password) => {
    const res = await axios.post("http://localhost:4000/login", { email, password });
    setToken(res.data.token);
    setRole(res.data.role);
  };

  const fetchInvoices = async () => {
    const res = await axios.get("http://localhost:4000/invoices", {
      headers: { Authorization: `Bearer ${token}` }
    });
    setData(res.data);
  };

  const fetchUsers = async () => {
    const res = await axios.get("http://localhost:4000/users", {
      headers: { Authorization: `Bearer ${token}` }
    });
    setData(res.data);
  };

  return (
    <div>
      {!token ? (
        <div>
          <h2>Login</h2>
          <button onClick={() => login("admin@saas.com", "admin123")}>Login as Admin</button>
          <button onClick={() => login("user@saas.com", "user123")}>Login as User</button>
        </div>
      ) : (
        <div>
          <h2>Dashboard ({role})</h2>
          {role === "user" && (
            <button onClick={fetchInvoices}>View My Invoices</button>
          )}
          {role === "admin" && (
            <button onClick={fetchUsers}>View All Users</button>
          )}
          <pre>{JSON.stringify(data, null, 2)}</pre>
        </div>
      )}
    </div>
  );
}


export default App;
# role-based-SaasS-billing-portal
