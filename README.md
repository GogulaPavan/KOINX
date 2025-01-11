# KOINX

Step 1: Setting up the Environment
Install dependencies:
bash
Copy code
npm init -y
npm install express mongoose node-cron axios dotenv
npm install --save-dev nodemon
Add a .env file for storing configuration:
plaintext
Copy code
MONGO_URI=<Your MongoDB URI>
PORT=3000
COINGECKO_API_BASE=https://api.coingecko.com/api/v3
Step 2: Database Schema
File: src/models/CryptoPrice.js

javascript
Copy code
const mongoose = require("mongoose");

const cryptoPriceSchema = new mongoose.Schema({
  coin: { type: String, required: true },
  price: { type: Number, required: true },
  marketCap: { type: Number, required: true },
  change24h: { type: Number, required: true },
  timestamp: { type: Date, default: Date.now },
});

module.exports = mongoose.model("CryptoPrice", cryptoPriceSchema);
Step 3: Background Job
File: src/jobs/fetchCryptoPrices.js

javascript
Copy code
const axios = require("axios");
const CryptoPrice = require("../models/CryptoPrice");

const coins = ["bitcoin", "matic-network", "ethereum"];

const fetchCryptoPrices = async () => {
  try {
    const results = await Promise.all(
      coins.map(async (coin) => {
        const { data } = await axios.get(
          `${process.env.COINGECKO_API_BASE}/simple/price`,
          {
            params: {
              ids: coin,
              vs_currencies: "usd",
              include_market_cap: true,
              include_24hr_change: true,
            },
          }
        );

        return {
          coin,
          price: data[coin].usd,
          marketCap: data[coin].usd_market_cap,
          change24h: data[coin].usd_24h_change,
        };
      })
    );

    // Save results to the database
    await CryptoPrice.insertMany(results.map((item) => ({ ...item })));
    console.log("Crypto prices fetched and stored.");
  } catch (err) {
    console.error("Error fetching crypto prices:", err.message);
  }
};

module.exports = fetchCryptoPrices;
Step 4: Routes
1. /stats File: src/routes/stats.js

javascript
Copy code
const express = require("express");
const CryptoPrice = require("../models/CryptoPrice");

const router = express.Router();

router.get("/stats", async (req, res) => {
  const { coin } = req.query;

  if (!coin) {
    return res.status(400).json({ error: "Coin is required" });
  }

  try {
    const latestRecord = await CryptoPrice.findOne({ coin })
      .sort({ timestamp: -1 })
      .lean();

    if (!latestRecord) {
      return res.status(404).json({ error: "No data found for this coin" });
    }

    res.json({
      price: latestRecord.price,
      marketCap: latestRecord.marketCap,
      "24hChange": latestRecord.change24h,
    });
  } catch (err) {
    res.status(500).json({ error: "Server error" });
  }
});

module.exports = router;
2. /deviation File: src/routes/deviation.js

javascript
Copy code
const express = require("express");
const CryptoPrice = require("../models/CryptoPrice");

const router = express.Router();

router.get("/deviation", async (req, res) => {
  const { coin } = req.query;

  if (!coin) {
    return res.status(400).json({ error: "Coin is required" });
  }

  try {
    const records = await CryptoPrice.find({ coin })
      .sort({ timestamp: -1 })
      .limit(100)
      .lean();

    if (!records.length) {
      return res.status(404).json({ error: "No data found for this coin" });
    }

    const prices = records.map((record) => record.price);
    const mean =
      prices.reduce((sum, price) => sum + price, 0) / prices.length;
    const variance =
      prices.reduce((sum, price) => sum + Math.pow(price - mean, 2), 0) /
      prices.length;
    const stdDev = Math.sqrt(variance);

    res.json({ deviation: stdDev.toFixed(2) });
  } catch (err) {
    res.status(500).json({ error: "Server error" });
  }
});

module.exports = router;
Step 5: App Initialization
File: src/app.js

javascript
Copy code
const express = require("express");
const mongoose = require("mongoose");
const statsRoute = require("./routes/stats");
const deviationRoute = require("./routes/deviation");

require("dotenv").config();

const app = express();

mongoose
  .connect(process.env.MONGO_URI, { useNewUrlParser: true, useUnifiedTopology: true })
  .then(() => console.log("MongoDB connected"))
  .catch((err) => console.error("MongoDB connection error:", err));

app.use(express.json());
app.use("/api", statsRoute);
app.use("/api", deviationRoute);

module.exports = app;
Step 6: Start Server
File: src/server.js

javascript
Copy code
const app = require("./app");
const cron = require("node-cron");
const fetchCryptoPrices = require("./jobs/fetchCryptoPrices");

const PORT = process.env.PORT || 3000;

app.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});

// Schedule background job to run every 2 hours
cron.schedule("0 */2 * * *", fetchCryptoPrices);
Step 7: Testing
Use Postman or curl to test /stats and /deviation APIs.
Optional Steps
Deploy MongoDB to MongoDB Atlas.
Deploy the backend using Heroku, AWS, or Azure.
Push code to GitHub with a proper README.md.
