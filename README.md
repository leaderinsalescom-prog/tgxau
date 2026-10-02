# GoldSignal Hub

**Project: GoldCopier Pro — Telegram-to-MT5 Auto Trade Copier**



---



**OVERVIEW**



Build a full-stack web application (mobile-first, responsive) that monitors a Telegram channel 24/7, parses XAUUSD trading signals, and automatically executes trades on a MetaTrader 5 account via a Python bridge. The app must never miss a signal entry or exit. Include a friendly step-by-step setup guide designed for non-technical users.



---



**TECH STACK**



- Frontend: React + Tailwind CSS (mobile-first)

- Backend: Supabase (database + edge functions)

- Telegram listener: Python (Telethon library) running on VPS

- MT5 bridge: Python (MetaTrader5 library) running on VPS

- Signal parsing: Claude AI API (claude-sonnet-4-6)



---



**CORE FUNCTIONALITY — SIGNAL PARSING**



Use this system prompt for the AI signal parser. Only act on messages that pass validation:



```

You are a XAUUSD signal parser. Detect and extract valid

signals regardless of format. Be intelligent — signals come

in dozens of styles, informal language, mixed languages,

emojis, and incomplete formatting.



VALID SIGNAL FORMATS TO DETECT:



FORMAT 1 - Clean structured:

"BUY XAUUSD 4303.6 TP1: 4305 TP2: 4307 SL: 4289"



FORMAT 2 - Zone entry:

"Buy Gold @4312.5-4302.5 SL: 4298 TP1: 4316"



FORMAT 3 - Underscore style:

"SELL_4340_4246 SL___4356 TP___4325"



FORMAT 4 - Inline compact:

"GOLD SELL 4317 TP 4297 SL 4327"



FORMAT 5 - Emoji heavy:

"🔻XAUUSD SELL NOW (4338) TARGET1 (4334) STOP (4249)"



FORMAT 6 - Zone keyword:

"SELL Zone: 4311-4314 SL: 4322 Targets: 4308 4305"



FORMAT 7 - Pip-based TP:

"BUY GOLD ZN:4323-4320 TP:30/50/100pips SL:4314"



FORMAT 8 - Natural language:

"Gold buy from 2650, targets 2660 2670 2680, stop 2640"



FORMAT 9 - Multi-line structured:

"XAUUSD BUY

Entry: 2345

TP1: 2350

TP2: 2360

TP3: 2370

SL: 2335"



FORMAT 10 - Slash separated:

"BUY GOLD 2310/2305 SL 2295 TP 2320/2330/2345"



FORMAT 11 - Arrow style:

"GOLD ▲ BUY 2290 → TP 2300 | SL 2280"



FORMAT 12 - Update/modification message:

"Move SL to entry on XAUUSD BUY"

→ parse as SL update command, not a new signal



FORMAT 13 - Partial close instruction:

"Close 50% XAUUSD now at 2345"

→ parse as position management command



FORMAT 14 - Range with pips:

"BUY GOLD zone 2300-2295, SL 30 pips below,

TP 50/100/150 pips"



FORMAT 15 - Informal/conversational:

"guys buy gold now around 2310 sl at 2295 tp 2330 2350"



FORMAT 16 - Forwarded/reposted signal:

"[Forwarded from XYZ channel] BUY XAUUSD 2310

SL 2295 TP 2330"

→ extract signal from forwarded content,

ignore channel attribution



FORMAT 17 - Table or listed format:

"Pair: XAUUSD

Direction: BUY

Entry: 2310

SL: 2290

TP1: 2325 TP2: 2340"



FORMAT 18 - Signal with risk warning:

"⚠️ HIGH RISK — BUY GOLD 2310 SL 2280 TP 2350 2380"

→ parse signal, flag risk as HIGH



FORMAT 19 - Close/cancel instruction:

"Cancel XAUUSD buy order" or "Close gold trade now"

→ parse as cancel/close command



FORMAT 20 - Non-English or mixed language:

"Gold خريد 2310 sl 2290 tp 2330"

or "BUY الذهب 2310"

→ detect BUY/SELL equivalent in Arabic/Urdu/mixed text



INTELLIGENT PARSING RULES:

• If direction is missing but context is clear, infer

  direction from SL/TP position relative to entry

• If entry is missing but message says NOW or MARKET,

  use current market price as entry

• If TP is in pips, convert to price using market price

• If SL is in pips, convert to price using entry price

• Never hallucinate prices — if a value cannot be clearly

  parsed, mark that field as null

• Add confidence score 0.0–1.0 to every output

  Below 0.7 = do not execute, flag for manual review



IGNORE THESE MESSAGES:

• "STANDBYYYY" / "ARE YOU READY" / "NEW SIGNAL COMING"

• "Join backup channel" / VIP promotions

• Market commentary without price levels

• Updates like "60+ PIPS RUNNING" / "SL hit no more trades"

• Any message without at least entry + SL or entry + TP



A SIGNAL IS VALID IF IT HAS:

• Direction (BUY/SELL/LONG/SHORT/BUY NOW/SELL NOW)

• At least one price (entry, zone, or market)

• At least SL or TP



OUTPUT JSON ONLY:

{

  "valid": true/false,

  "direction": "BUY" or "SELL",

  "entry": [price1, price2] or [price],

  "entry_type": "limit" or "zone" or "market",

  "sl": price or null,

  "tp": [tp1, tp2, tp3, tp4],

  "tp_type": "price" or "pips",

  "risk": "HIGH" or "NORMAL",

  "confidence": 0.0–1.0,

  "source_format": "clean/zone/emoji/compact/pip/

                    natural/mixed/forwarded",

  "raw": "original message"

}



If invalid:

{"valid": false, "reason": "specific reason here"}

```



---



**TRADE EXECUTION LOGIC**



When a valid signal is received with confidence above 0.7:



1. Place two entries (or one if signal has only one entry price) using the user's configured lot size

2. **Entry price logic:**

   - Current price matches signal entry (same integer, e.g. 4062.xxx matches 4062) → execute immediately at market

   - Current price is below signal entry (e.g. signal says 4062, price is 4059) → execute immediately at market price

   - Current price is above signal entry (e.g. signal says 4062, price is 4065) → place a limit order at signal entry price

3. **Skip signal entirely if:**

   - Current price has already reached or passed TP1 before entry is taken

   - Current price is already at or beyond the SL level

   - Signal confidence is below 0.7 — flag for manual review instead

4. Set SL and TP exactly as provided in the signal

5. At TP3 → close Position 1 fully, move SL of Position 2 to exact entry price (breakeven)

6. Execution must be near-instant to avoid slippage on fast gold moves



---



**TRADING MODES**



User selects one mode in Settings:



**Mode 1 — Quick 100 Pips:**

- Close ALL open positions when combined pips = 100

- Exit both entries immediately at that point



**Mode 2 — Hybrid:**

- Close 50% of positions at 100 pips (e.g. 3 of 6)

- Move SL to entry on remaining positions

- Let remaining positions run to TP4, TP5 and beyond



---



**POSITION SIZING**



- User sets: lot size (e.g. 0.01) + number of positions (e.g. 6)

- App splits positions across the two entries

- Example: 6 positions, 0.01 lot → 3 on Entry 1, 3 on Entry 2



---



**CHANNEL SELECTION**



- After Telegram is connected via Python/Telethon, app fetches and displays the full list of the user's joined channels and groups

- User selects one or multiple channels to monitor

- For each selected channel, display the last received message so user can verify it is working

- All selected channels saved in Supabase settings



---



**ACTIVE POSITION MANAGEMENT**



Real-time panel showing all open positions with:

- Instrument, direction, entry price, current P&L, SL, TP levels

- Buttons: Close Now, Edit SL, Edit TP

- All changes pushed immediately to MT5 via Python bridge



---



**DASHBOARD**



Header bar (always visible):

- Current market session (London / New York / Asian / Overlap)

- Market status: Open / Closed

- Saudi Arabia Standard Time (AST = UTC+3)



Dashboard tiles:

- Account Balance (live from MT5)

- Total Pips Today

- Winning Streak

- Win Rate %

- Average R:R

- Best Trade / Worst Trade

- Average Trade Duration

- Weekly Goal progress bar (editable in Settings)



---



**TRADE HISTORY SECTION**



Filterable by date range. Each trade record shows:

- Direction (Long / Short)

- Instrument / pair

- Entry price

- Close price

- Date and time opened

- Date and time closed

- Lot size

- P&L in pips and USD

- Close reason: TP1 / TP2 / TP3 / 100 Pips / Manual / SL



Export: PDF export with clean layout, branding, and applied filters



---



**ANALYTICS SECTION**



- Calendar heatmap (daily P&L, green/red color coded)

- Equity curve chart

- Win/loss ratio pie chart

- Best trading days and hours heatmap

- Monthly P&L summary



---



**SETTINGS SECTION**



- Telegram Account: connect, show status, channel selector with last message preview

- MetaTrader 5: account number, password, server — show Live/Demo badge and real-time balance once connected and actually functional

- Trading Mode: Mode 1 or Mode 2

- Lot Size and Position Count

- Weekly Goal in USD

- Instrument Focus: XAUUSD default, expandable later

- Notifications: enable/disable trade alerts



---



**SETUP GUIDE — Non-Technical Friendly**



The setup guide must feel like a friendly assistant walking the user through each step. No technical jargon. Big buttons. Progress bar showing Step X of 6. Each step only appears after the previous one is confirmed complete.



**STEP 1 — Welcome**

"Welcome to GoldCopier Pro! Let's get you set up in about 10 minutes. We will connect your Telegram and MetaTrader 5 so your trades run automatically while you sleep."

→ [Let's Begin →] button



**STEP 2 — Connect Telegram**

Show these sub-steps one at a time:

- "Open this link: my.telegram.org" → [Open Link] button

- "Log in with your Telegram phone number"

- "Click API Development Tools"

- "Create a new app — name it anything"

- "Copy your API ID and API Hash and paste below"

- Two input fields: API ID / API Hash

- [Connect Telegram] button

- Success: green checkmark "Telegram Connected ✓"

- Failure: plain English error message and what to do next

- Every sensitive field has a "Why do we need this?" tooltip



**STEP 3 — Select Signal Channel**

"Here are all your Telegram groups and channels. Select the one that sends gold signals."



- Scrollable list with channel names and icons

- Multi-select allowed

- Shows last message from each channel for verification

- [Confirm Channels →] button



**STEP 4 — Connect MetaTrader 5**

"Enter your MT5 account details. You can find these in your broker's welcome email."

- Fields: Account Number / Password / Server Name

- Server tooltip: "Looks like Exness-Real3 or ICMarkets-Live01 — check your broker email"

- [Connect MT5] button

- Success: shows live balance and account name

- Failure: plain English error and common fixes listed



**STEP 5 — Trading Preferences**

"How do you want to trade?"

- Lot Size input with helper text: "0.01 = micro lot, safe for beginners"

- Number of positions (default: 2)

- Trading Mode toggle with plain descriptions:

  - Mode 1: "Exit everything at 100 pips profit"

  - Mode 2: "Close half at 100 pips, let the rest run"

- Weekly profit goal in USD

- [Save & Continue →]



**STEP 6 — Test and Go Live**

"Let's make sure everything works before any real trades go through."

- [Send Test Signal] button

- Shows each pipeline step passing live:

  - ✓ Telegram received

  - ✓ Signal parsed

  - ✓ MT5 connected

  - ✓ Order simulated (not real money)

- All pass: "You're all set! GoldCopier Pro is now running 24/7." → [Go Live ✓]

- Any fail: highlights exactly which step failed with a simple plain-English fix



Progress bar visible at all times during setup. Back button on every step. Help/FAQ button always accessible.



---



**CRITICAL REQUIREMENTS**



- App runs 24/7 and never misses a signal

- Mobile-first UI, fully responsive on all screen sizes

- All Supabase calls must use correct SDK syntax — use `.upsert()` instead of `.insert().onConflict()` to avoid the Error 500 bug

- Python bridge script must be provided as a clean downloadable file, never corrupted by copy-paste

- MT5 balance must reflect live account data, not a placeholder

- Telegram monitor status must reflect actual connection, not hardcoded text

- Telegram and MT5 tiles in Settings must be fully functional, not decorative

- Remove all unused UI components and placeholder tiles

- Every single feature must be functional and tested end to end

- Signal confidence below 0.7 must never auto-execute — show manual review flag instead

- Double check every connection, every Supabase query, every API call, every PDF export, every chart before considering anything complete



---

This project was built with [Lovable](https://lovable.dev).

**Live app**: https://tgxau.lovable.app

## Build with Lovable

Continue developing this project in the [Lovable editor](https://lovable.dev/projects/5a2249d6-aa48-4674-93e5-4f3b2cdc9de1).

- **Ship faster**: describe what you want to build and Lovable handles the code.
- **Stay in sync**: every change made in Lovable is committed straight to this repository.
- **Full ownership**: this code is yours. Push to `main` on GitHub and your changes sync back into Lovable, ready for your next prompt.

## Development

Prefer working locally? You need Node.js and npm — [install with nvm](https://github.com/nvm-sh/nvm#installing-and-updating).

```sh
git clone <this-repository-url>
cd <repository-name>
npm i
npm run dev
```
