# coupon_distributor
const express = require('express');
const bodyParser = require('body-parser');
const cookieParser = require('cookie-parser');
const app = express();
const PORT = process.env.PORT || 3000;

// Middleware
app.use(bodyParser.urlencoded({ extended: true }));
app.use(cookieParser());

// Coupon List
const coupons = ['COUPON1', 'COUPON2', 'COUPON3', 'COUPON4'];
let currentIndex = 0;

// IP Tracking (in-memory store)
const ipTracker = {};

// Helper Functions
function getIPAddress(req) {
    return req.headers['x-forwarded-for'] || req.connection.remoteAddress;
}

function canClaimCoupon(ip, cookies) {
    const currentTime = Date.now();
    const oneHourInMillis = 60 * 60 * 1000;

    // Check IP restriction
    if (ipTracker[ip] && (currentTime - ipTracker[ip]) < oneHourInMillis) {
        return false;
    }

    // Check cookie restriction
    if (cookies.coupon_claim_id) {
        return false;
    }

    return true;
}

// Routes
app.get('/', (req, res) => {
    res.send(`
        <h1>Coupon Distribution</h1>
        <form action="/claim" method="POST">
            <button type="submit">Claim Coupon</button>
        </form>
    `);
});

app.post('/claim', (req, res) => {
    const ip = getIPAddress(req);
    const cookies = req.cookies;

    if (!canClaimCoupon(ip, cookies)) {
        const remainingTime = Math.ceil((ipTracker[ip] + (60 * 60 * 1000) - Date.now()) / 1000 / 60);
        return res.send(`<p>You can claim another coupon in ${remainingTime} minutes.</p>`);
    }

    // Assign coupon
    const coupon = coupons[currentIndex];
    currentIndex = (currentIndex + 1) % coupons.length;

    // Update IP tracker
    ipTracker[ip] = Date.now();

    // Set cookie
    res.cookie('coupon_claim_id', 'claimed', { maxAge: 60 * 60 * 1000, httpOnly: true });

    // Respond to user
    res.send(`<p>Coupon claimed successfully: ${coupon}</p>`);
});

// Start Server
app.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
});
