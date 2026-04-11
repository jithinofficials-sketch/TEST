# Analytics Dashboard - Testing Guide

## Overview
The Analytics Dashboard tracks customer behavior and sales data to provide insights and recommendations for optimizing your currency conversion strategy.

---

## 1. Dashboard Metrics (Top Cards)

### What Data is Displayed:

#### **Total Visits**
- **What it shows:** Number of customer visits to your store
- **How it's tracked:** Stored in browser local storage when customers visit the store
- **Testing:** Visit the store multiple times and verify the count increases

#### **Currency Clicks** 
- **What it shows:** Number of times customers switched currencies
- **How it's tracked:** Counted each time a customer clicks to change currency using the Bucks switcher
- **Testing:** Switch currencies multiple times and verify the count increases

#### **Total Sales**
- **What it shows:** Total revenue generated across all currencies
- **How it's tracked:** Captured when checkout is completed, using the order amount
- **Testing:** Complete test orders and verify sales total updates

#### **Total Orders**
- **What it shows:** Number of completed orders
- **How it's tracked:** Counted when checkout_completed event fires
- **Testing:** Complete test orders and verify order count increases

#### **Top Currency**
- **What it shows:** The currency that was switched to most frequently
- **How it's tracked:** Identifies which currency has the highest number of switches
- **Testing:** Switch to a specific currency multiple times and verify it shows as top currency

#### **Average Order Value**
- **What it shows:** Average sales per order (Total Sales ÷ Total Orders)
- **How it's calculated:** Automatically computed from total sales and orders
- **Testing:** Complete orders with different amounts and verify the average is correct

---

## 2. Charts Section

### **Currency Conversion Trends** (Bar Chart)
- **What it shows:** Number of currency switches over time
- **How it's tracked:** Records each currency switch with timestamp
- **Data displayed:** Currency code and number of switches
- **Testing:** 
  - Switch currencies on different days
  - Verify the chart shows switches by date
  - Check that currency codes are displayed correctly

### **Revenue by Currency** (Bar Chart)
- **What it shows:** Sales amount for each currency
- **How it's tracked:** 
  - Captures the **last selected currency** when checkout is completed
  - Uses `selectedCurrency` field from the analytics pixel
  - This is the currency the customer was browsing in, NOT the checkout currency
- **Data displayed:** Currency code and total sales amount
- **Testing:**
  - Switch to EUR, then complete checkout
  - Verify sales are attributed to EUR (browsing currency)
  - Complete multiple orders in different browsing currencies
  - Verify each currency shows correct sales total

---

## 3. Currency Performance Tables

### **Currency Switches Table**
- **What it shows:** List of currencies with switch counts
- **Columns:** Currency code, Number of switches
- **How it's tracked:** Counts every time customer switches to that currency
- **Testing:**
  - Switch to different currencies (USD, EUR, GBP, etc.)
  - Verify each currency appears in the table
  - Verify switch counts are accurate

### **Sales by Browsing Currency Table**
- **What it shows:** Sales breakdown by the currency customers were browsing in
- **Columns:** Currency code, Sales amount, Number of orders
- **How it's tracked:** 
  - Records the **last selected currency** before checkout
  - Stored in session storage as `buckscc_customer_currency`
  - Sent to analytics when checkout completes
- **Testing:**
  - Browse in EUR, complete checkout → Sales should show under EUR
  - Browse in GBP, complete checkout → Sales should show under GBP
  - Verify sales amounts match order totals

### **Sales by Country Table**
- **What it shows:** Sales breakdown by customer's country
- **Columns:** Country name, Sales amount, Number of orders
- **How it's tracked:**
  - Uses `billingCountry` from checkout billing address
  - Also tracks `visitorCountry` from session storage (geo-location)
- **Testing:**
  - Complete orders with different billing countries
  - Verify each country appears in the table
  - Verify sales amounts are correct per country

---

## 4. Smart Recommendations

### **Add Currency to Selected Currencies**
- **What it suggests:** Currencies that should be added to your selected currency list
- **When it appears:** 
  - A currency has switches but is NOT in your selected currencies list
  - This happens when customers are auto-switched based on their location
  - Minimum threshold: 1 currency switch
- **Example:** 
  - EUR has 10 switches but is not in selected currencies
  - Suggestion: "Add EUR to Selected Currencies"
- **Testing:**
  1. Remove EUR from selected currencies in settings
  2. Enable auto-location currency switching
  3. Visit from a European location (or use VPN)
  4. Customer will be auto-switched to EUR
  5. Verify suggestion appears to add EUR

### **Price Increase Suggestion**
- **What it suggests:** Increase prices in your top-performing country
- **When it appears:**
  - A country has the highest sales amount
  - Sales amount is greater than threshold (minimum: $1)
  - Country exists in a Shopify market
  - Overall average order value is >= $1
- **Conditions:**
  - If country is in a **single-country market**: Suggests price increase
  - If country is in a **multi-country market**: Suggests creating separate market first
  - If only 1 country total across all markets: Suggestion is hidden
- **Example:**
  - United States has $5,000 in sales (highest)
  - US is in its own market
  - Suggestion: "Increase Prices in United States Market"
- **Testing:**
  1. Complete multiple orders from one country (e.g., US)
  2. Ensure that country has a dedicated Shopify market
  3. Verify sales exceed $1
  4. Check suggestion appears for price increase

### **Market Optimization Suggestion**
- **What it suggests:** Add high-performing countries to Shopify Markets
- **When it appears:**
  - A country has sales but is NOT in any Shopify market
  - Country sales amount is greater than threshold (minimum: $1)
  - Country-specific average order value is >= $1
  - Overall average order value is >= $1
- **Shows:** Country name, top currency used, sales amount, order count
- **Example:**
  - Germany has $1,200 in sales using EUR
  - Germany is not in any market
  - Suggestion: "Add Germany (EUR) to Markets"
- **Testing:**
  1. Complete orders from a country (e.g., Germany)
  2. Ensure Germany is NOT in any Shopify market
  3. Verify sales exceed $1
  4. Check suggestion appears to add Germany to markets

---

## 5. Data Tracking Flow

### **How Data is Captured:**

1. **Customer Visits Store**
   - Visit count stored in local storage
   - Visitor country detected and stored in session storage

2. **Customer Switches Currency**
   - Currency switch event recorded
   - Selected currency stored in session storage (`buckscc_customer_currency`)
   - Switch count incremented

3. **Customer Completes Checkout**
   - Analytics pixel fires on `checkout_completed` event
   - Data sent to analytics API:
     - Order amount
     - Checkout currency (what Shopify charged)
     - **Selected currency** (what customer was browsing in - from session storage)
     - Billing country (from checkout address)
     - Visitor country (from session storage)
     - Shop domain
     - Timestamp

4. **Data Stored in Database**
   - Sales by country and currency
   - Currency conversion trends
   - Order counts and totals

5. **Dashboard Displays Data**
   - Aggregates data from database
   - Calculates metrics and ratios
   - Generates suggestions based on thresholds

---

## 6. Testing Checklist

### **Basic Functionality:**
- [ ] Visit store → Verify Total Visits increases
- [ ] Switch currency → Verify Currency Clicks increases
- [ ] Complete order → Verify Total Sales and Orders update
- [ ] Check Top Currency shows most-switched currency

### **Charts:**
- [ ] Switch currencies on different days → Verify trend chart updates
- [ ] Complete orders in different browsing currencies → Verify revenue chart shows correct currencies

### **Tables:**
- [ ] Switch to multiple currencies → Verify all appear in Currency Switches table
- [ ] Complete orders while browsing in different currencies → Verify Sales by Browsing Currency table
- [ ] Complete orders from different countries → Verify Sales by Country table

### **Recommendations:**
- [ ] Auto-switch to currency not in selected list → Verify "Add Currency" suggestion
- [ ] Generate high sales in one country with market → Verify "Price Increase" suggestion
- [ ] Generate sales in country without market → Verify "Market Optimization" suggestion

### **Edge Cases:**
- [ ] Zero sales → Verify no suggestions appear
- [ ] Only one country → Verify price increase suggestion is hidden
- [ ] All countries in markets → Verify no market optimization suggestions

---

## 7. Important Notes

### **Browsing Currency vs Checkout Currency:**
- **Browsing Currency:** The currency the customer selected/was viewing prices in (tracked by Bucks)
- **Checkout Currency:** The currency Shopify actually charged (may be different)
- **Analytics uses BROWSING CURRENCY** for revenue attribution to understand customer preferences

### **Thresholds:**
All thresholds are currently set to minimum values (1) for testing:
- Minimum visits: 1
- Minimum conversions: 1  
- Minimum currency switches: 1
- Minimum sales ratio (avg order value): $1

### **Data Refresh:**
- Dashboard data updates based on date range selected (Today, Last 7 Days, Last 30 Days)
- Suggestions are generated each time the page loads
- Real-time updates require page refresh

---

## 8. Common Testing Scenarios

### **Scenario 1: New Store Setup**
1. No data initially → All metrics show 0
2. Visit store → Total Visits = 1
3. Switch currency → Currency Clicks = 1
4. Complete order → Sales and Orders update
5. Verify all tables populate with data

### **Scenario 2: Multi-Currency Sales**
1. Browse in EUR, complete order for $100
2. Browse in GBP, complete order for $150  
3. Browse in USD, complete order for $200
4. Verify Revenue by Currency chart shows:
   - EUR: $100
   - GBP: $150
   - USD: $200

### **Scenario 3: Market Recommendations**
1. Complete $500 in sales from Germany (not in market)
2. Verify suggestion: "Add Germany (EUR) to Markets"
3. Add Germany to a market in Shopify
4. Refresh dashboard
5. Verify suggestion changes to price increase or disappears

### **Scenario 4: Auto-Location Currency**
1. Remove JPY from selected currencies
2. Enable auto-location switching
3. Visit from Japan (or use VPN)
4. Customer auto-switched to JPY
5. Verify suggestion: "Add JPY to Selected Currencies"

---

## Support

For technical issues or questions about the analytics implementation, contact the development team.
