//APP.CSS

@tailwind base;
@tailwind components;
@tailwind utilities;

/* Custom global scrollbar polish to match your modern slate layout */
body {
  @apply bg-slate-50 text-slate-800 antialiased;
}

::-webkit-scrollbar {
  width: 6px;
  height: 6px;
}

::-webkit-scrollbar-track {
  @apply bg-slate-100;
}

::-webkit-scrollbar-thumb {
  @apply bg-slate-300 rounded-full hover:bg-slate-400 transition-colors;
}
_____________________________________________________________________________________________________________________________________
//POSTCSS.CONFIG.JS

module.exports = {
  plugins: {
    tailwindcss: {},
    autoprefixer: {},
  },
}
_____________________________________________________________________________________________________________________________________
//TAILWIND.CONFIG.JS

/** @type {import('tailwindcss').Config} */
module.exports = {
  content: [
    "index.html",
    "./src/**/*.{js,jsx,ts,tsx}", 
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
_____________________________________________________________________________________________________________________________________
// server.js
const express = require('express');
const mysql = require('mysql2');
const cors = require('cors');
const session = require('express-session');
const bcrypt = require('bcryptjs');

const app = express();
const PORT = 5000;

app.use(cors({ origin: 'http://localhost:3000', credentials: true }));
app.use(express.json());

app.use(session({
    secret: 'secret_key',
    resave: false,
    saveUninitialized: false,
    cookie: { secure: false, httpOnly: true, maxAge: 1000 * 60 * 60 * 24, sameSite: 'lax' }
}));

const db = mysql.createConnection({
    host: 'localhost',
    user: 'root',
    password: '',
    database: 'y_bus_booking' 
});

db.connect((err) => {
    if (err) {
        console.error('❌ Database offline:', err.message);
    }else{
        console.log('✅ Database connected.');
    }
});

// ==========================================
// BACKGROUND AUTOMATION: EXPIRED SCHEDULES
// ==========================================
const checkExpiredSchedules = () => {
    const query = 'UPDATE yk_schedules SET ScheduleStatus = "Inactive" WHERE DepartureTime = NOW() AND ScheduleStatus = "Active"';
    db.query(query);
};
setInterval(checkExpiredSchedules, 60000); // Automatically runs every 60 seconds

// ==========================================
// AUTHENTICATION ENDPOINTS
// ==========================================
app.post('/login', (req, res) => {
    const { username, password } = req.body;
    const query = 'SELECT * FROM yk_users WHERE UserName = ?';
    db.query(query, [username], async (err, results) => {
        if (err || results.length === 0) return res.status(401).json({ message: 'User not found.' });
        const user = results[0];
        const match = await bcrypt.compare(password, user.Password);
        if (!match) return res.status(401).json({ message: 'Invalid credentials.' });

        req.session.user = { id: user.UserID, username: user.UserName, role: user.UserRole };
        res.json({ success: true, message: 'Login successful.', role: user.UserRole });
    });
});

app.post('/register-customer', async (req, res) => {
    const { username, password } = req.body;
    const hashedPassword = await bcrypt.hash(password, 10);
    const query = 'INSERT INTO yk_users (UserName, Password, UserRole) VALUES (?, ?, "customer")';
    db.query(query, [username, hashedPassword], (err) => {
        if (err) return res.status(500).json({ message: 'Failed to create customer.' });
        res.status(201).json({ message: 'Customer registered successfully.' });
    });
});

app.get('/get-customers', (req, res) => {
    db.query('SELECT UserID, UserName, UserRole FROM yk_users WHERE UserRole = "customer"', (err, results) => {
        res.json(results);
    });
});

app.post('/logout', (req, res) => {
    req.session.destroy(() => { res.clearCookie('connect.sid').json({ message: 'Logged out.' }); });
});

// ==========================================
// BUS MANAGEMENT ENDPOINTS (yk_buses)
// ==========================================
app.get('/buses', (req, res) => {
    db.query('SELECT * FROM yk_buses', (err, results) => res.json(results));
});

app.post('/buses', (req, res) => {
    const { plate, seats, bustype } = req.body;
    db.query('INSERT INTO yk_buses (PlateNumber, TotalSeats, BusType) VALUES (?, ?, ?)', [plate, seats, bustype], () => res.json({ message: 'Bus added.' }));
});

app.put('/buses/:id', (req, res) => {
    const { plate, seats, bustype } = req.body;
    db.query('UPDATE yk_buses SET PlateNumber = ?, TotalSeats = ?, BusType = ? WHERE BusID = ?', [plate, seats, bustype, req.params.id], () => res.json({ message: 'Bus updated.' }));
});

app.delete('/buses/:id', (req, res) => {
    db.query('DELETE FROM yk_buses WHERE BusID = ?', [req.params.id], () => res.json({ message: 'Bus deleted.' }));
});

// ==========================================
// SCHEDULE MANAGEMENT ENDPOINTS (yk_schedules)
// ==========================================
app.get('/schedules', (req, res) => {
    db.query('SELECT * FROM yk_schedules', (err, results) => res.json(results));
});

app.post('/schedules', (req, res) => {
    const { busId, route, departure, destination, depTime, arrTime, price, status } = req.body;
    const query = 'INSERT INTO yk_schedules (BusID, RouteName, DeparturePoint, Destination, DepartureTime, EstimatedArrivalTime, TicketPrice, ScheduleStatus) VALUES (?, ?, ?, ?, ?, ?, ?, ?)';
    db.query(query, [busId, route, departure, destination, depTime, arrTime, price, status], () => res.json({ message: 'Schedule added.' }));
});

app.put('/schedules/:id', (req, res) => {
    const { busId, route, departure, destination, depTime, arrTime, price, status } = req.body;
    const query = 'UPDATE yk_schedules SET BusID = ?, RouteName = ?, DeparturePoint = ?, Destination = ?, DepartureTime = ?, EstimatedArrivalTime = ?, TicketPrice = ?, ScheduleStatus = ? WHERE ScheduleID = ?';
    db.query(query, [busId, route, departure, destination, depTime, arrTime, price, status, req.params.id], () => res.json({ message: 'Schedule updated.' }));
});

app.delete('/schedules/:id', (req, res) => {
    db.query('DELETE FROM yk_schedules WHERE ScheduleID = ?', [req.params.id], () => res.json({ message: 'Schedule deleted.' }));
});

// ==========================================
// BOOKING MANAGEMENT ENGINE (yk_bookings) WITH SEAT REDUCTION RULES
// ==========================================
app.get('/bookings', (req, res) => {
    db.query('SELECT * FROM yk_bookings', (err, results) => res.json(results));
});

// ==========================================
// BOOKING MANAGEMENT ENGINE WITH SEAT BLOCKING
// ==========================================
app.post('/bookings', (req, res) => {
    const { scheduleId, passengerName, gender, phone, seatNumber, paymentStatus } = req.body;
    const userId = req.session.user ? req.session.user.id : 1; 

    // RULE STEP 1: Check if this specific seat number is already reserved for this trip schedule
    const checkSeatQuery = 'SELECT * FROM yk_bookings WHERE ScheduleID = ? AND SeatNumber = ?';
    
    db.query(checkSeatQuery, [scheduleId, seatNumber], (err, seatCheckResults) => {
        if (err) return res.status(500).json({ message: 'Database query error.' });
        
        // If a matching row is found, stop execution immediately!
        if (seatCheckResults.length > 0) {
            return res.status(400).json({ message: `Seat number ${seatNumber} has already been reserved by another passenger. Please pick a different seat.` });
        }

        // RULE STEP 2: Find the bus mapped to this schedule to check overall seating capacity
        db.query('SELECT BusID FROM yk_schedules WHERE ScheduleID = ?', [scheduleId], (err, schedRes) => {
            if (schedRes.length === 0) return res.status(404).json({ message: 'Schedule not found.' });
            const busId = schedRes[0].BusID;

            db.query('SELECT TotalSeats FROM yk_buses WHERE BusID = ?', [busId], (err, busRes) => {
                const currentSeats = busRes[0].TotalSeats;

                if (currentSeats <= 0) {
                    return res.status(400).json({ message: 'This bus is completely full!' });
                }

                // RULE STEP 3: Write the booking entry since the seat is safe and open
                const bookQuery = 'INSERT INTO yk_bookings (ScheduleID, UserID, PassengerName, PassengerGender, PassengerPhone, SeatNumber, PaymentStatus, BookingDate) VALUES (?, ?, ?, ?, ?, ?, ?, NOW())';
                db.query(bookQuery, [scheduleId, userId, passengerName, gender, phone, seatNumber, paymentStatus], () => {
                    
                    const newSeats = currentSeats - 1;

                    if (newSeats === 0) {
                        // Bus full? Delete the schedule immediately!
                        db.query('DELETE FROM yk_schedules WHERE ScheduleID = ?', [scheduleId], () => {
                            res.json({ message: `Booking successful! Seat ${seatNumber} is yours. This bus is now full; schedule removed.` });
                        });
                    } else {
                        // Reduce seat count by 1
                        db.query('UPDATE yk_buses SET TotalSeats = ? WHERE BusID = ?', [newSeats, busId], () => {
                            res.json({ message: `Booking successful! Seat ${seatNumber} has been reserved.` });
                        });
                    }
                });
            });
        });
    });
});

app.delete('/bookings/:id', (req, res) => {
    db.query('DELETE FROM yk_bookings WHERE BookingID = ?', [req.params.id], () => res.json({ message: 'Booking deleted.' }));
});

// ==========================================
// REPORT SYSTEM GENERATOR DATA VECTORS
// ==========================================
app.get('/report/agent', (req, res) => {
    const query = `
        SELECT s.ScheduleID, s.RouteName, s.DepartureTime, s.TicketPrice, b.PlateNumber,
        (SELECT COUNT(*) FROM yk_bookings WHERE ScheduleID = s.ScheduleID) as TotalBookings
        FROM yk_schedules s
        JOIN yk_buses b ON s.BusID = b.BusID`;
    db.query(query, (err, results) => res.json(results));
});

app.get('/report/customer', (req, res) => {
    // We remove the UserID check entirely so it pulls every row in the table
    const query = `
        SELECT bk.BookingID, bk.PassengerName, bk.SeatNumber, bk.PaymentStatus, sch.RouteName, sch.DepartureTime, sch.TicketPrice
        FROM yk_bookings bk
        JOIN yk_schedules sch ON bk.ScheduleID = sch.ScheduleID`;
        
    db.query(query, (err, results) => {
        if (err) return res.status(500).json({ message: 'Error generating report.' });
        res.json(results);
    });
});

app.listen(PORT, () => {console.log(`Active server execution node on port ${PORT}`)});

_______________________________________________________________________________________________________________________________________
//2.generate.js
const bcrypt = require('bcryptjs');

async function hashMyPassword() {
    const password = "admin123";
    
    // Add 'await' here to wait for the actual string result
    const hashedPassword = await bcrypt.hash(password, 10); 

    console.log("Password:" + password + " Hashed Password: " + hashedPassword); 
}

hashMyPassword();
_______________________________________________________________________________________________________________________________________
//FRONTEND
_______________________________________________________________________________________________________________________________________
//NAVBAR.JSX

import React from 'react';
import { Link, useNavigate } from 'react-router-dom';
import api from '../api/config';

export function AgentNavbar() {
  const navigate = useNavigate();
  const logout = async () => { 
    await api.post('/logout'); navigate('/login'); 
  };
  return (
    <nav className="bg-green-600 p-4 text-white flex justify-between items-center shadow-md font-sans mb-6">
      <span className="font-bold text-xl">Agent</span>
      <div className="flex gap-6 items-center">
        <Link to="/agent" className="hover:underline font-medium">Customers</Link>
        <Link to="/manage-buses" className="hover:underline font-medium">Buses</Link>
        <Link to="/manage-schedules" className="hover:underline font-medium">Schedules</Link>
        <Link to="/agent-report" className="hover:underline font-medium">Reports</Link>
        <button onClick={logout} className="bg-red-500 px-3 py-1.5 rounded font-bold hover:bg-red-600 transition">Logout</button>
      </div>
    </nav>
  );
}

export function CustomerNavbar() {
  const navigate = useNavigate();
  const logout = async () => { await api.post('/logout'); navigate('/login'); };
  return (
    <nav className="bg-emerald-600 p-4 text-white flex justify-between items-center shadow-md font-sans mb-6">
      <span className="font-bold text-xl">Passenger</span>
      <div className="flex gap-6 items-center">
        <Link to="/customer" className="hover:underline font-medium">Book Tickets</Link>
        <Link to="/customer-report" className="hover:underline font-medium">Tickets Report</Link>
        <button onClick={logout} className="bg-red-500 px-3 py-1.5 rounded font-bold hover:bg-red-600 transition">Logout</button>
      </div>
    </nav>
  );
}

_______________________________________________________________________________________________________________________________________
//CONFIG.JS

import axios from 'axios';

// Centralized Axios Configuration for Swift Wheels
const api = axios.create({
    baseURL: 'http://localhost:5000', // Maps to your backend Express engine port
    timeout: 10000,                       // 10-second automatic request timeout window
    headers: {
    
        'Content-Type': 'application/json'
    },
    withCredentials: true                 // CRITICAL: Forces browser cookies to pass on cross-origin requests
});

export default api;

_______________________________________________________________________________________________________________________________________
//APP.JSX

import React from 'react';
import { BrowserRouter as Router, Routes, Route, Navigate } from 'react-router-dom';

import Login from './pages/Login';
import AgentDashboard from './pages/AgentDashboard';
import CustomerDashboard from './pages/CustomerDashboard';
import BusManagement from './pages/BusManagement';
import ScheduleManagement from './pages/ScheduleManagement';
import AgentReport from './pages/AgentReport';
import CustomerReport from './pages/CustomerReport';

export default function App() {
  return (
    <Router>
      <Routes>
        <Route path="/" element={<Login />} />
        <Route path="/login" element={<Login />} />
        <Route path="/agent" element={<AgentDashboard />} />
        <Route path="/customer" element={<CustomerDashboard />} />
        <Route path="/manage-buses" element={<BusManagement />} />
        <Route path="/manage-schedules" element={<ScheduleManagement />} />
        <Route path="/agent-report" element={<AgentReport />} />
        <Route path="/customer-report" element={<CustomerReport />} />
        <Route path="*" element={<Navigate to="/login" replace />} />
      </Routes>
    </Router>
  );
}

_______________________________________________________________________________________________________________________________________
//LOGIN.JSX

import React, { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import api from '../api/config';

const Login = () => {
  const [data, setData] = useState({ username: '', password: '' });
  const navigate = useNavigate();

  const handleChange = (e) => {
    setData({ ...data, [e.target.name]: e.target.value });  
  };

  const handleSubmit = async (e) => {
    e.preventDefault();
    try {
      const response = await api.post('/login', data);
      
      if (response.data.message === 'Login successful.') {
        alert(`Logged in as ${response.data.role}`);
        if (response.data.role === 'agent') navigate('/agent');
        else navigate('/customer');
      } else {
        alert(response.data.message || 'Invalid credentials');
      }
    } catch (error) {
      alert('Failed');
    }
  };

  return (
    <div className="min-h-screen bg-green-50 flex items-center justify-center">
      <div className="bg-white p-8 rounded-xl shadow-md w-96 border border-green-200">
        <h1 className="text-3xl font-bold text-center text-green-700 mb-6">Sign In</h1>
        
        <form onSubmit={handleSubmit} className="flex flex-col">
          <label className="font-semibold mb-1 text-green-900">Username:</label>
          <input type="text" name="username" placeholder="Username..." onChange={handleChange} className="border border-green-300 p-2 rounded mb-4 outline-none" required />

          <label className="font-semibold mb-1 text-green-900">Password:</label>
          <input type="password" name="password" placeholder="Password..." onChange={handleChange} className="border border-green-300 p-2 rounded mb-6 outline-none" required />

          <button type="submit" className="bg-green-600 text-white py-2 px-4 rounded font-bold hover:bg-green-700 transition">Log In</button>
        </form>
      </div>
    </div>
  );
};

export default Login;

______________________________________________________________________________________________________________________________________
//AGENTDASHBOARD.JSX

import React, { useState, useEffect } from 'react';
import api from '../api/config';
import { AgentNavbar } from '../components/Navbar'; // Added the shared navbar link

const AgentDashboard = () => {
  const [customerData, setCustomerData] = useState({ username: '', password: '' });
  const [userList, setUserList] = useState([]); 

  // Fetch all registered customers from the database
  const fetchCustomers = async () => {
    try {
      const response = await api.get('/get-customers');
      setUserList(response.data);
    } catch (error) {
      console.error('Failed to load user data');
    }
  };

  useEffect(() => {
    fetchCustomers();
  }, []);

  const handleInputChange = (e) => {
    setCustomerData({ ...customerData, [e.target.name]: e.target.value });
  };

  const handleRegisterCustomer = async (e) => {
    e.preventDefault();
    try {
      const response = await api.post('/register-customer', customerData);
      if (response.data.message === 'Customer registered successfully.') {
        alert('Customer Created Successfully!');
        setCustomerData({ username: '', password: '' });
        fetchCustomers(); // Automatically loop and update the table using .map()
      }
    } catch (error) {
      alert('Failed to register customer.');
    }
  };

  return (
    <div className="min-h-screen bg-green-50">
      {/* Universal Top Navigation Header Bar */}
      <AgentNavbar />

      {/* Main Form and Table Split Layout Screen Area */}
      <div className="p-8 flex flex-col lg:flex-row gap-8 items-start font-sans">
        
        {/* LEFT CARD: Registration Form Inputs */}
        <div className="bg-white p-6 rounded-xl shadow-md w-full lg:w-96 border border-green-200">
          <h3 className="text-xl font-bold text-green-700 mb-4">Register New Customer</h3>
          <form onSubmit={handleRegisterCustomer} className="flex flex-col">
            <label className="font-semibold mb-1 text-green-900">Username:</label>
            <input 
              type="text" 
              name="username" 
              value={customerData.username} 
              onChange={handleInputChange} 
              className="border border-green-300 p-2 rounded mb-4 outline-none" 
              required 
            />

            <label className="font-semibold mb-1 text-green-900">Password:</label>
            <input 
              type="password" 
              name="password" 
              value={customerData.password} 
              onChange={handleInputChange} 
              className="border border-green-300 p-2 rounded mb-6 outline-none" 
              required 
            />

            <button type="submit" className="bg-green-600 text-white py-2 px-4 rounded font-bold hover:bg-green-700 transition">
              Add Account
            </button>
          </form>
        </div>

        {/* RIGHT CARD: Live Customer Database Row View using .map() */}
        <div className="bg-white p-6 rounded-xl shadow-md flex-1 w-full border border-green-200">
          <h3 className="text-xl font-bold text-green-700 mb-4">Registered Customers</h3>
          <table className="w-full border-collapse border border-green-200 text-left">
            <thead>
              <tr className="bg-green-100 text-green-800">
                <th className="border border-green-200 p-3">Username</th>
                <th className="border border-green-200 p-3">User Role</th>
              </tr>
            </thead>
            <tbody>
              {userList.map((user, index) => (
                <tr key={index} className="hover:bg-green-50 transition text-emerald-700">
                  <td className="border border-green-200 p-3">{user.UserName}</td>
                  <td className="border border-green-200 p-3 text-green-600 font-semibold">{user.UserRole}</td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
      </div>
    </div>
  );
};

export default AgentDashboard;

________________________________________________________________________________________________________________________________________
//AGENTREPORT.JSX
import React, { useState, useEffect } from 'react';
import api from '../api/config';
import { AgentNavbar } from '../components/Navbar';

const AgentReport = () => {
  const [reportData, setReportData] = useState([]);
  useEffect(() => {
    api.get('/report/agent').then(res => setReportData(res.data));
  }, []);

  return (
    <div className="min-h-screen bg-green-50">
      <AgentNavbar />
      <div className="p-8 font-sans">
        <div className="bg-white p-6 rounded-xl shadow-md border border-green-200">
          <h2 className="text-2xl font-bold text-green-800 mb-6">Report</h2>
            </div>
            <button onClick={() => window.print()} className="bg-green-600 text-white font-bold px-4 py-2 rounded-lg hover:bg-green-700 print:hidden shadow">Print Report</button>
          </div>
          <table className="w-full border-collapse border border-green-200 text-left">
            <thead>
              <tr className="bg-green-100 text-green-800">
                <th className="border border-green-200 p-3">Route Name</th>
                <th className="border border-green-200 p-3">Assigned Bus Plate</th>
                <th className="border border-green-200 p-3">Departure Date/Time</th>
                <th className="border border-green-200 p-3">Ticket Base Fare</th>
                <th className="border border-green-200 p-3">Total Sold Tickets</th>
              </tr>
            </thead>
            <tbody>
              {reportData.map((row, index) => (
                <tr key={index} className="hover:bg-green-50 text-gray-700">
                  <td className="border border-green-200 p-3 font-semibold">{row.RouteName}</td>
                  <td className="border border-green-200 p-3">{row.PlateNumber}</td>
                  <td className="border border-green-200 p-3">{row.DepartureTime}</td>
                  <td className="border border-green-200 p-3 text-green-600 font-bold">{row.TicketPrice}</td>
                  <td className="border border-green-200 p-3 font-bold text-center bg-green-50/50">{row.TotalBookings} seats</td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
  );
};

export default AgentReport;

________________________________________________________________________________________________________________________________________
// BUS MANAGEMENT.JSX

import React, { useState, useEffect } from 'react';
import api from '../api/config';
import { AgentNavbar } from '../components/Navbar';

const BusManagement = () => {
  const [buses, setBuses] = useState([]);
  const [form, setForm] = useState({ plate: '', seats: '', bustype: '' });
  const [editingId, setEditingId] = useState(null);

  const fetchBuses = async () => { const res = await api.get('/buses'); setBuses(res.data); };
  useEffect(() => { 
    fetchBuses(); 
  }, []);

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (editingId) {
      await api.put(`/buses/${editingId}`, form);
      setEditingId(null);
    } else {
      await api.post('/buses', form);
    }
    setForm({ plate: '', seats: '', bustype: '' });
    fetchBuses();
  };

  const handleEdit = (bus) => {
    setEditingId(bus.BusID);
    setForm({ plate: bus.PlateNumber, seats: bus.TotalSeats, bustype: bus.BusType });
  };

  const handleDelete = async (id) => {
    await api.delete(`/buses/${id}`);
    fetchBuses();
  };

  return (
    <div className="min-h-screen bg-green-50">
      <AgentNavbar />
      <div className="p-8 flex flex-col lg:flex-row gap-8 items-start font-sans">
        
        {/* Form */}
        <div className="bg-white p-6 rounded-xl shadow-md w-full lg:w-96 border border-green-200">
          <h3 className="text-xl font-bold text-green-700 mb-4">{editingId ? 'Update Bus' : 'Add New Bus'}</h3>
          <form onSubmit={handleSubmit} className="flex flex-col">
            <label className="font-semibold mb-1">Plate Number:</label>
            <input type="text" value={form.plate} onChange={(e) => setForm({...form, plate: e.target.value})} className="border border-green-300 p-2 rounded mb-4 outline-none" required />

            <label className="font-semibold mb-1">Total Seats:</label>
            <input type="number" value={form.seats} onChange={(e) => setForm({...form, seats: e.target.value})} className="border border-green-300 p-2 rounded mb-4 outline-none" required />

            <label className="font-semibold mb-1">Bus Type:</label>
                <select 
              value={form.bustype} 
              onChange={(e) => setForm({...form, bustype: e.target.value})} 
              className="border border-green-300 p-2 rounded mb-4 bg-white outline-none" 
              required
            >
              <option value="">-- Choose Bus Type --</option>
              <option value="Minibus">Coach Minibus</option>
              <option value="Coaster">Coaster</option> 
              <option value="Double Decker">Double Decker</option>
            </select>
            <button type="submit" className="bg-green-600 text-white py-2 rounded font-bold hover:bg-green-700">{editingId ? 'Save Updates' : 'Add Bus'}</button>
          </form>
        </div>

        {/* Display Table using .map() */}
        <div className="bg-white p-6 rounded-xl shadow-md flex-1 w-full border border-green-200">
          <h3 className="text-xl font-bold text-green-700 mb-4">Bus Fleet Registry</h3>
          <table className="w-full border-collapse border border-green-200 text-left">
            <thead>
              <tr className="bg-green-100 text-green-800">
                <th className="border border-green-200 p-3">Plate Number</th>
                <th className="border border-green-200 p-3">Total Seats</th>
                <th className="border border-green-200 p-3">Bus Type</th>
                <th className="border border-green-200 p-3 text-center">Actions</th>
              </tr>
            </thead>
            <tbody>
              {buses.map((bus) => (
                <tr key={bus.BusID} className="hover:bg-green-50 text-gray-700">
                  <td className="border border-green-200 p-3">{bus.PlateNumber}</td>
                  <td className="border border-green-200 p-3">{bus.TotalSeats}</td>
                  <td className="border border-green-200 p-3">{bus.BusType}</td>
                  <td className="border border-green-200 p-3 flex gap-2 justify-center">
                    <button onClick={() => handleEdit(bus)} className="bg-amber-500 text-white px-3 py-1 rounded font-bold hover:bg-amber-600 text-sm">Edit</button>
                    <button onClick={() => handleDelete(bus.BusID)} className="bg-red-500 text-white px-3 py-1 rounded font-bold hover:bg-red-600 text-sm">Delete</button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>

      </div>
    </div>
  );
};

export default BusManagement;
________________________________________________________________________________________________________________________________________
//CUSTOMERDASHBOARD.JSX

import React, { useState, useEffect } from 'react';
import api from '../api/config';
import { CustomerNavbar } from '../components/Navbar';

const CustomerDashboard = () => {
  const [schedules, setSchedules] = useState([]);
  const [bookings, setBookings] = useState([]);
  const [form, setForm] = useState({ scheduleId: '', passengerName: '', gender: 'Male', phone: '', seatNumber: '', paymentStatus: 'Paid' });

  const loadPageData = async () => {
    const sRes = await api.get('/schedules'); setSchedules(sRes.data);
    const bRes = await api.get('/bookings'); setBookings(bRes.data);
  };
  useEffect(() => { loadPageData(); }, []);

  const handleBooking = async (e) => {
    e.preventDefault();
    try {
      const res = await api.post('/bookings', form);
      alert(res.data.message);
      setForm({ scheduleId: '', passengerName: '', gender: 'Male', phone: '', seatNumber: '', paymentStatus: 'Paid' });
      loadPageData(); 
    } catch (err) {
      alert(err.response.data.message || 'Booking process failed.');
    }
  };

  return (
    <div className="min-h-screen bg-green-50">
      <CustomerNavbar />
      <div className="p-8 flex flex-col lg:flex-row gap-8 items-start font-sans">
        
        {/* Booking Form Layout */}
        <div className="bg-white p-6 rounded-xl shadow-md w-full lg:w-96 border border-emerald-200">
          <h3 className="text-xl font-bold text-emerald-700 mb-4">Book Bus Ticket</h3>
          <form onSubmit={handleBooking} className="flex flex-col text-sm">
            <label className="font-semibold mb-1">Select Active Schedule Route:</label>
            <select value={form.scheduleId} onChange={(e) => setForm({...form, scheduleId: e.target.value})} className="border border-emerald-300 p-2 rounded mb-4 bg-white" required>
              <option value="">-- Choose Route --</option>
              {schedules.filter(s => s.ScheduleStatus === 'Active').map(s => (
                <option key={s.ScheduleID} value={s.ScheduleID}>{s.RouteName} ({s.TicketPrice})</option>
              ))}
            </select>

            <label className="font-semibold mb-1">Passenger Full Name:</label>
            <input type="text" value={form.passengerName} onChange={(e) => setForm({...form, passengerName: e.target.value})} className="border border-emerald-300 p-2 rounded mb-4" required />

            <label className="font-semibold mb-1">Gender:</label>
            <select value={form.gender} onChange={(e) => setForm({...form, gender: e.target.value})} className="border border-emerald-300 p-2 rounded mb-4 bg-white">
              <option value="Male">Male</option>
              <option value="Female">Female</option>
            </select>

            <label className="font-semibold mb-1">Phone Number:</label>
            <input type="text" value={form.phone} onChange={(e) => setForm({...form, phone: e.target.value})} className="border border-emerald-300 p-2 rounded mb-4" required />

            <label className="font-semibold mb-1">Preferred Seat Number:</label>
            <input type="number" value={form.seatNumber} onChange={(e) => setForm({...form, seatNumber: e.target.value})} className="border border-emerald-300 p-2 rounded mb-6" required />

            <button type="submit" className="bg-emerald-600 text-white py-2 rounded font-bold hover:bg-emerald-700">Confirm Booking</button>
          </form>
        </div>

        {/* Existing Bookings Display Table */}
        <div className="bg-white p-6 rounded-xl shadow-md flex-1 w-full border border-emerald-200">
          <h3 className="text-xl font-bold text-emerald-700 mb-4">Passenger Booking</h3>
          <table className="w-full border-collapse border border-emerald-200 text-left text-xs">
            <thead>
              <tr className="bg-emerald-100 text-emerald-800">
                <th className="border border-emerald-200 p-3">Passenger</th>
                <th className="border border-emerald-200 p-3">Seat</th>
                <th className="border border-emerald-200 p-3">Payment</th>
                <th className="border border-emerald-200 p-3 text-center">Action</th>
              </tr>
            </thead>
            <tbody>
              {bookings.map((b) => (
                <tr key={b.BookingID} className="hover:bg-emerald-50 text-gray-700">
                  <td className="border border-emerald-200 p-3 font-semibold">{b.PassengerName} ({b.PassengerGender})</td>
                  <td className="border border-emerald-200 p-3">No. {b.SeatNumber}</td>
                  <td className="border border-emerald-200 p-3 text-emerald-600 font-bold">{b.PaymentStatus}</td>
                  <td className="border border-emerald-200 p-3 text-center">
                    <button onClick={async () => { await api.delete(`/bookings/${b.BookingID}`); loadPageData(); }} className="bg-red-500 text-white px-3 py-1 rounded hover:bg-red-600">Cancel</button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>

      </div>
    </div>
  );
};

export default CustomerDashboard;

________________________________________________________________________________________________________________________________________

CUSTOMERREPORT.JSX

import React, { useState, useEffect } from 'react';
import api from '../api/config';
import { CustomerNavbar } from '../components/Navbar';

const CustomerReport = () => {
  const [tickets, setTickets] = useState([]);
  useEffect(() => {
    api.get('/report/customer').then(res => setTickets(res.data));
  }, []);

  return (
    <div className="min-h-screen bg-green-50">
      <CustomerNavbar />
      <div className="p-8 font-sans">
        <div className="bg-white p-6 rounded-xl shadow-md border border-emerald-200">
          <h2 className="text-2xl font-bold text-emerald-800 mb-6">My Personal Boarding Manifest</h2>
            </div>
            <button onClick={() => window.print()} className="bg-green-600 text-white font-bold px-4 py-2 rounded-lg hover:bg-green-700 print:hidden shadow">Print Report</button>
          </div>
          <table className="w-full border-collapse border border-emerald-200 text-left">
            <thead>
              <tr className="bg-emerald-100 text-emerald-800">
                <th className="border border-emerald-200 p-3">Passenger Name</th>
                <th className="border border-emerald-200 p-3">Route Destination</th>
                <th className="border border-emerald-200 p-3">Departure Time</th>
                <th className="border border-emerald-200 p-3">Assigned Seat</th>
                <th className="border border-emerald-200 p-3">Payment Status</th>
              </tr>
            </thead>
            <tbody>
              {tickets.map((t) => (
                <tr key={t.BookingID} className="hover:bg-emerald-50 text-gray-700">
                      <td className="border border-emerald-200 p-3 font-semibold">{t.PassengerName}</td>
                  <td className="border border-emerald-200 p-3">{t.RouteName}</td>
                  <td className="border border-emerald-200 p-3 text-sm">{t.DepartureTime}</td>
                  <td className="border border-emerald-200 p-3 font-bold">Seat {t.SeatNumber}</td>
                  <td className="border border-emerald-200 p-3"><span className="bg-emerald-200 text-emerald-800 font-bold px-2 py-1 rounded text-xs">{t.PaymentStatus}</span></td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>
  );
};

export default CustomerReport;
________________________________________________________________________________________________________________________________________

//SCHEDULEMANAGEMENT.JSX

import React, { useState, useEffect } from 'react';
import api from '../api/config';
import { AgentNavbar } from '../components/Navbar';

const ScheduleManagement = () => {
  const [schedules, setSchedules] = useState([]);
  const [buses, setBuses] = useState([]); // Added state to hold our fleet list
  const [form, setForm] = useState({ busId: '', route: '', departure: '', destination: '', depTime: '', arrTime: '', price: '', status: 'Active' });
  const [editingId, setEditingId] = useState(null);

  const fetchData = async () => { 
    const resSched = await api.get('/schedules'); 
    setSchedules(resSched.data);
    const resBuses = await api.get('/buses'); 
    setBuses(resBuses.data); // Fetch buses so we can map them in the form
  };

  useEffect(() => { fetchData(); }, []);

  const handleSubmit = async (e) => {
    e.preventDefault();
    if (editingId) {
      await api.put(`/schedules/${editingId}`, form);
      setEditingId(null);
    } else {
      await api.post('/schedules', form);
    }
    setForm({ busId: '', route: '', departure: '', destination: '', depTime: '', arrTime: '', price: '', status: 'Active' });
    fetchData();
  };

  return (
    <div className="min-h-screen bg-green-50">
      <AgentNavbar />
      <div className="p-6 flex flex-col xl:flex-row gap-6 font-sans">
        
        {/* Form Container */}
        <div className="bg-white p-6 rounded-xl shadow-md w-full xl:w-96 border border-green-200">
          <h3 className="text-xl font-bold text-green-700 mb-4">{editingId ? 'Edit Schedule' : 'Create Schedule'}</h3>
          <form onSubmit={handleSubmit} className="flex flex-col text-sm">
            
            {/* NEW: Dropdown selector mapping our buses dynamically */}
            <label className="font-semibold mb-1">Select Bus Assignment:</label>
            <select 
              value={form.busId} 
              onChange={(e) => setForm({...form, busId: e.target.value})} 
              className="border border-green-300 p-2 rounded mb-4 bg-white outline-none" 
              required
            >
              <option value="">-- Choose Bus (Plate Number) --</option>
              {buses.map((bus) => (
                <option key={bus.BusID} value={bus.BusID}>
                  {bus.PlateNumber} ({bus.BusType})
                </option>
              ))}
            </select>

            <label className="font-semibold mb-0.5">Route Name:</label>
            <input type="text" placeholder="Kigali - Musanze" value={form.route} onChange={(e) => setForm({...form, route: e.target.value})} className="border border-green-300 p-1.5 rounded mb-3 " required />

            <label className="font-semibold mb-0.5">Departure Point:</label>
            <input type="text" value={form.departure} onChange={(e) => setForm({...form, departure: e.target.value})} className="border border-green-300 p-1.5 rounded mb-3 outline-none" required />

            <label className="font-semibold mb-0.5">Destination Terminus:</label>
            <input type="text" value={form.destination} onChange={(e) => setForm({...form, destination: e.target.value})} className="border border-green-300 p-1.5 rounded mb-3 outline-none" required />

            <label className="font-semibold mb-0.5">Departure Time:</label>
            <input type="datetime-local" value={form.depTime} onChange={(e) => setForm({...form, depTime: e.target.value})} className="border border-green-300 p-1.5 rounded mb-3 outline-none" required />

            <label className="font-semibold mb-0.5">Arrival Time:</label>
            <input type="datetime-local" value={form.arrTime} onChange={(e) => setForm({...form, arrTime: e.target.value})} className="border border-green-300 p-1.5 rounded mb-3 outline-none" required />

            <label className="font-semibold mb-0.5">Ticket Price:</label>
            <input type="number" value={form.price} onChange={(e) => setForm({...form, price: e.target.value})} className="border border-green-300 p-1.5 rounded mb-3 outline-none" required />

            <label className="font-semibold mb-0.5">Status:</label>
            <select value={form.status} onChange={(e) => setForm({...form, status: e.target.value})} className="border border-green-300 p-1.5 rounded mb-4 outline-none">
              <option value="Active">Active</option>
              <option value="Inactive">Inactive</option>
            </select>

            <button type="submit" className="bg-green-600 text-white py-2 rounded font-bold hover:bg-green-700">{editingId ? 'Update' : 'Post Schedule'}</button>
          </form>
        </div>

        {/* Display Table */}
        <div className="bg-white p-6 rounded-xl shadow-md flex-1 overflow-x-auto border border-green-200">
          <h3 className="text-xl font-bold text-green-700 mb-4">Master Route Schedules</h3>
          <table className="w-full border-collapse border border-green-200 text-left text-xs">
            <thead>
              <tr className="bg-green-100 text-green-800">
                <th className="border border-green-200 p-2">Route</th>
                <th className="border border-green-200 p-2">From / To</th>
                <th className="border border-green-200 p-2">Departure</th>
                <th className="border border-green-200 p-2">Price</th>
                <th className="border border-green-200 p-2">Status</th>
                <th className="border border-green-200 p-2 text-center">Actions</th>
              </tr>
            </thead>
            <tbody>
              {schedules.map((sc) => (
                <tr key={sc.ScheduleID} className="hover:bg-green-50 text-gray-700">
                  <td className="border border-green-200 p-2 font-bold">{sc.RouteName}</td>
                  <td className="border border-green-200 p-2">{sc.DeparturePoint} → {sc.Destination}</td>
                  <td className="border border-green-200 p-2">{sc.DepartureTime}</td>
                  <td className="border border-green-200 p-2">{sc.TicketPrice}</td>
                  <td className={`border border-green-200 p-2 font-bold ${sc.ScheduleStatus === 'Active' ? 'text-green-600' : 'text-red-500'}`}>{sc.ScheduleStatus}</td>
                  <td className="border border-green-200 p-2 flex gap-1 justify-center">
                    <button onClick={() => { setEditingId(sc.ScheduleID); setForm({ busId: sc.BusID, route: sc.RouteName, departure: sc.DeparturePoint, destination: sc.Destination, depTime: sc.DepartureTime, arrTime: sc.EstimatedArrivalTime, price: sc.TicketPrice, status: sc.ScheduleStatus }); }} className="bg-amber-500 text-white px-2 py-0.5 rounded font-bold hover:bg-amber-600">Edit</button>
                    <button onClick={async () => { await api.delete(`/schedules/${sc.ScheduleID}`); fetchData(); }} className="bg-red-500 text-white px-2 py-0.5 rounded font-bold hover:bg-red-600">Delete</button>
                  </td>
                </tr>
              ))}
            </tbody>
          </table>
        </div>

      </div>
    </div>
  );
};

export default ScheduleManagement;



