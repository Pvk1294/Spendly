<div align="center">

# Spendly

### Your money, understood automatically.

*A premium personal finance app for India — tracks every transaction from your bank SMS, categorizes it intelligently, and turns your spending into clear, actionable insight. No bank logins. No manual entry. No noise.*

[![Version](https://img.shields.io/badge/version-1.5-8B5CF6?style=flat-square)](.)
[![Platform](https://img.shields.io/badge/platform-Android-3DDC84?style=flat-square&logo=android)](.)
[![React Native](https://img.shields.io/badge/React%20Native-0.74-61DAFB?style=flat-square&logo=react)](.)
[![TypeScript](https://img.shields.io/badge/TypeScript-strict-3178C6?style=flat-square&logo=typescript)](.)

</div>

---

## What is Spendly?

Spendly is a personal finance app built for how Indian banking actually works — through SMS.

Every UPI payment, every card swipe, every bank transfer generates a text message. Spendly reads those messages silently, in the background, and converts them into a clean expense ledger. By the time you open the app after a purchase, the transaction is already there — categorized, timestamped, and counted toward your monthly totals.

No spreadsheets. No syncing your bank account to a third party. No remembering to log anything.

---

## The Problem

Personal finance apps have a retention problem. Most people try one, use it for a week, and stop. The friction is always the same.

**Manual entry is abandoned almost immediately.** Logging every purchase is a habit that almost no one maintains. The app becomes a task, not a tool.

**Account aggregators ask for too much trust.** Apps that connect directly to your bank require credentials or OAuth access to a service you've never heard of. The data goes somewhere. Most users aren't comfortable with that, and rightfully so.

**Finance dashboards feel like work.** Dense charts, confusing categories, endless setup. Apps designed for power users end up unused by everyone else.

**Investment and spending live in separate worlds.** Users switch between a broking app for SIPs and a finance app for expenses. There's no unified picture.

The result: most people in India have no idea where their money goes until the credit card bill arrives.

---

## The Solution

Spendly solves this by removing the manual layer entirely.

**SMS-based automatic tracking** means transactions appear in the app the moment they happen — no tapping, no entering amounts, no choosing categories manually every time.

**Merchant learning** means that once you categorize Swiggy as Food, every future Swiggy order is categorized for you. The app gets smarter with each correction, until corrections are rare.

**Calm, clean analytics** surface the information that changes behavior — monthly category breakdowns, trend lines, and a clear view of where the money went — without overwhelming the user with data they didn't ask for.

**Investment visibility** keeps SIPs and lump-sums in a separate, dedicated view so long-term wealth building never gets confused with day-to-day spending.

**Privacy by design** means raw SMS never leaves the device. Only the parsed result — amount, merchant, category, date — is ever stored on a server.

---

## Key Features

### Automatic Expense Detection
Bank SMS arrives, Spendly parses it in the background, and a new transaction appears in your ledger — silently and instantly. Covers UPI, debit cards, credit cards, and netbanking across all major Indian banks.

### Smart Categorization
Tag a merchant once. After that, every transaction from that merchant — no matter how the bank formats the name — is categorized automatically. No repetition.

### Spending Pattern Analysis
Monthly breakdowns by category, rolling trends, and date-range views. Designed to give you just enough insight to notice patterns, without overwhelming you with dashboards you'll stop reading.

### Investment Overview
A separate ledger for SIPs, lump-sums, and holdings. Your investment transfers stay out of your spending numbers so both views remain meaningful.

### Full Transaction History
Searchable, filterable, and editable. Add notes, correct categories, and manage transactions the way you'd want to — with a clean mobile interface that doesn't feel like a spreadsheet.

### Guided Onboarding
A thoughtful onboarding experience explains what the app does and why it needs SMS access before asking for any permissions. First-time users understand the product before they commit to it.

### Custom Categories
Create categories with custom names, colors, and emoji icons. The experience is personal — your finances organized the way you think about them, not the way a generic template assumes you do.

### Offline-First
The app works without a network connection. Everything important lives on the device first and syncs when connectivity is available.

---

## Product Experience

**First launch** starts with an onboarding carousel — a few screens that explain the core idea before any sign-in or permissions are requested. Users understand what they're agreeing to.

**Sign-in** takes under a minute via OTP on WhatsApp or Google Sign-In.

**After permissions are granted**, the app performs a one-time import of the last 60 days of bank SMS history. A progress screen shows the import happening in real time. By the time it finishes, the user already has months of transactions waiting for them.

**The home screen** shows a spending summary for the current period, a category breakdown, and a feed of recent transactions. There are no empty states, no "add your first transaction" prompts. The data is already there.

**The transactions screen** is a scrollable, searchable list with category filters. Tapping any transaction shows the full detail and lets you edit the category, add a note, or delete it.

**The analysis screen** shows where the month's money went — by category, by week, with comparisons to prior periods. The charts are designed to be readable at a glance.

**The investments screen** is separate and intentional. SIPs and portfolio holdings are tracked as a distinct ledger. Long-term wealth and day-to-day cashflow don't bleed into each other.

**In-app onboarding** provides contextual guidance the first time a user encounters each major feature — not a pop-up tour, but lightweight prompts that appear when they're actually relevant.

---

## Design Philosophy

**Dark, premium UI.** A deep dark theme with a purple fintech accent. Feels modern, reduces eye strain, and matches the premium expectations of a product handling financial data.

**Calm over stimulating.** Finance apps have a tendency to fill every pixel with data. Spendly makes the opposite choice: show the number that matters, not every number that exists.

**Mobile-first interactions.** Every screen is built for thumb navigation. Bottom sheets, swipe gestures, haptic feedback, and smooth transitions make the app feel native — not a web page in a wrapper.

**Progressive disclosure.** Complex features are introduced when the user is ready for them, not on first launch. Onboarding is split into a pre-auth carousel and contextual in-app guidance so neither is overwhelming.

**Consistency.** One color palette. One type scale. One set of spacing rules. The visual language is consistent enough that users can predict what a new screen will look like before they navigate to it.

---

## Technical Highlights

| Area | Approach |
|---|---|
| **Mobile** | React Native 0.74, TypeScript strict, zero web views |
| **Local state** | Zustand for auth, preferences, and UI state |
| **Server state** | TanStack Query v5 with AsyncStorage persistence |
| **Animations** | Reanimated 3, Shopify Skia, Lottie |
| **SMS engine** | Custom regex parser (~570 lines) with per-bank patterns and confidence scoring |
| **Dedup** | Two-level: SMS-hash on device + composite key on server |
| **Merchant learning** | Per-user mapping table that auto-classifies future transactions |
| **Backend** | Node.js / Express / Prisma / PostgreSQL, feature-sliced modules |
| **Privacy** | On-device parsing; only structured fields reach the server |
| **AI trainer** | Python + Ollama (Mistral-7B) pipeline for expanding SMS pattern coverage |

---

## Architecture Overview

The system has three layers, each with a clear job.

**On-device (React Native)** — All SMS parsing happens here. A native Android listener intercepts new bank messages, runs them through a regex engine that extracts amount, merchant, type, and date, deduplicates against a 60-day hash window, and stores the result locally. The raw SMS never leaves the phone by default.

**Backend (Node / PostgreSQL)** — Stores structured transaction data, runs a second deduplication pass on composite keys, applies merchant-to-category mappings, and serves aggregated insights. The backend knows about transactions, not SMS.

**AI trainer (Python / Ollama)** — An offline pipeline that trains a custom Mistral-7B model on user-confirmed labelled examples. Not used at request time — runs asynchronously to improve pattern coverage over time without impacting app performance or battery life.

---

## Screenshots

> _Screenshots will be added here. Screens below represent the full navigation surface._

| | | |
|---|---|---|
| Onboarding carousel | Home dashboard | Spending analysis |
| Transaction list | Transaction detail | Category management |
| Investment portfolio | SMS sync progress | Profile & settings |

---

## Challenges Solved

**Parsing Indian bank SMS reliably.** Indian banking SMS is inconsistent — every bank uses different phrasing, different amount formats, different merchant name patterns. Some SMS that look like transactions are actually OTPs or limit-change notifications. The parser has to get this right on every message with no user intervention.

**Dedup across reinstalls.** Users reinstall apps. When they do, the 60-day import runs again. Without a robust deduplication system, every transaction would be doubled. Spendly uses a content-hash at the device level and a composite key at the server level to prevent this regardless of how many times the app is reinstalled.

**Making merchant learning feel magical.** The first time a user sees "SWIGGY*BGLR01" in their transaction list, they tag it as Food. The system has to correctly match that same merchant the next time it appears as "Swiggy Bangalore" or "SWIGGY" and apply the same category. Normalization, fuzzy matching, and a persistent per-user mapping table make this work.

**Keeping analytics fast without complex queries.** Insights that feel real-time — category totals, monthly trends, rolling comparisons — need to be computed efficiently over potentially thousands of transactions. The backend aggregation layer and client-side TanStack Query caching keep every screen responsive.

**Onboarding without friction.** The app needs SMS read permission, notification permission, and battery optimization exemption (for background SMS capture on aggressive Android ROMs). Asking for all three upfront is a recipe for abandonment. Spendly staggers permission requests with context screens that explain the value before the system dialog appears.

**OEM Android battery killers.** On Xiaomi, OnePlus, and similar devices with aggressive background process management, SMS listeners can be silenced without the user knowing. A dedicated battery optimization service detects this and requests the appropriate exemption — once, with a clear explanation.

---

## Version 1.5 Highlights

The v1.5 update was focused on experience quality and feature completeness across the full user journey.

- **Restored and expanded onboarding** — Pre-auth carousel rebuilt from scratch with a cleaner narrative flow and updated visuals
- **Contextual in-app onboarding** — Lightweight tutorial system that surfaces guidance when features are first encountered, not all at once
- **Custom category colors** — Full HSV color wheel for category personalization (previously limited to presets)
- **Analytics refinements** — Improved chart rendering, better date-range handling, and more legible category breakdowns
- **Investment section** — Enhanced portfolio view with better data entry and a cleaner layout
- **Navigation overhaul** — Reduced tap depth to key screens, smoother tab transitions
- **SMS edge cases** — Expanded bank pattern coverage and improved confidence scoring for ambiguous messages
- **Battery optimization** — More reliable background SMS capture on OEM Android builds

---

## Future Vision

**AI budgeting assistant.** A conversational layer that can answer questions like "how much did I spend on food last month compared to the month before?" and surface proactive alerts when spending is trending high.

**Smart recommendations.** Pattern detection that notices recurring subscriptions, flags unusual spend spikes, and suggests categories for review without requiring manual audit.

**Bank statement scanning.** OCR-based PDF import for banks that don't send transactional SMS reliably — credit card statements, salary slips, and passbook exports.

**Predictive analytics.** Month-end spend projections based on current trajectory. "At your current pace you'll spend ₹8,400 on food this month — ₹1,200 more than last month."

**iOS parity.** iOS does not expose SMS to third-party apps. The roadmap includes a share-extension approach for manually forwarding bank messages, or screenshot-based OCR parsing for iOS users.

**Collaborative finance.** Shared expense views for couples and roommates — visibility into joint spending without exposing individual transaction history.

---

## About the Project

Spendly is a solo end-to-end project — product, design, mobile, backend, AI pipeline, and deployment — built to solve a real and underserved problem in the Indian personal finance space.

The core thesis is that automatic tracking only works if it's also private. Every technical decision in the architecture reflects that: parse on-device, sync only structured data, never ask for bank credentials.

The product is in active use and under active development.

---

<div align="center">

**Built by Prateek Verma**

[pvk1294@gmail.com](mailto:pvk1294@gmail.com) · [info@digiexe.com](mailto:info@digiexe.com)

</div>
