## Allowence — Product Specification (MVP)

### 1) Vision
A simple app to set self-allowances for habits or spending. Users define how much they can consume per interval, then log usage with one tap. The app enforces hard caps, makes logging effortless, and shows progress and savings (money or units avoided).

### 2) Platforms & Privacy
- iOS and Android (cross‑platform).
- Local-only by default. No account required.
- Optional privacy mode: biometric lock (Face ID/Touch ID) and in-app PIN. Data remains on-device.

### 3) Core Concepts
- **Allowance**: A budget of units for a chosen interval (e.g., 4 cigarettes/day, 4 drinks/week, $200/month).
- **Unit**: Free-form label (e.g., cigarettes, drinks, dollars, minutes). Units can be monetary or non‑monetary.
- **Cost per unit**: If unit is not money, user can set a monetary cost per unit to compute savings.
- **Type**: Reset or Growth.
  - Reset: balance resets to amount each interval.
  - Growth: balance increases by amount each interval up to a maximum saving amount.
- **Hard cap**: Users cannot log beyond zero balance.
- **Non-retroactive edits**: Edits to allowance settings affect only future intervals.

### 4) MVP Features
- **Allowances list**: Card list showing name, current balance, progress ring/bar, remaining vs total, and quick actions.
- **Quick logging**:
  - One‑tap presets (+1, +0.5, +custom). Long‑press to edit presets.
  - Optional note field on custom log.
  - Enforces hard cap (no negative balance). Show inline error if exceeding.
- **Progress visuals**:
  - Per‑allowance progress bar with color thresholds (e.g., <50% green, 50–80% amber, >80% red).
  - Remaining counter (e.g., 2/4 today; $60/$200 this month).
- **Custom intervals**:
  - Built‑in: Day, Week, Month.
  - Custom: Every N days/weeks/months (e.g., every 3 days, every 2 weeks).
- **Shortcuts (App Quick Actions)**:
  - iOS: UIApplicationShortcutItems for “Log +1” on top 1–3 allowances.
  - Android: Static/Dynamic App Shortcuts for the same.
- **Privacy**:
  - Local-only storage, opt‑in biometric/PIN lock, quick lock on background.

Widgets are important but can ship post‑MVP (see Roadmap).

### 5) Data Model (local)
Allowance
- id (string/UUID)
- name (string)
- unitLabel (string) — e.g., "cigarettes", "drinks", "$" or "minutes"
- isMonetaryUnit (boolean) — true if unitLabel represents currency
- costPerUnit (number | null) — monetary value per unit for savings calc (ignored if isMonetaryUnit)
- amountPerInterval (number)
- intervalType ('day' | 'week' | 'month' | 'custom')
- customIntervalValue (number | null) — N for every N units
- customIntervalUnit ('days' | 'weeks' | 'months' | null)
- type ('reset' | 'growth')
- maxSavingAmount (number | null) — required if type = 'growth'
- currentBalance (number)
- createdAt (timestamp)
- archivedAt (timestamp | null)

Log
- id (string/UUID)
- allowanceId (string)
- delta (number) — negative when consuming (e.g., -1, -0.5, -5)
- note (string | null)
- createdAt (timestamp)
- isEstimate (boolean) — true when user logs an estimation for today

System
- intervalAnchors: For each allowance, anchor intervals to device local time.

### 6) Core Logic
Interval tick (when an allowance interval elapses):
- If type = 'reset':
  - currentBalance := amountPerInterval
- If type = 'growth':
  - currentBalance := min(currentBalance + amountPerInterval, maxSavingAmount)

Logging consumption:
- Proposed logging amounts are negative deltas. Reject if `currentBalance + delta < 0` (hard cap).
- On accept: `currentBalance := currentBalance + delta`

Savings and “units not consumed”:
- For a given interval window: let `consumed = sum(|delta| where delta < 0 and not isEstimate)`
- Units remaining (so far): `remaining = max(0, effectiveAllowance - consumed)`
- Units not consumed (vs allowance): `sparedUnits = remaining`
- Monetary savings for non‑monetary units (if costPerUnit is set):
  - `savedMoney = sparedUnits * costPerUnit`
- For monetary units (e.g., `$`): treat `costPerUnit = 1` and `savedMoney = remaining`

Estimates (today):
- Users can log estimated consumption for the current day with `isEstimate = true`.
- Estimates do not reduce balance; they only affect projections/graphs.
- Daily projection for today: `projectedConsumedToday = actualConsumedToday + estimatedUnitsToday`

Notes on edits:
- Edits to allowance settings affect only future intervals. Past logs and closed intervals remain unchanged.

### 7) UI
Screens
- Home (Allowances List): cards with name, progress bar, remaining/total, quick +1/+0.5/+ custom.
- Log Sheet: preset buttons, keypad for custom amount, optional note, estimation toggle.
- Create/Edit Allowance: name, unit, interval (incl. custom every N), amount, type (reset/growth), max saving (if growth), cost per unit (optional), monetary vs non‑monetary toggle.
- Settings: privacy lock (enable/disable biometric/PIN), data export (CSV later, out of scope MVP).

Graphs
- Per‑allowance: daily/weekly/monthly bar chart of consumption, with allowance line.
- Savings overlay: show spared units and computed saved money (if costPerUnit set).
- Today: show projected bar including estimate separate from actual.

Empty states & errors
- Clear call to action to create first allowance.
- Inline errors for exceeding hard cap; soft nudge to adjust custom amount.

### 8) Technical Approach (proposal)
- Framework: React Native (Expo)
  - Quick Actions: `react-native-quick-actions` or native modules per platform.
  - Charts: `victory-native` or `react-native-svg-charts`.
  - Local DB: SQLite (Expo SQLite) with a light ORM (e.g., Drizzle ORM RN or direct SQL).
  - Secure storage for lock credentials: Keychain (iOS), Keystore (Android) via `expo-secure-store`.
- Time handling: anchor intervals to device local time; recompute next interval boundaries on DST change.
- State management: Redux Toolkit or Zustand; background task to process interval rollovers on app launch/resume.

### 9) Edge Cases
- Interval boundaries: weekly starts on user’s locale week start (Mon/Sun) or per‑allowance choice.
- Month shortness: if custom intervals in months, roll to same day or last day when shorter.
- Floating‑point: store amounts in integers when sensible (e.g., basis points) or use decimal library.
- Presets per allowance: default [+1, +0.5]; customizable.

### 10) Roadmap (post‑MVP)
- Home/lock‑screen widgets (iOS/Android) for quick logging and glanceable progress.
- CSV export and historical analytics.
- Cloud sync with E2E encryption.
- Advanced carryover options (hybrid), goals/streaks.
- Reminders (user opted not now).

### 11) Acceptance Criteria (MVP)
- Create, edit, archive allowances with specified fields and interval types.
- Logging via quick actions and custom amounts; hard cap enforced.
- Progress ring/bar updates correctly; color thresholds reflect usage.
- Custom intervals (every N days/weeks/months) function with correct rollover.
- Estimates can be added for today and appear in graphs without reducing balance.
- Savings displayed when `costPerUnit` or monetary unit is provided.
- App Quick Actions available to log +1 for top allowances on both platforms.
- Privacy lock works; data remains local.
