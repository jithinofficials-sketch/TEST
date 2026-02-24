# Country-Based Switching Feature RFC

**Metadata:**
- **Author:** Jithin C
- **Status:** Draft
- **Type:** #feature-rfc
- **Reviewers:** Tech Lead

---

## **Problem**

Currently, the app only supports **currency switching**, but merchants operating in multiple countries via Shopify Markets need **country-based switching** to provide localized experiences (language, pricing, shipping, taxes, etc.).

**Pain Points:**
- Merchants with Shopify Markets enabled cannot offer country selection to customers
- Currency switching alone doesn't trigger full market localization (taxes, shipping rules, language)
- No way to configure which countries to display in the switcher
- Customers in non-market countries have no fallback mechanism

**Current Workarounds:**
- Merchants manually add country selectors using custom code
- Currency switching is used as a proxy, but doesn't trigger full localization

---

## **Why This Matters?**

**Why now:**
- Shopify Markets adoption is growing rapidly
- Merchants need seamless country switching to improve international conversion rates
- Competitors offer this feature; we're losing potential customers

**Benefits:**
- **Revenue:** Premium feature opportunity (tiered pricing)
- **Merchant Value:** Better international customer experience → higher conversions
- **Product Quality:** Complete localization solution (not just currency)
- **Competitive Edge:** Match/exceed competitor offerings

---

## **Proposed Solution**

Add **country-based switching** alongside existing currency switching, with full admin configuration and widget support.

### **Admin Side (Configuration)**

**Layout Options:**

We have **two approaches** for the admin configuration UI:

#### **Option A: Separate "Country Switching" Tab** ✅ Recommended

Create a new dedicated tab in the settings navigation alongside existing tabs.

**Pros:**
- Clean separation of concerns (currency vs country)
- Easier to add country-specific features in the future
- Simpler premium gating (entire tab can be premium)
- Better UX for merchants with complex multi-market setups
- Follows existing pattern (separate tabs for different features)

**Cons:**
- Adds one more tab to navigation
- Slight increase in development time (new tab component)

**Implementation:**
- Add "Country Switching" to tab navigation in `pages/settings/index.jsx`
- Create new component: `components/country-switching-tab/CountrySwitchingTab.jsx`
- Similar structure to existing `SelectCurrencies.jsx`

---

#### **Option B: Integrate into Current Settings Tab**

Add country switching configuration within the existing "Currency Selector" tab (rename to "Currency & Country Switching").

**Pros:**
- All localization settings in one place
- No additional navigation complexity
- Faster initial implementation

**Cons:**
- Tab becomes crowded with too many options
- Harder to organize premium vs free features
- Mixing two distinct concepts (currency vs country)
- May confuse merchants who only want currency OR country switching

**Implementation:**
- Rename existing tab to "Localization" or "Currency & Country"
- Add country switching section below currency selector
- Use collapsible cards to separate currency vs country configs

---

**Recommended Approach: Option A (Separate Tab)**

Reasoning:
- Cleaner UX and better scalability
- Easier premium feature management
- Follows app's existing architecture pattern
- Better for future country-specific features (language switching, regional pricing rules, etc.)

---

### **Configuration Options** (Regardless of Layout)

1. **Enable/Disable Country Switching**
   - Toggle: ON/OFF
   - Free feature for all users

2. **Select Countries** (Premium Feature)
   - Multi-select dropdown showing only Shopify Markets-enabled countries
   - Auto-fetch from Shopify Markets API
   - Validation: Only allow countries configured in Shopify Markets
   - **Free tier:** Max 3 countries
   - **Premium tier:** Unlimited countries

3. **Display Options** (Free)
   - Radio buttons:
     - Show both switchers (currency + country)
     - Show only country switcher
     - Show only currency switcher
   - Default: "Show both switchers"

4. **Theme & Styling** (Reuse existing)
   - Use same theme configurations as currency switcher
   - Same color pickers (background, text, hover)
   - Same flag styles and display options
   - Same positioning options (floating, header, custom)
   - **Note:** Both switchers share styling to maintain consistency

5. **Premium Features**
   - Select more than 3 countries (Free: max 3, Premium: unlimited)
   - Advanced fallback logic (auto-detect country → currency fallback)
   - Custom country labels/names (e.g., "Deutschland" instead of "Germany")
   - Analytics dashboard for country selection tracking

---

### **Widget Side (Implementation)**

**Changes Required:**

1. **New Template Files**
   - `selectCountry.js` - Generate country dropdown HTML
   - `selectCountryEvents.js` - Handle country dropdown UI/events

2. **Modified Files**
   - `addWrapperDiv.js` - Create country wrapper container
   - `addTemplate.js` - Inject country template alongside currency
   - `changeMultiCurrency.js` - Add country switching logic with fallback
   - `setConfig.js` - Parse `selectedCountries` from config

3. **Conversion Logic**
   ```javascript
   // When country is selected:
   if (country in ShopifyMarkets) {
     // Full localization
     POST /localization {
       country_code: 'DE',
       form_type: 'localization'
     }
   } else {
     // Fallback to currency
     POST /localization {
       currency: 'EUR',
       form_type: 'currency'
     }
   }
   ```

4. **Display Logic**
   - Both switchers stacked vertically with flex layout
   - Same CSS classes for consistent styling
   - Unique selectors to avoid conflicts (`buckcc` vs `buckcc-country`)
   - Responsive: Stack on mobile, side-by-side on desktop (optional)

**Edge Cases:**
- Country not in Shopify Markets → fallback to currency switching
- No countries selected → hide country switcher
- Both switchers disabled → show nothing
- Mobile responsive layout for both switchers
- Country disabled in Markets after selection → skip in dropdown

---

## **Technical Details**

### **Database Schema Changes**

**Settings Table:**
```javascript
{
  countrySwitchingEnabled: Boolean,
  selectedCountries: JSON, // [{"code": "US", "name": "United States"}, {"code": "GB", "name": "United Kingdom"}]
  displayMode: String, // "both" | "country_only" | "currency_only"
  
  // Reuse existing fields:
  themeType: String,
  backgroundColor: String,
  textColor: String,
  hoverColor: String,
  // ... etc
}
```

### **API Changes**

**New Endpoint:** `GET /api/v1/markets/countries`
- Fetch available countries from Shopify Markets API
- Return only countries enabled in merchant's Markets settings
- Response format:
  ```javascript
  {
    "countries": [
      {"code": "US", "name": "United States", "currency": "USD"},
      {"code": "GB", "name": "United Kingdom", "currency": "GBP"}
    ]
  }
  ```

### **Frontend (Admin) Changes**

**New Component:** `CountrySwitchingTab.jsx` (if Option A)
- Similar structure to `SelectCurrencies.jsx`
- Multi-select for countries (filtered by Markets)
- Display mode radio buttons
- Enable/disable toggle
- Premium upgrade prompt if > 3 countries selected

**Modified Component:** `index.jsx` (Settings page)
- Add new tab to tab navigation (if Option A)
- OR add new section to existing tab (if Option B)
- Handle `selectedCountries` serialization (JSON.stringify)

### **Widget Changes**

**Flow:**
```
1. setConfig.js → Parse selectedCountries from JSON
2. addWrapperDiv.js → Create .buckscc-country-wrapper
3. addTemplate.js → Inject country template
4. selectCountry.js → Generate <select> with countries
5. selectCountryEvents.js → Transform to custom dropdown
6. User clicks country → changeMultiCurrency() with country flag
7. POST /localization with country_code or currency fallback
8. Page reloads with new market/currency
```

**Code Structure:**
```javascript
// selectCountry.js
const selectCountry = config => {
  const countries = config.selectedCountries || [];
  let html = '<select class="buckcc buckcc-country">';
  countries.forEach(country => {
    html += `<option value="${country.code}">${country.name}</option>`;
  });
  html += '</select>';
  return html;
};

// selectCountryEvents.js (similar to selectEvents.js)
const events = config => {
  hxo$('select.buckcc-country').each(function() {
    // Transform to custom dropdown
    // Attach click handlers
    // Trigger changeMultiCurrency with country flag
  });
};
```

---

## **General Questions/FAQ**

**Q: Why separate tab instead of adding to Currency Selector tab?**  
A: Keeps UI clean, allows for future country-specific features, easier to make premium. However, both options are viable - see "Admin Side (Configuration)" section above.

**Q: How do we prevent showing countries not in Shopify Markets?**  
A: Fetch available countries from Shopify Markets API, filter dropdown to only show those.

**Q: What if merchant disables a country in Markets after selecting it?**  
A: Widget validates against `availableCountries` array, skips disabled countries.

**Q: How does fallback work for non-market countries?**  
A: Widget maintains country→currency mapping, falls back to currency switching if country not in Markets.

**Q: Can merchants customize country names?**  
A: Premium feature - allow custom labels (e.g., "Deutschland" instead of "Germany").

**Q: Will this work without Shopify Markets?**  
A: Yes, but with limited functionality. Country switching will fallback to currency switching only.

**Q: How do we handle styling conflicts between currency and country switchers?**  
A: Use unique CSS classes (`buckcc` vs `buckcc-country`) and shared theme variables.

---

## **Open Questions**

- Should we allow country switching without Shopify Markets enabled? (Fallback to currency only?)
- What's the ideal premium tier pricing for unlimited countries?
- Should we add analytics to track which countries are selected most?
- Do we need separate positioning for country vs currency switcher?
- Should country switcher trigger language switching too (if available in Markets)?
- Should we auto-detect and pre-select countries based on merchant's Markets configuration?

---

## **Implementation Plan**

| **Phase** | **Goal** | **Timeline** | **Status** |
|-----------|----------|--------------|------------|
| **Phase 0: Decision** | Finalize admin layout approach (Option A vs B) | 0.5 days | Pending |
| - Review with team |  |  |  |
| - Decide on separate tab vs integrated |  |  |  |
| **Phase 1: Admin Setup** | Create Country Switching configuration UI | 2-3 days | Pending |
| - Add new tab component (if Option A) OR section (if Option B) |  |  |  |
| - Add selectedCountries field to DB schema |  |  |  |
| - Create Markets API endpoint |  |  |  |
| - Implement country multi-select with Markets filtering |  |  |  |
| - Add display mode radio buttons |  |  |  |
| - Save/load configuration |  |  |  |
| **Phase 2: Widget Template** | Create country switcher template & events | 2 days | Pending |
| - Create selectCountry.js |  |  |  |
| - Create selectCountryEvents.js |  |  |  |
| - Add unique CSS classes (buckcc-country) |  |  |  |
| - Implement custom dropdown UI |  |  |  |
| **Phase 3: Integration** | Connect admin config to widget | 1 day | Pending |
| - Modify addWrapperDiv.js to create country wrapper |  |  |  |
| - Modify addTemplate.js to inject country template |  |  |  |
| - Add flex layout for both switchers |  |  |  |
| - Parse selectedCountries in setConfig.js |  |  |  |
| **Phase 4: Conversion Logic** | Implement country switching with fallback | 2 days | Pending |
| - Extend changeMultiCurrency.js with country support |  |  |  |
| - Add country→currency mapping |  |  |  |
| - Implement Markets validation |  |  |  |
| - Test Markets vs non-Markets countries |  |  |  |
| - Add fallback to currency switching |  |  |  |
| **Phase 5: Testing & Polish** | QA, edge cases, styling fixes | 2 days | Pending |
| - Test both switchers together |  |  |  |
| - Test display modes (both/country only/currency only) |  |  |  |
| - Mobile responsive testing |  |  |  |
| - Cross-browser testing |  |  |  |
| - Edge case testing (disabled countries, no Markets, etc.) |  |  |  |
| **Phase 6: Premium Features** | Add premium tier logic | 1 day | Pending |
| - Limit free tier to 3 countries |  |  |  |
| - Add upgrade prompts in admin |  |  |  |
| - Add custom country labels (premium) |  |  |  |
| - Add analytics tracking (premium) |  |  |  |

**Total Estimated Time:** 10.5-11.5 days

**Release Strategy:** Incremental rollout
- Phase 0-1: Internal review and admin UI development
- Phase 2-4: Beta release to select merchants (with Markets enabled)
- Phase 5: Full release (free tier - max 3 countries)
- Phase 6: Premium tier launch with unlimited countries

---

## **Success Metrics**

- **Adoption Rate:** % of merchants with Markets who enable country switching
- **Conversion Impact:** Change in conversion rate for international customers
- **Premium Upgrades:** % of free users who upgrade for unlimited countries
- **Support Tickets:** Reduction in localization-related support requests
- **Usage Analytics:** Most selected countries, switching frequency

---

## **Risks & Mitigation**

| **Risk** | **Impact** | **Mitigation** |
|----------|-----------|----------------|
| Shopify Markets API changes | High | Version API calls, add error handling, monitor Shopify changelog |
| Performance impact (two switchers) | Medium | Lazy load, optimize CSS, use shared event handlers |
| Merchant confusion (two switchers) | Medium | Clear documentation, onboarding tooltips, display mode options |
| Premium tier adoption low | Medium | A/B test pricing, offer trial period, highlight value proposition |
| Browser compatibility issues | Low | Cross-browser testing, polyfills for older browsers |

---

## **Appendix**

### **Comparison: Option A vs Option B**

| **Criteria** | **Option A (Separate Tab)** | **Option B (Integrated)** |
|--------------|----------------------------|---------------------------|
| **Development Time** | +0.5 days | Baseline |
| **UX Clarity** | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐ Good |
| **Scalability** | ⭐⭐⭐⭐⭐ Excellent | ⭐⭐⭐ Good |
| **Premium Gating** | ⭐⭐⭐⭐⭐ Easy | ⭐⭐⭐ Moderate |
| **Navigation Complexity** | ⭐⭐⭐ Adds one tab | ⭐⭐⭐⭐⭐ No change |
| **Future Features** | ⭐⭐⭐⭐⭐ Easy to add | ⭐⭐⭐ Harder to add |
| **Maintenance** | ⭐⭐⭐⭐⭐ Isolated | ⭐⭐⭐ Coupled |

**Recommendation:** Option A (Separate Tab) for better long-term maintainability and UX.

---

**Next Steps:**
1. Review RFC with tech lead
2. Decide on admin layout approach (Option A vs B)
3. Get approval for premium tier pricing model
4. Begin Phase 0 implementation

