# 03 — Auth Screen

> **Route:** `/auth` | **File:** `lib/screens/auth_screen.dart` | **Auth:** None

---

## Purpose

Handles user authentication (Google Sign-In and Phone OTP) and profile display. Shows a sign-in view for unauthenticated users and a profile view for authenticated users.

---

## Layout — Sign-In Mode

```
┌─────────────────────────────────────┐
│ (ambient violet glow, top-right)     │
│                                      │
│  ← Back                              │
│                                      │
│      Welcome, Traveller              │  ← 26px, w800, gold, letterSpacing 1
│                                      │
│   Sign in to enter your realms and   │  ← 14px, ash
│       continue your story.           │
│                                      │
│  ┌──────────────────────────────┐    │
│  │  Google    │    Phone        │    │  ← Tab selector (44px height)
│  └──────────────────────────────┘    │
│                                      │
│  ┌─ Google Tab ──────────────────┐   │
│  │  ┌────┐                       │   │
│  │  │ G  │  Sign in with Google  │   │  ← Google icon card
│  │  └────┘  Fast and secure…     │   │
│  │                               │   │
│  │  [  Continue with Google   ]  │   │  ← 52px height, gold button
│  │                               │   │
│  │  Google sign-in not config…   │   │  ← Fallback message
│  └───────────────────────────────┘   │
│                                      │
│  OR                                  │
│                                      │
│  ┌─ Phone Tab ───────────────────┐   │
│  │  [📱 Phone number field    ]  │   │  ← +1 555 000 0000 hint
│  │  [  Send Verification Code ]  │   │  ← 50px gold button
│  │                               │   │
│  │  (After code sent:)           │   │
│  │  [  — — — — — —  code   ]    │   │  ← Centered, 22px, letterSpacing 8
│  │  [   Enter the Realm      ]  │   │  ← 50px, violet button
│  └───────────────────────────────┘   │
│                                      │
│  ┌ ⚠ Error message ─────────────┐   │  ← crimson bg @10%, crimson border
│  └───────────────────────────────┘   │
│                                      │
│  ┌ ✅ Success message ──────────┐   │  ← verdant bg @10%, verdant border
│  └───────────────────────────────┘   │
└─────────────────────────────────────┘
```

---

## Layout — Profile Mode (Authenticated)

```
┌─────────────────────────────────────┐
│  ← Back                              │
│                                      │
│          ┌──────────┐                │
│          │   A      │                │  ← 80px circle, violet gradient, goldDim border
│          │ (avatar) │                │
│          └──────────┘                │
│                                      │
│          Username                    │  ← 22px, w700, parchment
│          user@email.com              │  ← ash
│          +1 555 000 0000             │  ← ash (optional)
│                                      │
│        ╭──────────╮                  │
│        │ PREMIUM  │                  │  ← Tier badge (violet/gold/ash)
│        ╰──────────╯                  │
│                                      │
│  ┌─ Profile Card ────────────────┐   │
│  │  🧭 Browse Worlds          →  │   │
│  │  ─────────────────────────── │   │
│  │  📖 Your Realms            →  │   │
│  └───────────────────────────────┘   │
│                                      │
│  [  🔓 Sign Out              ]      │  ← OutlinedButton, crimson
└─────────────────────────────────────┘
```

### Tier Badge Colors

| Tier | Background | Border | Text |
|---|---|---|---|
| `free` | ash @15% | ash @40% | ash |
| `premium` | gold @15% | gold @40% | gold |
| `creator` | violet @15% | violet @40% | violet |

---

## Tab Selector

```
Height: 44px
Background: void2
Border Radius: 12px
Border: goldDim @30%

Active Tab:
  - Background: void4
  - Border: goldDim @50%
  - Border Radius: 10px
  - Text: gold, w600, 13px

Inactive Tab:
  - Text: ash
```

---

## Auth Methods

### Google Sign-In
1. `GoogleAuthService.init()` with `GOOGLE_WEB_CLIENT_ID` from `.env`
2. `GoogleAuthService.signIn()` → get `idToken`
3. `AuthService.loginWithGoogle(idToken)` → `User`
4. Navigate to `/` (home)

### Phone OTP
1. Normalize phone number (strip non-digits, prefix `+`)
2. `AuthService.sendOtp(phone)` → shows success banner
3. User enters 6-digit code (centered, large letterSpacing)
4. `AuthService.verifyOtp(phone, code)` → `User`
5. Navigate to `/` (home)

---

## State

- Local `StatefulWidget` state
- TabController with 2 tabs
- TextEditingControllers: `_phoneController`, `_otpController`
- State variables: `_currentUser`, `_googleReady`, `_isLoading`, `_codeSent`, `_errorMessage`, `_successMessage`
