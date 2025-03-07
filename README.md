// server.js

// 1. Import Dependencies
const express = require("express");
const mongoose = require("mongoose");
const axios = require("axios");
const cors = require("cors");

// 2. Create Express App and apply middleware
const app = express();
app.use(express.json());
app.use(cors());

// 3. Connect to MongoDB
mongoose
  .connect("mongodb://localhost:27017/transactions", {
    useNewUrlParser: true,
    useUnifiedTopology: true,
  })
  .then(() => console.log("Connected to MongoDB"))
  .catch((err) => console.error("MongoDB connection error:", err));

// 4. Define the Transaction Schema and Model
const transactionSchema = new mongoose.Schema({
  id: Number,
  title: String,
  description: String,
  price: Number,
  category: String,
  sold: Boolean,
  dateOfSale: Date,
});

const Transaction = mongoose.model("Transaction", transactionSchema);

// 5. Helper Function: Get Date Range for a Month (ignoring year)
function getMonthDateRange(month) {
  const monthNames = [
    "January",
    "February",
    "March",
    "April",
    "May",
    "June",
    "July",
    "August",
    "September",
    "October",
    "November",
    "December",
  ];
  // Get index (0-based) for the provided month
  const monthIndex = monthNames.findIndex(
    (m) => m.toLowerCase() === month.toLowerCase()
  );
  if (monthIndex === -1) {
    throw new Error("Invalid month provided");
  }
  // Use a fixed year (2022) to create the date range
  const startDate = new Date(2022, monthIndex, 1);
  const endDate = new Date(2022, monthIndex + 1, 1);
  return { startDate, endDate };
}

// 6. API: Initialize Database with Seed Data
app.get("/api/initialize", async (req, res) => {
  try {
    // Fetch data from the third-party API
    const { data } = await axios.get(
      "https://s3.amazonaws.com/roxiler.com/product_transaction.json"
    );

    // Optionally clear existing data before seeding
    await Transaction.deleteMany({});
    // Insert fetched data into MongoDB
    await Transaction.insertMany(data);

    res.json({
      message: "Database initialized successfully!",
      count: data.length,
    });
  } catch (error) {
    res.status(500).json({
      error: "Error initializing database",
      details: error.message,
    });
  }
});

// 7. API: List All Transactions with Search & Pagination
app.get("/api/transactions", async (req, res) => {
  try {
    // Extract query parameters (month is required)
    const { month, search = "", page = 1, perPage = 10 } = req.query;
    if (!month)
      return res.status(400).json({ error: "Month query parameter is required" });

    // Get date range for the provided month (ignoring year)
    const { startDate, endDate } = getMonthDateRange(month);

    // Build filter: match transactions within the month and (if search is provided) match title, description, or price.
    const filter = {
      dateOfSale: { $gte: startDate, $lt: endDate },
      $or: [
        { title: { $regex: search, $options: "i" } },
        { description: { $regex: search, $options: "i" } },
        { price: !isNaN(search) ? Number(search) : { $exists: true } },
      ],
    };

    // If the search parameter is empty, remove the $or condition so we get all records for the month.
    if (search.trim() === "") {
      delete filter.$or;
    }

    // Find matching transactions and apply pagination.
    const transactions = await Transaction.find(filter)
      .skip((page - 1) * perPage)
      .limit(Number(perPage));

    // Count total matching transactions
    const total = await Transaction.countDocuments(filter);

    res.json({ transactions, total });
  } catch (error) {
    res.status(500).json({
      error: "Error fetching transactions",
      details: error.message,
    });
  }
});

// 8. API: Statistics for a Selected Month
app.get("/api/statistics", async (req, res) => {
  try {
    const { month } = req.query;
    if (!month)
      return res.status(400).json({ error: "Month query parameter is required" });
    const { startDate, endDate } = getMonthDateRange(month);

    // Get all transactions for the selected month
    const transactions = await Transaction.find({
      dateOfSale: { $gte: startDate, $lt: endDate },
    });

    // Calculate statistics:
    // - totalSaleAmount: sum of prices for sold items
    // - totalSoldItems: count of sold items
    // - totalNotSoldItems: count of unsold items
    const totalSaleAmount = transactions.reduce(
      (sum, t) => sum + (t.sold ? t.price : 0),
      0
    );
    const totalSoldItems = transactions.filter((t) => t.sold).length;
    const totalNotSoldItems = transactions.filter((t) => !t.sold).length;

    res.json({ totalSaleAmount, totalSoldItems, totalNotSoldItems });
  } catch (error) {
    res.status(500).json({
      error: "Error fetching statistics",
      details: error.message,
    });
  }
});

// 9. API: Bar Chart Data (Price Ranges) for a Selected Month
app.get("/api/bar-chart", async (req, res) => {
  try {
    const { month } = req.query;
    if (!month)
      return res.status(400).json({ error: "Month query parameter is required" });
    const { startDate, endDate } = getMonthDateRange(month);

    // Get transactions for the month
    const transactions = await Transaction.find({
      dateOfSale: { $gte: startDate, $lt: endDate },
    });

    // Define price ranges and initialize counts
    const priceRanges = {
      "0-100": 0,
      "101-200": 0,
      "201-300": 0,
      "301-400": 0,
      "401-500": 0,
      "501-600": 0,
      "601-700": 0,
      "701-800": 0,
      "801-900": 0,
      "901-above": 0,
    };

    // Count items falling into each price range
    transactions.forEach((t) => {
      const price = t.price;
      if (price >= 0 && price <= 100) priceRanges["0-100"]++;
      else if (price >= 101 && price <= 200) priceRanges["101-200"]++;
      else if (price >= 201 && price <= 300) priceRanges["201-300"]++;
      else if (price >= 301 && price <= 400) priceRanges["301-400"]++;
      else if (price >= 401 && price <= 500) priceRanges["401-500"]++;
      else if (price >= 501 && price <= 600) priceRanges["501-600"]++;
      else if (price >= 601 && price <= 700) priceRanges["601-700"]++;
      else if (price >= 701 && price <= 800) priceRanges["701-800"]++;
      else if (price >= 801 && price <= 900) priceRanges["801-900"]++;
      else if (price > 900) priceRanges["901-above"]++;
    });

    res.json(priceRanges);
  } catch (error) {
    res.status(500).json({
      error: "Error fetching bar chart data",
      details: error.message,
    });
  }
});

// 10. API: Pie Chart Data (Category Distribution) for a Selected Month
app.get("/api/pie-chart", async (req, res) => {
  try {
    const { month } = req.query;
    if (!month)
      return res.status(400).json({ error: "Month query parameter is required" });
    const { startDate, endDate } = getMonthDateRange(month);

    // Get transactions for the month
    const transactions = await Transaction.find({
      dateOfSale: { $gte: startDate, $lt: endDate },
    });

    // Count the number of items in each category
    const categoryCount = {};
    transactions.forEach((t) => {
      categoryCount[t.category] = (categoryCount[t.category] || 0) + 1;
    });

    res.json(categoryCount);
  } catch (error) {
    res.status(500).json({
      error: "Error fetching pie chart data",
      details: error.message,
    });
  }
});

// 11. API: Combined Data from Statistics
