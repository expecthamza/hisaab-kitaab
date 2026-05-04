HisabKitab — Smart Personal Finance Dashboard
"Hisab Kitab" (حساب کتاب) — Urdu for "Accounts & Ledger"
 
A comprehensive, offline-first personal finance management application built as a single self-contained HTML file. Designed for Pakistani users, HisabKitab combines expense tracking, income management, investment analysis, Islamic finance tools, health tracking, and AI-powered advisory — all without requiring any installation, backend server, or internet connection for core functionality.
 
Author: {Hamza Ijaz Qazi}
 
Table of Contents
1. Overview
2. Architecture
3. Core Modules
4. Feature Reference
5. Security System
6. Supabase Cloud Sync
7. UI & Theming Engine
8. Data Management
9. Function Reference
10. Deployment
11. Technology Stack
 

 
Overview
HisabKitab is a zero-dependency, single-file Progressive Web Application that runs entirely in the browser. All data is persisted locally via localStorage and IndexedDB, with optional encrypted cloud backup through Supabase. The application is structured as a multi-page dashboard where each "page" is a hidden <div> revealed on navigation — eliminating page reloads and delivering a native app-like experience.
 
Design Philosophy
 
Principle	Implementation
Offline-first	All core features work without internet
Privacy-first	Data never leaves the device unless explicitly synced
Zero installation	Single .html file — open in any modern browser
Pakistan-centric	PKR currency, Hijri calendar, prayer times, Zakat, KIBOR rates
Accessible	High-contrast mode, scalable fonts, screen-reader ARIA labels
 

 
Architecture
         
    HisabKitab (index.html)    
    │    
    ├── Early Theme Inject (inline <script> before CSS paint)    
    ├── CSS Layer    
    │   ├── CSS Custom Properties (12 colour themes)    
    │   ├── Layout System (sidebar + main, responsive grid)    
    │   ├── Component Styles (cards, modals, forms, tables)    
    │   └── Animation & Wallpaper Engine    
    │    
    ├── HTML Layer    
    │   ├── Sidebar Navigation (26 pages)    
    │   ├── FAB Button Group (fab5 — 9 floating action buttons)    
    │   ├── Modal Overlays (expenses, EMIs, income, goals …)    
    │   └── Page Sections (hidden divs, shown via showPage())    
    │    
    └── JavaScript Layer    
        ├── State Engine      — loadState / saveState / IDB    
        ├── Security Layer    — PIN, encryption, biometrics    
        ├── Render Engine     — per-page render functions    
        ├── Chart Engine      — Canvas-based donut, trend, bar    
        ├── Sync Layer        — Supabase REST via fetch()    
        ├── AI Advisor        — Anthropic Claude API integration    
        ├── Voice Control     — Web Speech API    
        ├── Theme Engine      — 12 themes + 9 animated wallpapers    
        └── Utility Functions — formatters, validators, helpers    
         
 

 
Core Modules
1. State Engine
The state engine is the single source of truth for all application data. All user data lives in one JavaScript object (window.state) that is serialised to localStorage on every change.
 
loadState()
Reads raw JSON from localStorage under the key hisabKitabState and hydrates the global window.state object. Falls back to IndexedDB if localStorage is unavailable. On first launch, initialises sensible defaults (empty arrays for expenses/EMIs/goals, default settings). Also runs migration logic to handle state shape changes between versions.
 
_loadStateFromRaw(raw)
Internal parser called by loadState(). Deserialises the JSON string, validates critical fields, and applies forward-migration patches — for example, converting legacy flat expense objects to the normalised schema used in v14+. Handles malformed data gracefully without crashing.
 
saveState()
Debounced save function (300ms delay) that serialises window.state to JSON and writes it to localStorage. The debounce prevents excessive writes when rapid UI changes occur (e.g., typing in a form). Automatically triggers Supabase cloud sync after saving if configured.
 
saveStateNow()
Synchronous, non-debounced variant of saveState(). Used for critical operations — PIN changes, encryption toggles, data imports — where immediate persistence is required before any async code runs.
 
_IDB (IndexedDB Interface)
An IIFE-encapsulated IndexedDB wrapper providing get(key) and set(key, value) methods as fallback storage when localStorage quota is exceeded. Used transparently by loadState() and saveStateNow().
 

 
2. Navigation System
showPage(id, btn)
The central navigation controller. Hides all .page divs and shows only the requested one by toggling the .active class. Updates the active state of the corresponding .nav-item in the sidebar. Triggers the appropriate render function for the newly visible page (e.g., renderExpenses() when navigating to the Expenses page). Uses a dirty-page tracking set to avoid re-rendering pages whose data has not changed.
 
openModal(id) / closeModal(id)
Shows and hides modal overlay dialogs by toggling the .open class on .modal-overlay elements. Handles focus trapping and body scroll locking to ensure accessible modal behaviour.
 

 
3. Dashboard Module
renderDashboard()
The primary dashboard renderer. Computes and displays five key financial metrics in the stat cards: Monthly Income, Total Expenses, Net Surplus/Deficit, Savings Balance, and Financial Health Score. Calls drawDonut(), drawTrendChart(), renderAlerts(), renderUpcomingDue(), and widget syncs in sequence to fully refresh the dashboard view.
 
drawDonut()
Renders the expense-breakdown donut chart on an HTML5 Canvas element. Groups expenses by category, calculates percentage shares, and draws arc segments with colour coding. Displays the net surplus or deficit value in the centre. Re-draws only when data has changed, using a hash-based change detection mechanism (_chartNeedsRedraw()).
 
drawTrendChart()
Plots a six-month income vs. expense trend as a dual-line chart on Canvas. Aggregates historical transactions by month and renders smooth Bézier curves with shaded fill areas beneath each line.
 
renderAlerts(budgetPct, balance, income)
Evaluates the current financial state against configurable thresholds and renders contextual alert banners: budget utilisation warnings (>80%), low savings alerts, EMI overload warnings (EMIs >40% of income), and deficit notifications.
 
calcHealthScore(income, expenses, budget, emiTotal, balance)
Computes a 0–100 Financial Health Score using a weighted rubric across four dimensions: savings rate (30%), budget adherence (25%), EMI-to-income ratio (25%), and emergency fund adequacy (20%). Returns both a numeric score and a descriptive label (Excellent / Good / Fair / Critical).
 
syncDashboardStatCards()
Updates the five summary stat cards on the dashboard with animated count-up transitions. Called after renderDashboard() to apply the numerical animation effect without re-rendering the full page.
 

 
4. Expense Management
saveExpense()
Reads the expense modal form fields (title, amount, category, type, due date, payment method, recurrence, notes), validates required inputs, and either creates a new expense object with a unique ID or updates an existing one in state.expenses. Calls saveState() and renderExpenses() on completion. Triggers an overspend check for the category.
 
editExpense(id)
Locates the expense with the given ID in state.expenses, pre-populates the expense modal form with its values, and opens the modal. Sets an internal _editingExpenseId flag so saveExpense() knows to update rather than insert.
 
deleteExpense(id)
Removes the expense with the given ID from state.expenses with a confirmation prompt. Calls saveState() and re-renders the expenses list and dashboard.
 
renderExpenses()
Renders the full filterable, paginated expense table. Applies the active month filter, search query, and category filter. Groups expenses by type (Fixed / Controllable / Savings). Supports load-more pagination via hkExpLoadMore() for performance with large datasets.
 
checkCategoryOverspend(expense)
After each new expense is saved, this function checks whether the current month's total spending in that category has exceeded the user-defined budget limit for that category. If breached, it surfaces a toast notification with the overage amount.
 

 
5. EMI & Loan Management
saveEMI()
Validates and persists a new EMI (Equated Monthly Instalment) record containing loan name, principal, interest rate, tenure, start date, and payment method. Stores the record in state.emis and re-renders the EMI page.
 
calcEMI()
Computes the monthly EMI payment using the standard amortisation formula:
EMI = P × r × (1+r)^n / ((1+r)^n - 1)
where P is principal, r is the monthly interest rate, and n is the tenure in months. Displays the result live in the EMI calculator widget as the user types.
 
renderEMIs()
Renders the active EMI list as progress cards. For each EMI, displays the monthly payment, total paid to date, remaining balance, and a progress bar showing completion percentage. Calculates and shows the total monthly EMI commitment across all active loans.
 
markEMIPaid(id)
Increments the paidMonths counter on an EMI record and logs the payment date. If all instalments are paid, marks the EMI as completed and moves it to a separate completed section. Triggers saveState() and re-renders.
 
deleteEMI(id)
Removes the EMI with the given ID after confirmation. Updates the dashboard EMI total accordingly.
 

 
6. Income Management
saveIncome()
Creates or updates an income source record in state.incomeSources. Supports multiple income types (Salary, Freelance, Business, Rental, Investment, Other) and frequency (Monthly, Weekly, Bi-weekly, Annually). Computes monthlyEquiv — the normalised monthly equivalent — and updates state.settings.income as the sum of all active sources.
 
renderIncomeSources()
Renders the income sources list with individual cards showing source name, type badge, monthly equivalent, and edit/delete controls. Displays the aggregate monthly income at the top.
 
syncIncomeStats()
Updates the income summary statistics without a full page re-render. Calculates total monthly income, YTD income, and average monthly income from the income sources array.
 

 
7. Savings Goals
editSavingsGoal(id) / saveGoal()
Manages savings goal records containing a target name, emoji, target amount, current saved amount, and target date. saveGoal() inserts or updates the goal and triggers a confetti animation (hkConfetti()) when a goal is marked as fully funded.
 
renderGoalCards()
Renders each savings goal as a progress card with a circular or linear progress indicator, the saved-vs-target ratio, estimated completion date, and quick-deposit controls. Highlights fully funded goals with a distinct completed state.
 

 
8. Emergency Fund Module
hkRenderEmergency()
Renders the Emergency Fund card on the dashboard. Calculates the recommended fund size as a configurable multiple (3–12 months) of monthly essential expenses. Displays the current fund balance, funding percentage, and colour-coded status indicator.
 
hkSaveEmg()
Persists emergency fund settings (target months, current balance) to state.emergencyFund and triggers a dashboard refresh.
 
hkConfirmDeposit()
Processes a deposit into the emergency fund. Adds the deposit amount to state.emergencyFund.balance, logs the transaction, and triggers a confetti animation if the target is reached for the first time.
 
hkApplyEmgPreset(months)
Applies a preset target multiplier (3, 6, or 12 months) to the emergency fund goal and recalculates the target amount based on current monthly expenses.
 

 
9. Analytics Module
renderAnalytics() / renderEnhancedAnalytics()
Renders the Analytics page with a suite of financial charts and insights. Includes: six-month income/expense trend, category breakdown donut, month-over-month change indicators, and top spending categories ranked by amount.
 
buildMonthData()
Aggregates state.expenses and state.incomeSources into a month-keyed lookup object used by all analytics charts. Handles multi-month ranges and normalises data for chart rendering.
 
drawSixMonthTrendChart()
Plots a detailed dual-line trend chart covering the past six months. Uses Canvas 2D API with anti-aliased Bézier curves, gradient fill, and interactive data-point markers.
 
drawBarChartCanvas(id, labels, values)
A reusable bar chart renderer that accepts an SVG or Canvas element ID, an array of category labels, and a corresponding values array. Draws proportionally scaled bars with value labels. Used for category-spend breakdown and month comparisons.
 

 
10. Investment Calculator
renderInvestment()
Renders the investment comparison tool that projects returns across multiple instrument types: Fixed Deposit, NSS (National Savings Scheme), Mutual Funds, Stocks, Real Estate, and Gold. Allows the user to select an investment type via card selection.
 
calcInvestment()
Computes the projected future value of an investment given principal, duration, and expected annual return rate using compound interest. Applies inflation adjustment to display real (inflation-adjusted) returns alongside nominal returns.
 
calcExtraMoney()
An "extra money" scenario planner. Given a one-off lump sum, it projects outcomes across all supported investment categories simultaneously, enabling direct comparison of where to best deploy surplus funds.
 
drawInvChart(data)
Renders a Canvas bar chart comparing projected returns across investment categories, with colour-coded bars and formatted PKR value labels.
 

 
11. Net Worth Tracker
saveAsset()
Saves an asset record (name, category, current value, acquisition date) to state.assets. Supports asset categories: Property, Vehicle, Gold, Cash, Investments, Other.
 
renderNetWorth()
Calculates and renders the total net worth as Assets minus Liabilities (EMI outstanding balances + debts). Displays a breakdown table of individual assets and liability items with their current values.
 

 
12. Budget Planner
The budget planner allows per-category monthly spending limits. It compares actual spending (from state.expenses) against configured limits and renders a utilisation bar for each category. Categories exceeding 100% utilisation are flagged in red.
 

 
13. Debt Planner
renderDebtCards()
Renders active debt records with outstanding balance, interest rate type (Fixed / Variable / KIBOR-linked), monthly payment, and remaining tenure. Calculates total debt burden and monthly commitment.
 
calcKIBORRate()
Computes the effective interest rate for variable-rate debts by summing the current KIBOR base rate and the user-entered spread. Updates the rate field live as inputs change.
 

 
14. Lending Ledger
renderLendingCards()
Renders the ledger of money lent to and borrowed from contacts. Separates entries into two tabs — "I Lent" and "I Borrowed" — with per-entry outstanding balance, due date, and WhatsApp reminder functionality.
 
sendLendingReminder()
Constructs a polite, formatted WhatsApp reminder message for an overdue lending entry and opens it via the wa.me deep link — allowing the user to send reminders directly without leaving the app.
 

 
15. Bill Split Module
buildPersonRows()
Dynamically renders input rows for each participant in a bill-split calculation. Supports both equal and custom (weighted) split modes.
 
shareSplitWA(sid)
Generates a formatted WhatsApp share message for a saved bill split, listing each person's share amount, and opens the WhatsApp deep link.
 

 
16. Currency Converter
Uses the Open Exchange Rates API (open.er-api.com) to fetch live PKR exchange rates. Caches the response for one hour to avoid redundant API calls. Renders a filterable rate table and a bidirectional converter widget. Falls back to cached rates on network failure.
 

 
17. Zakat Calculator
hkUpdateZakatPrices()
Fetches live gold and silver prices to compute the current Nisab threshold (the minimum wealth qualifying for Zakat). Updates the Nisab display in real time.
 
The Zakat calculator evaluates total zakatable assets (cash, gold, silver, investments, trade goods, receivables) against the Nisab threshold and computes 2.5% Zakat liability. Displays a detailed breakdown by asset category.
 

 
18. Calendar System
HisabKitab includes a dual calendar system supporting both Gregorian and Islamic (Hijri) calendars.
 
renderCalendar() / renderGeorgianCalendarWidget()
Renders an interactive monthly Gregorian calendar with event markers for: expense due dates, EMI payment dates, savings goals, user-set reminders, and Pakistani national holidays.
 
renderHijriCalendar()
Renders the Islamic calendar with full Hijri date calculation, month navigation, and markers for Islamic observances: Ramadan, Eid al-Fitr, Eid al-Adha, Muharram, Mawlid al-Nabi, and other significant dates.
 
gregorianToHijri(gy, gm, gd) / hijriToGregorian(hy, hm, hd)
Pure JavaScript implementations of Gregorian↔Hijri calendar conversion algorithms. Used to annotate the Gregorian calendar with Hijri dates and vice versa without any external library.
 
renderIslamicEvents() / renderDashboardIslamicEvents()
Renders upcoming Islamic events as a list with days-remaining countdowns, displayed both on the Calendar page and as a widget on the Dashboard.
 

 
19. Reminder System
saveReminder() / renderRemindersList()
Manages time-based reminders with title, date, category, and recurrence. renderRemindersList() sorts reminders chronologically and visually distinguishes overdue, due-today, and upcoming entries.
 
toggleReminderDone(id)
Marks a reminder as completed, applies a visual strike-through, and logs the completion timestamp.
 

 
20. Holiday Planner
saveHoliday() / renderHolidays()
Manages holiday/trip records with destination, dates, budget, and savings target. Calculates days remaining to the trip and monthly savings required to meet the holiday budget.
 

 
21. Tax Module
Provides a simplified Pakistani income tax calculator based on the current FBR (Federal Board of Revenue) tax slabs. Accepts annual taxable income and computes: total tax liability, effective tax rate, monthly tax deduction, and remaining net take-home pay.
 

 
22. Utility Bills Tracker
hkRenderCustomUtilities() / saveCustomUtility()
Tracks recurring utility bills (electricity, gas, water, internet, and custom bills) with amount, due date, and payment status. Renders bills as cards sorted by upcoming due date, with overdue bills highlighted in red.
 

 
23. AI Advisor Module
(Anthropic Claude API Integration)
The AI Advisor page provides a conversational financial advisor powered by the Anthropic Claude API. The user's financial state — income, expenses, EMIs, savings, goals, net worth, health score — is automatically compiled into a structured context object and injected into the system prompt. This gives the AI advisor full awareness of the user's financial situation without requiring manual input.
 
The advisor supports: spending analysis, savings recommendations, EMI restructuring advice, investment guidance, Zakat guidance, and general financial planning queries — all contextualised to the user's actual numbers.
 
Users supply their own Anthropic API key in Settings. The key is stored locally in state.settings.anthropicKey and never transmitted to any server other than the Anthropic API.
 

 
24. Voice Control
hkVoiceToggle()
Activates or deactivates the Web Speech API recognition engine. When active, listens for natural-language voice commands and parses them to trigger app functions — for example, "add expense 500 food" triggers a new expense entry, or "show analytics" navigates to the Analytics page.
 

 
25. Life Pages
Yaad Dahani (یاد دہانی) — Reminders & To-Do
saveYaadDahani() / renderYaadDahani()
A lightweight to-do and note reminder system separate from the financial reminder module. Supports priority levels, category tags, due dates, and completion toggles.
 
Sehat (صحت) — Health Tracker
renderSehat()
A daily health log tracking hydration (water intake with configurable daily goal), medication schedule, and general wellness notes. Persists daily health logs keyed by date in state.sehatLog.
 
addWater() / removeWater()
Increments or decrements the water intake counter for the current day. Triggers a celebration animation when the daily goal is met.
 
addMedicine() / toggleMed(medId)
Adds a medication to the daily schedule and toggles its taken/not-taken status. Logs completion timestamps for adherence tracking.
 
Madad (مدد) — Emergency Contacts
saveMadadContact() / renderMadad()
Manages an emergency contacts directory with name, relationship, phone number, and category (Medical, Police, Family, etc.). One-tap calling and WhatsApp links are rendered for each contact.
 
Notes (نوٹس) — Daily Journal
saveNote() / renderNotes()
A rich-text daily journal module. Notes are stored with timestamps and support category tagging. The notes list is searchable and filterable by date range.
 

 
26. FAB Quick-Add System
The Floating Action Button (FAB) group (#fab5) provides instant access to nine frequently used actions from any page without opening the sidebar.
 
hkFabToggle(force)
Toggles the Quick-Add modal open or closed. When opened, initialises the type-selection UI by calling hkFabSetType() to ensure one type is always visually highlighted. Focuses the amount input automatically for immediate entry.
 
hkFabSetType(type)
Switches the Quick-Add modal between three entry modes: Expense, Income, and Saving. Updates the pill button active state with colour-coded styling (red for Expense, green for Income, blue for Saving) and shows/hides the relevant form rows for each mode.
 
hkFabSave()
Processes the Quick-Add form submission. Depending on the selected type: creates a new expense in state.expenses, creates a new income source in state.incomeSources, or increments state.settings.savings. Calls saveState() and triggers a full master sync after saving.
 

 
Security System
PIN Lock
showPinLock(successCallback) / hidePinLock()
Renders a 4-digit PIN entry overlay that blocks access to the application until the correct PIN is entered. successCallback is invoked on successful PIN verification, enabling the caller to proceed with a protected operation.
 
pinKey(digit) / pinBackspace() / processPinEntry()
Handle PIN input digit-by-digit. processPinEntry() hashes the entered PIN with a stored salt using RC4+Base64 and compares against the stored hash. Implements a lockout policy (30-second delay after 5 failed attempts).
 
hashPin(pin, salt)
Hashes a PIN using RC4 stream cipher keyed with a random salt, then Base64-encodes the result. Used for both PIN setup and verification — the raw PIN is never stored.
 
Data Encryption
encryptData(plaintext, passphrase) / decryptData(ciphertext, passphrase)
Implements RC4 stream cipher encryption/decryption on the entire state JSON string. When encryption is enabled, all saveState() calls encrypt the JSON before writing to localStorage. All loadState() calls require successful decryption before hydrating window.state.
 
rc4Cipher(key, data) / strToB64(str) / b64ToStr(b64)
Low-level cryptographic primitives. rc4Cipher() implements the RC4 key-scheduling and pseudo-random generation algorithm. strToB64()/b64ToStr() provide Base64 encoding/decoding for binary-safe localStorage storage.
 
Biometric Authentication
hkBiometricRegister() / hkBiometricUnlock()
Implement WebAuthn (FIDO2) biometric authentication using the browser's PublicKeyCredential API. hkBiometricRegister() creates a passkey credential bound to the device's authenticator (fingerprint sensor, Face ID, Windows Hello). hkBiometricUnlock() challenges the authenticator and, on success, invokes the pending unlock callback — bypassing PIN entry on supported devices.
 
hkBiometricAvailable()
Detects whether the current browser and device support WebAuthn platform authenticators. Returns a boolean used to conditionally show/hide biometric UI controls.
 

 
Supabase Cloud Sync
The Supabase sync layer is a self-contained IIFE that communicates with a Supabase PostgreSQL database exclusively via the PostgREST REST API — no Supabase SDK is required.
 
Configuration
Credentials (url and key) are hardcoded as defaults and stored in localStorage under hk_sb_config_v2. Every visitor is auto-connected using the embedded credentials. The device is identified by a unique user_id (hk_<timestamp>_<random>) stored in localStorage, ensuring multi-device data isolation within the shared database.
 
REST Helpers
_hdrs(includeContentType)
Constructs the HTTP headers for Supabase REST requests. Always includes apikey and Authorization: Bearer with the JWT anon key. Content-Type: application/json is included only for write requests (POST/PATCH) — omitting it from GET and DELETE requests prevents unnecessary CORS preflight rejections.
 
_fetchWithRetry(url, opts, retries)
Wraps the native fetch() with automatic retry logic. On network failure (Failed to fetch), retries the request up to 3 times with a 1.2-second delay between attempts. Eliminates transient connectivity errors causing permanent sync failure.
 
_upsert(table, rows) / _select(table, extra) / _deleteAll(table)
The three fundamental database operations. _upsert() uses Prefer: resolution=merge-duplicates to perform an INSERT-or-UPDATE. _select() queries rows filtered by user_id. _deleteAll() deletes all rows for the current device from a table before re-inserting fresh data.
 
Data Mapping
Twelve mapper functions (_mapExpenses(), _mapEmis(), _mapSettings(), etc.) transform the flat window.state object into normalised row arrays matching the Supabase table schemas. Each mapper extracts only the fields relevant to its table, ensuring clean, strongly-typed database rows.
 
Sync Operations
_doPush(showToast)
Serialises all application state into database rows and upserts them across all twelve tables in parallel using Promise.all(). On success, updates the last-sync timestamp. On failure, surfaces a descriptive error message distinguishing between CORS/network errors, missing tables (42P01), RLS policy violations (42501), and authentication failures (401/403).
 
_doPull(showToast)
Reads all rows for the current device from the database and merges them back into window.state. Calls saveState(), then triggers a full UI re-render to reflect the pulled data. Used for cross-device sync and data recovery.
 

 
UI & Theming Engine
Theme System
HisabKitab ships with 12 colour themes, each defined as a complete set of CSS Custom Properties on [data-theme]:
 
Theme	Description
dark	Default deep navy — easy on the eyes
light	Clean white with blue accents
saffron-dusk	Warm amber inspired by Pakistani sunsets
jade-circuit	Emerald cyber-tech green
crimson-forge	Industrial red — bold and striking
polar-ice	Arctic blue clarity
volcanic-ash	Dark ash with molten orange
monsoon-teal	Deep aquatic teal
neon-bazar	Electric purple, inspired by Lahore nights
gilded-manuscript	Antique gold — Mughal manuscript aesthetic
desert-mirage	Dusty violet, inspired by the Thar Desert
naval-neon	Deep navy with cyan neon accents
 
applyColorThemeById(id, save) / applyStoredSettings()
Applies a theme by setting data-theme on <html> and updating the --accent CSS variable. applyStoredSettings() is called on page load to restore the previously selected theme, font size, high-contrast mode, and wallpaper.
 
Animated Wallpapers
Nine Canvas-based animated wallpapers render as a fixed background layer (#hk-wallpaper-canvas):
 
Wallpaper	Animation
starfield	Scrolling parallax star field
aurora	Soft Northern Lights colour waves
mughal	Geometric Mughal pattern — subtle tile shift
circuit	Flowing digital circuit traces
nebula	Deep-space nebula cloud animation
monsoon	Falling rain with puddle ripples
calligraphy	Fading Urdu/Arabic calligraphy strokes
dunes	Wind-driven sand dune undulation
matrix	Green falling code rain
 
Each wallpaper implements a tick(state) function called on every requestAnimationFrame. The wallpaper engine pauses all animations when document.hidden is true (tab not active) to conserve battery.
 
Sidebar System
hkSidebarToggle()
Collapses the sidebar to a 56px icon-only strip, reclaiming horizontal screen space. Persists the collapsed state in localStorage and adjusts body.paddingLeft to match.
 
hkSidebarClose() / hkSidebarOpen()
Fully hides (translateX(-100%)) or restores the sidebar on both desktop and mobile. On desktop, a floating ☰ reopener tab slides in from the left edge when the sidebar is hidden, providing a persistent way to restore navigation. The hidden state survives page refresh via localStorage.
 

 
Data Management
Backup & Restore
exportJSON()
Serialises the entire window.state object to a JSON file and triggers a browser download. The export includes all expenses, EMIs, income sources, savings goals, debts, assets, reminders, and settings — enabling a full data backup.
 
exportCSV()
Exports the expenses array as a CSV file with columns: Date, Description, Category, Type, Amount, Payment Method. Compatible with Excel, Google Sheets, and any standard spreadsheet application.
 
importData(input)
Reads a JSON file from a file input element, validates its structure, and merges it into the current window.state. Prompts for confirmation before overwriting existing data.
 
confirmReset()
Presents a confirmation dialog and, on approval, clears all application data from localStorage and reloads the page to a clean initial state.
 
Report Generation
hkGenerateHTMLReport() / hkGenerateWordReport()
Compile a full financial report — including income overview, expense detail table, active EMIs, and savings goals — and export it as either a styled HTML file or a .doc Word-compatible document. Reports include the user's inflation rate, budget utilisation, and savings rate.
 

 
Function Reference
 
Function	Module	Purpose
loadState()	State	Load data from localStorage/IDB
saveState()	State	Debounced state persistence
saveStateNow()	State	Immediate state persistence
showPage(id, btn)	Navigation	Page switching and render dispatch
openModal(id)	UI	Show a modal overlay
closeModal(id)	UI	Hide a modal overlay
renderDashboard()	Dashboard	Full dashboard refresh
drawDonut()	Charts	Expense breakdown donut chart
drawTrendChart()	Charts	6-month income/expense trend
calcHealthScore()	Dashboard	0–100 financial health score
saveExpense()	Expenses	Create/update an expense
renderExpenses()	Expenses	Render filtered expense list
saveEMI()	EMI	Save a loan record
calcEMI()	EMI	Compute monthly instalment
renderEMIs()	EMI	Render active loan cards
saveIncome()	Income	Save an income source
renderIncomeSources()	Income	Render income list
hkRenderEmergency()	Emergency	Render emergency fund card
hkFabToggle()	FAB	Open/close Quick-Add modal
hkFabSetType(type)	FAB	Switch expense/income/saving mode
hkFabSave()	FAB	Submit Quick-Add entry
renderAnalytics()	Analytics	Full analytics page render
calcInvestment()	Investment	Project investment returns
renderNetWorth()	Net Worth	Compute and display net worth
renderHijriCalendar()	Calendar	Render Islamic calendar
gregorianToHijri()	Calendar	Date conversion algorithm
hkVoiceToggle()	Voice	Toggle speech recognition
encryptData()	Security	RC4 encrypt state JSON
decryptData()	Security	RC4 decrypt state JSON
hashPin()	Security	Hash PIN with salt
hkBiometricRegister()	Security	Register WebAuthn credential
hkBiometricUnlock()	Security	Authenticate via biometrics
_doPush()	Sync	Push state to Supabase
_doPull()	Sync	Pull state from Supabase
_fetchWithRetry()	Sync	Fetch with 3-attempt retry
applyColorThemeById()	Theme	Apply a colour theme
applyWallpaperById()	Theme	Apply animated wallpaper
hkSidebarClose()	Sidebar	Fully hide sidebar
hkSidebarOpen()	Sidebar	Restore hidden sidebar
exportJSON()	Backup	Download full data backup
exportCSV()	Backup	Download expenses as CSV
importData()	Backup	Restore from JSON backup
showToastMsg(msg, type)	UI	Display notification toast
uid()	Utility	Generate unique record ID
fmt(n)	Utility	Format number as PKR string
hkConfetti(el)	UI	Trigger confetti celebration
hkCountUp(el, val)	UI	Animated number count-up
 

 
Deployment
Vercel (Recommended)
         
    project-root/    
    ├── index.html      ← The entire application    
    └── vercel.json     ← Routing configuration    
         
 
`vercel.json`
         
    {    
      "rewrites": [    
        { "source": "/(.*)", "destination": "/index.html" }    
      ]    
    }    
         
 
1. Push both files to a GitHub repository
2. Import the repository into Vercel
3. Deploy — no build step required
4. Share the Vercel URL; Supabase auto-connects for every visitor
 
Local Usage
Simply open index.html in any modern browser. No web server is required for core functionality.
 

 
Technology Stack
 
Technology	Usage
HTML5 / CSS3	Structure and styling — no framework
Vanilla JavaScript (ES6+)	All application logic
Canvas 2D API	Charts, donut, trend, bar, wallpapers
Web Speech API	Voice command recognition
WebAuthn / FIDO2	Biometric authentication
IndexedDB	Overflow storage fallback
localStorage	Primary state persistence
Supabase PostgREST	Cloud sync REST API
Anthropic Claude API	AI financial advisor
Open Exchange Rates	Live PKR currency rates
Font Awesome	Icon library (CDN)
Google Fonts	Syne + IBM Plex Mono typefaces
 

 
Browser Compatibility
 
Browser	Support
Chrome 90+	✅ Full (including biometrics)
Firefox 88+	✅ Full
Safari 15+	✅ Full (including Face ID via WebAuthn)
Edge 90+	✅ Full
Mobile Chrome	✅ Full (Android biometrics)
Mobile Safari	✅ Full (Face ID / Touch ID)
 

 
License
This project is proprietary software. All rights reserved. Unauthorised reproduction, distribution, or modification without explicit written permission is prohibited.
 

 
Supabase Database Schema
To enable cloud sync, execute the following SQL in your Supabase project's SQL Editor (Database → SQL Editor → New Query). This creates all required tables and configures Row Level Security policies for anonymous public access.
 
         
    -- ═══════════════════════════════════════════════════    
    --  HisabKitab — Full Database Schema    
    --  Paste this into Supabase SQL Editor and click Run    
    -- ═══════════════════════════════════════════════════    
         
    -- 1. Expenses    
    CREATE TABLE IF NOT EXISTS hk_expenses (    
      id            TEXT PRIMARY KEY,    
      user_id       TEXT NOT NULL,    
      title         TEXT,    
      amount        NUMERIC,    
      category      TEXT,    
      type          TEXT,    
      date          TEXT,    
      payment       TEXT,    
      notes         TEXT,    
      recurring     TEXT,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_expenses ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_expenses" ON hk_expenses FOR ALL TO anon USING (true) WITH CHECK (true);    
         
    -- 2. EMIs / Loans    
    CREATE TABLE IF NOT EXISTS hk_emis (    
      id            TEXT PRIMARY KEY,    
      user_id       TEXT NOT NULL,    
      name          TEXT,    
      principal     NUMERIC,    
      rate          NUMERIC,    
      tenure        INT,    
      paid_months   INT DEFAULT 0,    
      start_date    TEXT,    
      payment       TEXT,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_emis ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_emis" ON hk_emis FOR ALL TO anon USING (true) WITH CHECK (true);    
         
    -- 3. Income Sources    
    CREATE TABLE IF NOT EXISTS hk_income_sources (    
      id            TEXT PRIMARY KEY,    
      user_id       TEXT NOT NULL,    
      name          TEXT,    
      type          TEXT,    
      amount        NUMERIC,    
      frequency     TEXT,    
      monthly_equiv NUMERIC,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_income_sources ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_income" ON hk_income_sources FOR ALL TO anon USING (true) WITH CHECK (true);    
         
    -- 4. Savings Goals    
    CREATE TABLE IF NOT EXISTS hk_savings_goals (    
      id            TEXT PRIMARY KEY,    
      user_id       TEXT NOT NULL,    
      name          TEXT,    
      emoji         TEXT,    
      target        NUMERIC,    
      saved         NUMERIC DEFAULT 0,    
      target_date   TEXT,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_savings_goals ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_goals" ON hk_savings_goals FOR ALL TO anon USING (true) WITH CHECK (true);    
         
    -- 5. Assets (Net Worth)    
    CREATE TABLE IF NOT EXISTS hk_assets (    
      id            TEXT PRIMARY KEY,    
      user_id       TEXT NOT NULL,    
      name          TEXT,    
      category      TEXT,    
      value         NUMERIC,    
      date          TEXT,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_assets ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_assets" ON hk_assets FOR ALL TO anon USING (true) WITH CHECK (true);    
         
    -- 6. Debts    
    CREATE TABLE IF NOT EXISTS hk_debts (    
      id            TEXT PRIMARY KEY,    
      user_id       TEXT NOT NULL,    
      name          TEXT,    
      amount        NUMERIC,    
      rate          NUMERIC,    
      rate_type     TEXT,    
      monthly       NUMERIC,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_debts ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_debts" ON hk_debts FOR ALL TO anon USING (true) WITH CHECK (true);    
         
    -- 7. Lending Ledger    
    CREATE TABLE IF NOT EXISTS hk_lending (    
      id            TEXT PRIMARY KEY,    
      user_id       TEXT NOT NULL,    
      person        TEXT,    
      amount        NUMERIC,    
      direction     TEXT,   -- 'lent' or 'borrowed'    
      due_date      TEXT,    
      notes         TEXT,    
      settled       BOOLEAN DEFAULT FALSE,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_lending ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_lending" ON hk_lending FOR ALL TO anon USING (true) WITH CHECK (true);    
         
    -- 8. Reminders    
    CREATE TABLE IF NOT EXISTS hk_reminders (    
      id            TEXT PRIMARY KEY,    
      user_id       TEXT NOT NULL,    
      title         TEXT,    
      date          TEXT,    
      category      TEXT,    
      recurrence    TEXT,    
      done          BOOLEAN DEFAULT FALSE,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_reminders ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_reminders" ON hk_reminders FOR ALL TO anon USING (true) WITH CHECK (true);    
         
    -- 9. Upcoming Events / Holidays    
    CREATE TABLE IF NOT EXISTS hk_upcoming (    
      id            TEXT PRIMARY KEY,    
      user_id       TEXT NOT NULL,    
      title         TEXT,    
      emoji         TEXT,    
      date          TEXT,    
      budget        NUMERIC DEFAULT 0,    
      saved         NUMERIC DEFAULT 0,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_upcoming ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_upcoming" ON hk_upcoming FOR ALL TO anon USING (true) WITH CHECK (true);    
         
    -- 10. Emergency Fund    
    CREATE TABLE IF NOT EXISTS hk_emergency_fund (    
      id            TEXT PRIMARY KEY,   -- always the user_id    
      user_id       TEXT NOT NULL,    
      balance       NUMERIC DEFAULT 0,    
      target        NUMERIC DEFAULT 0,    
      target_months INT DEFAULT 6,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_emergency_fund ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_emg" ON hk_emergency_fund FOR ALL TO anon USING (true) WITH CHECK (true);    
         
    -- 11. Settings (one row per device)    
    CREATE TABLE IF NOT EXISTS hk_settings (    
      user_id       TEXT PRIMARY KEY,    
      income        NUMERIC DEFAULT 0,    
      budget        NUMERIC DEFAULT 0,    
      savings       NUMERIC DEFAULT 0,    
      currency      TEXT DEFAULT 'PKR',    
      theme         TEXT DEFAULT 'dark',    
      font_size     TEXT DEFAULT 'md',    
      high_contrast BOOLEAN DEFAULT FALSE,    
      accent        TEXT,    
      wallpaper     TEXT,    
      inflation     NUMERIC DEFAULT 6,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_settings ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_settings" ON hk_settings FOR ALL TO anon USING (true) WITH CHECK (true);    
         
    -- 12. Category Budgets    
    CREATE TABLE IF NOT EXISTS hk_category_budgets (    
      id            TEXT PRIMARY KEY,    
      user_id       TEXT NOT NULL,    
      category      TEXT,    
      monthly_limit NUMERIC DEFAULT 0,    
      updated_at    TIMESTAMPTZ DEFAULT NOW()    
    );    
    ALTER TABLE hk_category_budgets ENABLE ROW LEVEL SECURITY;    
    CREATE POLICY "anon_all_budgets" ON hk_category_budgets FOR ALL TO anon USING (true) WITH CHECK (true);    
         
 

 
Troubleshooting
Sync Errors
 
Error	Cause	Fix
Failed to fetch	CORS preflight rejected or no internet	Ensure the anon key starts with eyJ — not sb_publishable_. Check internet connectivity.
42P01 — relation does not exist	Tables not created yet	Run the SQL Schema above in Supabase SQL Editor.
42501 — row-level security	RLS enabled but no policies	Run the SQL Schema — policies are included.
401 / 403 Unauthorized	Wrong API key type	Use the Legacy anon JWT key (eyJ...) from Supabase → Settings → API.
Sync loaded. Ready: false	URL or key missing / invalid	Open Settings → Supabase Sync, verify URL and key, click Save.
 
PIN & Encryption
 
Problem	Fix
Forgot PIN	Tap "Forgot PIN?" on the lock screen. This clears the PIN — you will need to set a new one in Settings → Security. Note: data is not deleted.
Forgot encryption passphrase	Data cannot be recovered without the passphrase — this is by design. Use Export JSON regularly as a plaintext backup before enabling encryption.
Biometric not showing	Ensure the device supports WebAuthn platform authenticators. Register biometrics in Settings → Security before attempting to use them for unlock.
 
Display Issues
 
Problem	Fix
Charts not rendering	Ensure the browser is not blocking Canvas 2D. Disable privacy extensions temporarily to test.
Sidebar stuck on mobile	Clear site data (Application → Storage → Clear) and reload.
Wallpaper animation laggy	Reduce animation in Settings → Appearance, or switch to a static theme.
Data appears blank after import	Ensure the JSON file was exported from HisabKitab (not edited manually). Check browser console for _loadStateFromRaw error.
 

 
State Object Reference
The entire application state is a single JSON object. Key structure:
 
         
    window.state = {    
      settings: {    
        income:        Number,   // Monthly net income (PKR)    
        budget:        Number,   // Monthly expense budget (PKR)    
        savings:       Number,   // Current savings balance (PKR)    
        currency:      String,   // Display currency code ("PKR")    
        theme:         String,   // Active colour theme ID    
        fontSize:      String,   // "sm" | "md" | "lg" | "xl"    
        highContrast:  Boolean,    
        accent:        String,   // CSS colour value    
        wallpaper:     String,   // Wallpaper ID or null    
        inflation:     Number,   // Annual inflation rate (%)    
        name:          String,   // User's display name    
        anthropicKey:  String,   // Anthropic API key (local only)    
      },    
      expenses:       [ ExpenseRecord ],    
      emis:           [ EMIRecord ],    
      incomeSources:  [ IncomeSourceRecord ],    
      savingsGoals:   [ SavingsGoalRecord ],    
      assets:         [ AssetRecord ],    
      debts:          [ DebtRecord ],    
      lending:        [ LendingRecord ],    
      reminders:      [ ReminderRecord ],    
      upcoming:       [ UpcomingRecord ],    
      emergencyFund:  { balance, target, targetMonths },    
      budgets:        { [category]: Number },    
      sehatLog:       { [dateStr]: SehatDayRecord },    
      notes:          [ NoteRecord ],    
      yaadDahani:     [ YaadDahaniRecord ],    
      madadContacts:  [ ContactRecord ],    
      customUtilities:[ UtilityRecord ],    
      splits:         [ SplitRecord ],    
    }    
         
 

 
Changelog
v16 (Current)
● ✅ Sidebar fully hideable on desktop via ✕ button
● ✅ Floating ☰ reopener tab on left edge when sidebar hidden
● ✅ Hide/show state persists across page refreshes via localStorage
● ✅ Sidebar close button now rendered on all screen sizes (not mobile-only)
● ✅ FAB + button fixed — click-outside no longer immediately closes modal
● ✅ FAB type selection (Expense/Income/Saving) visually clear with colour-coded pill tabs
● ✅ FAB modal responsive — uses min(320px, 100vw - 28px) to prevent off-screen clipping
● ✅ Mobile sidebar slide-out repaired — GPU compositing translateZ(0) no longer overrides translateX(-100%)
 
v15
● ✅ Supabase JWT anon key hardcoded — auto-connects every visitor
● ✅ Settings panel pre-filled with Supabase credentials
● ✅ Content-Type removed from GET/DELETE headers (prevented CORS preflight)
● ✅ _fetchWithRetry() — 3-attempt retry with 1.2s backoff
● ✅ Descriptive sync error messages with exact remediation steps
 
v14
● ✅ Supabase cloud sync introduced (12 tables)
● ✅ Normalised state sync — device UID based isolation
● ✅ Pull-on-load for cross-device data access
● ✅ Full master sync on every saveState() call
 
Prior Versions (v1–v13)
● Progressive addition of all finance modules (expenses, EMI, income, goals, analytics)
● Islamic calendar + prayer times integration
● RC4 data encryption + PIN lock system
● WebAuthn biometric unlock
● AI advisor (Anthropic Claude)
● Voice control (Web Speech API)
● 12 colour themes + 9 animated wallpapers
● PWA offline capability
● CSV/JSON export and import
● Pakistani tax calculator and Zakat module
 

 
Project Structure
Since HisabKitab is a single-file application, the logical structure within index.html is:
 
         
    index.html    
    │    
    ├── <head>    
    │   ├── Meta tags, viewport, PWA manifest link    
    │   └── Early theme inject (prevents flash of wrong theme)    
    │    
    ├── <style> — ~2,800 lines    
    │   ├── CSS Custom Properties (all 12 themes)    
    │   ├── Layout: sidebar, main content, responsive breakpoints    
    │   ├── Components: cards, modals, forms, buttons, toasts    
    │   ├── Page-specific styles (expenses, calendar, analytics…)    
    │   ├── FAB system styles    
    │   ├── Wallpaper canvas styles    
    │   └── Animation keyframes    
    │    
    ├── <body>    
    │   ├── #hk-wallpaper-canvas (full-page animated background)    
    │   ├── .sidebar (navigation + branding + collapse controls)    
    │   ├── .main-content    
    │   │   ├── #page-dashboard    
    │   │   ├── #page-expenses    
    │   │   ├── #page-emi    
    │   │   ├── #page-income    
    │   │   ├── #page-savingsgoals    
    │   │   ├── #page-analytics    
    │   │   ├── #page-investment    
    │   │   ├── #page-networth    
    │   │   ├── #page-calendar    
    │   │   ├── #page-debtplanner    
    │   │   ├── #page-lending    
    │   │   ├── #page-budgets    
    │   │   ├── #page-splits    
    │   │   ├── #page-currency    
    │   │   ├── #page-zakat    
    │   │   ├── #page-tax    
    │   │   ├── #page-utility    
    │   │   ├── #page-aiadvisor    
    │   │   ├── #page-settings    
    │   │   ├── #page-backup    
    │   │   ├── #page-yaaddahani    
    │   │   ├── #page-sehat    
    │   │   ├── #page-madad    
    │   │   └── #page-notes    
    │   ├── Modal overlays (expense, EMI, goal, income, PIN lock…)    
    │   ├── #fab5 (floating action button group)    
    │   ├── #fabModal (quick-add panel)    
    │   ├── #hkSbReopenTab (sidebar reopener tab)    
    │   └── Toast notification container    
    │    
    └── <script> — ~17,500 lines    
        ├── Block 1:  Crypto utilities (RC4, Base64, encrypt/decrypt)    
        ├── Block 2:  PIN & biometric security system    
        ├── Block 3:  State engine (loadState, saveState, IDB)    
        ├── Block 4:  Core utilities (uid, fmt, fmtShort)    
        ├── Block 5:  Navigation (showPage, openModal, closeModal)    
        ├── Block 6:  Settings (loadSettings, saveSettings, applyTheme)    
        ├── Block 7:  Expense CRUD + rendering    
        ├── Block 8:  EMI calculator + rendering    
        ├── Block 9:  Holiday & upcoming event management    
        ├── Block 10: Net worth + asset management    
        ├── Block 11: Analytics + chart rendering    
        ├── Block 12: Investment calculator    
        ├── Block 13: Calendar (Gregorian + Hijri)    
        ├── Block 14: Currency converter    
        ├── Block 15: Dashboard (renderDashboard, renderAlerts, health score)    
        ├── Block 16: Backup/export/import    
        ├── Block 17: Sidebar toggle + mobile nav    
        ├── Block 18: Master sync + aura system    
        ├── Block 19: Enhanced UI cards (EMI, goals, debts, lending)    
        ├── Block 20: Enhanced analytics    
        ├── Block 21: Enhanced dashboard widgets    
        ├── Block 22: Sparklines + activity dots + count-up    
        ├── Block 23: Confetti + wallpaper engine    
        ├── Block 24: Emergency fund module    
        ├── Block 25: Zakat calculator + Islamic events    
        ├── Block 26: Auxiliary features (emoji, splits, utilities)    
        ├── Block 27: Wallpaper animation functions    
        ├── Block 28: Life pages (Yaad Dahani, Sehat, Madad, Notes)    
        ├── Block 29: Supabase sync layer (IIFE)    
        ├── Block 30: Theme grid + wallpaper grid renderer    
        ├── Block 31: FAB quick-add system    
        ├── Block 32: Sidebar close/open + reopener tab    
        └── Block 33: Initialisation (initAll, applyStoredSettings)    
         
 

 
Contributing
Contributions are welcome via pull request. Please observe the following standards:
 
1. Single file — all changes must remain within index.html. Do not introduce external JS or CSS files.
2. No dependencies — do not add NPM packages, CDN scripts (except those already present), or build tools.
3. State safety — all new data must be added to window.state with appropriate defaults, migration logic in _loadStateFromRaw(), and Supabase mapper functions.
4. Mobile-first — all UI must be tested at 375px width (iPhone SE) before submission.
5. RTL awareness — do not hard-code left/right directional styles; use start/end where possible to preserve Urdu RTL compatibility.
6. Performance — new render functions must implement dirty-checking to avoid unnecessary redraws.
 

 
HisabKitab — Smart Finance, Pakistani Roots.
Built with ❤️ for the people of Pakistan.
