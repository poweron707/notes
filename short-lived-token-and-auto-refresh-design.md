# Short-Lived Token & Auto-Refresh Design (Unified with Login Session)

**Date:** 2026-03-28 (Updated)

---

## Overview

Design for a short-lived action token (15 min TTL) that operates **within** the existing login session. The action token auto-refreshes at the last 1 minute of validity. If the login session expires, all action tokens are invalidated. This ensures two-layer security: identity (login session) + activity (action token).

---

## 1. Architecture

```
┌──────────────────────────────────────────────────────┐
│  Layer 1: Login Session (NextAuth)                   │
│  • Long-lived (7 days via refresh token)             │
│  • Handles: who you are (identity + role + groups)   │
│                                                      │
│  Layer 2: Short-Lived Action Token                   │
│  • 15 min TTL, auto-refresh at last 1 min            │
│  • Handles: what you're doing right now              │
│  • DEPENDS on Layer 1 — can't exist without it       │
│  • Inherits userId, role, groups from session        │
└──────────────────────────────────────────────────────┘
```

The short-lived token is a **child** of the login session. If the login session dies, all short-lived tokens die with it.

---

## 2. Token Generator — Session-Aware (`lib/short-token.ts`)

```ts
import crypto from "crypto";
import { Session } from "next-auth";

const SECRET = process.env.TOKEN_SECRET!;
const TOKEN_TTL = 15 * 60; // 15 minutes
const REFRESH_THRESHOLD = 60; // last 1 min

interface TokenPayload {
  // inherited from login session
  userId: string;
  role: string;
  groups: string[];
  sessionId: string;   // ties to login session

  // token-specific
  purpose: string;
  iat: number;
  exp: number;
}

export function createToken(session: Session, purpose: string): string {
  const now = Math.floor(Date.now() / 1000);
  const payload: TokenPayload = {
    userId: session.user.id,
    role: session.user.role,
    groups: session.user.groups,
    sessionId: session.user.sessionId,

    purpose,
    iat: now,
    exp: now + TOKEN_TTL,
  };

  const data = Buffer.from(JSON.stringify(payload)).toString("base64url");
  const sig = crypto
    .createHmac("sha256", SECRET)
    .update(data)
    .digest("base64url")
    .slice(0, 22);

  return `${data}.${sig}`;
}

export function verifyToken(token: string): TokenPayload | null {
  const [data, sig] = token.split(".");
  if (!data || !sig) return null;

  const expected = crypto
    .createHmac("sha256", SECRET)
    .update(data)
    .digest("base64url")
    .slice(0, 22);

  if (!crypto.timingSafeEqual(Buffer.from(sig), Buffer.from(expected))) {
    return null;
  }

  try {
    const payload: TokenPayload = JSON.parse(
      Buffer.from(data, "base64url").toString()
    );

    const now = Math.floor(Date.now() / 1000);
    if (now > payload.exp) return null;

    return payload;
  } catch {
    return null;
  }
}

export function needsRefresh(token: string): boolean {
  const [data] = token.split(".");
  if (!data) return true;

  try {
    const payload: TokenPayload = JSON.parse(
      Buffer.from(data, "base64url").toString()
    );
    const now = Math.floor(Date.now() / 1000);
    const remaining = payload.exp - now;
    return remaining <= REFRESH_THRESHOLD && remaining > 0;
  } catch {
    return true;
  }
}
```

---

## 3. Unified Verification Middleware (`lib/unified-auth.ts`)

Verifies **both** layers in one call:

```ts
import { auth } from "@/lib/auth";
import { verifyToken } from "@/lib/short-token";
import { NextRequest } from "next/server";

interface AuthResult {
  valid: boolean;
  error?: "NO_SESSION" | "SESSION_EXPIRED" | "TOKEN_INVALID" | "TOKEN_EXPIRED" | "SESSION_MISMATCH";
  userId?: string;
  role?: string;
  groups?: string[];
  purpose?: string;
}

export async function verifyUnified(req: NextRequest): Promise<AuthResult> {
  // Layer 1: check login session
  const session = await auth();

  if (!session?.user) {
    return { valid: false, error: "NO_SESSION" };
  }

  // Layer 2: check short-lived token (if provided)
  const actionToken = req.headers.get("x-action-token");

  if (!actionToken) {
    // no action token — return session-only auth
    return {
      valid: true,
      userId: session.user.id,
      role: session.user.role,
      groups: session.user.groups,
    };
  }

  // verify action token
  const tokenPayload = verifyToken(actionToken);
  if (!tokenPayload) {
    return { valid: false, error: "TOKEN_EXPIRED" };
  }

  // critical: token must belong to this session
  if (tokenPayload.userId !== session.user.id) {
    return { valid: false, error: "SESSION_MISMATCH" };
  }

  // verify the login session hasn't been invalidated
  if (tokenPayload.sessionId !== session.user.sessionId) {
    return { valid: false, error: "SESSION_MISMATCH" };
  }

  return {
    valid: true,
    userId: tokenPayload.userId,
    role: tokenPayload.role,
    groups: tokenPayload.groups,
    purpose: tokenPayload.purpose,
  };
}
```

---

## 4. Server API Routes

### Issue Token — Requires Active Login Session

```ts
// app/api/token/issue/route.ts
import { NextRequest, NextResponse } from "next/server";
import { auth } from "@/lib/auth";
import { createToken } from "@/lib/short-token";

export async function POST(req: NextRequest) {
  const session = await auth();

  if (!session?.user) {
    return NextResponse.json(
      { error: "LOGIN_REQUIRED", redirect: "/auth/signin" },
      { status: 401 }
    );
  }

  const { purpose } = await req.json();
  const token = createToken(session, purpose);

  return NextResponse.json({ token, expiresIn: 900 });
}
```

### Refresh Token — Re-validates Login Session

```ts
// app/api/token/refresh/route.ts
import { NextRequest, NextResponse } from "next/server";
import { auth } from "@/lib/auth";
import { verifyToken, createToken } from "@/lib/short-token";

export async function POST(req: NextRequest) {
  // must still be logged in
  const session = await auth();
  if (!session?.user) {
    return NextResponse.json(
      { error: "SESSION_EXPIRED", redirect: "/auth/signin" },
      { status: 401 }
    );
  }

  const { token } = await req.json();
  const payload = verifyToken(token);

  if (!payload) {
    return NextResponse.json(
      { error: "TOKEN_EXPIRED" },
      { status: 401 }
    );
  }

  // token must belong to current session
  if (payload.userId !== session.user.id) {
    return NextResponse.json(
      { error: "SESSION_MISMATCH" },
      { status: 403 }
    );
  }

  // issue fresh token with CURRENT session data (picks up role/group changes)
  const newToken = createToken(session, payload.purpose);

  return NextResponse.json({ token: newToken, expiresIn: 900 });
}
```

### Protected Campaign API — Both Layers

```ts
// app/api/campaigns/[campaignId]/route.ts
import { NextRequest, NextResponse } from "next/server";
import { verifyUnified } from "@/lib/unified-auth";
import { canAccessCampaign } from "@/lib/campaign-access";

export async function GET(
  req: NextRequest,
  { params }: { params: { campaignId: string } }
) {
  const authResult = await verifyUnified(req);

  if (!authResult.valid) {
    const status = authResult.error === "NO_SESSION" ? 401 : 403;
    return NextResponse.json(
      { error: authResult.error, redirect: status === 401 ? "/auth/signin" : undefined },
      { status }
    );
  }

  const access = await canAccessCampaign(authResult.userId!, params.campaignId);
  if (!access.allowed) {
    return NextResponse.json(
      { error: access.reason },
      { status: 403 }
    );
  }

  const campaign = await prisma.campaign.findUnique({
    where: { id: params.campaignId },
  });

  return NextResponse.json({ campaign });
}
```

---

## 5. Client-Side — Handles Both Session Types (`lib/token-client.ts`)

```ts
const REFRESH_CHECK_INTERVAL = 10_000;

let currentToken: string | null = null;
let refreshTimer: ReturnType<typeof setInterval> | null = null;

export async function startTokenSession(purpose: string): Promise<string> {
  const res = await fetch("/api/token/issue", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    credentials: "include", // send login session cookie
    body: JSON.stringify({ purpose }),
  });

  if (res.status === 401) {
    // login session expired
    window.location.href = "/auth/signin";
    throw new Error("Login required");
  }

  if (!res.ok) throw new Error("Failed to issue token");

  const { token } = await res.json();
  currentToken = token;
  startRefreshMonitor();
  return token;
}

export function getToken(): string | null {
  return currentToken;
}

function startRefreshMonitor() {
  if (refreshTimer) clearInterval(refreshTimer);

  refreshTimer = setInterval(async () => {
    if (!currentToken) return;
    if (!isExpiringSoon(currentToken)) return;

    try {
      const res = await fetch("/api/token/refresh", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        credentials: "include",
        body: JSON.stringify({ token: currentToken }),
      });

      if (res.ok) {
        const { token } = await res.json();
        currentToken = token;
      } else {
        const data = await res.json();
        if (data.error === "SESSION_EXPIRED") {
          window.location.href = "/auth/signin";
        }
        handleExpired();
      }
    } catch {
      handleExpired();
    }
  }, REFRESH_CHECK_INTERVAL);
}

// authenticated fetch that sends BOTH session cookie + action token
export async function securedFetch(
  url: string,
  options?: RequestInit
): Promise<Response> {
  if (!currentToken) throw new Error("No active token session");

  const res = await fetch(url, {
    ...options,
    credentials: "include",
    headers: {
      ...options?.headers,
      "x-action-token": currentToken,
    },
  });

  if (res.status === 401) {
    const data = await res.json();
    if (data.redirect) {
      window.location.href = data.redirect;
    }
    handleExpired();
    throw new Error(data.error);
  }

  return res;
}

function isExpiringSoon(token: string): boolean {
  try {
    const [data] = token.split(".");
    const payload = JSON.parse(atob(data.replace(/-/g, "+").replace(/_/g, "/")));
    const now = Math.floor(Date.now() / 1000);
    return payload.exp - now <= 60 && payload.exp - now > 0;
  } catch {
    return true;
  }
}

function handleExpired() {
  currentToken = null;
  if (refreshTimer) clearInterval(refreshTimer);
  window.dispatchEvent(new CustomEvent("token-expired"));
}

export function endTokenSession() {
  currentToken = null;
  if (refreshTimer) clearInterval(refreshTimer);
}
```

---

## 6. React Hook (`hooks/useSecureSession.ts`)

```tsx
"use client";

import { useState, useEffect } from "react";
import { useSession } from "next-auth/react";
import { startTokenSession, getToken, endTokenSession } from "@/lib/token-client";

export function useSecureSession(purpose: string) {
  const { data: session, status } = useSession();
  const [token, setToken] = useState<string | null>(null);
  const [expired, setExpired] = useState(false);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    // wait for login session
    if (status === "loading") return;

    if (status === "unauthenticated") {
      setExpired(true);
      setLoading(false);
      return;
    }

    // login session active — start action token
    startTokenSession(purpose)
      .then((t) => {
        setToken(t);
        setLoading(false);
      })
      .catch(() => {
        setExpired(true);
        setLoading(false);
      });

    const handler = () => {
      setToken(null);
      setExpired(true);
    };
    window.addEventListener("token-expired", handler);

    const refreshCheck = setInterval(() => {
      const current = getToken();
      if (current && current !== token) setToken(current);
    }, 5000);

    return () => {
      endTokenSession();
      window.removeEventListener("token-expired", handler);
      clearInterval(refreshCheck);
    };
  }, [purpose, status]);

  return {
    session,          // login session data
    token,            // action token
    expired,          // either session or token expired
    loading,
    isAuthenticated: status === "authenticated" && !!token,
  };
}
```

---

## 7. Usage Example

```tsx
"use client";

import { useSecureSession } from "@/hooks/useSecureSession";
import { securedFetch } from "@/lib/token-client";

export default function CampaignPage({ campaignId }: { campaignId: string }) {
  const { session, expired, loading, isAuthenticated } = useSecureSession("campaign-view");
  const [campaign, setCampaign] = useState(null);

  useEffect(() => {
    if (!isAuthenticated) return;

    securedFetch(`/api/campaigns/${campaignId}`)
      .then((res) => res.json())
      .then((data) => setCampaign(data.campaign));
  }, [isAuthenticated, campaignId]);

  if (loading) return <div>Loading...</div>;

  if (expired) {
    return (
      <div className="text-center py-20">
        <h2 className="text-xl font-bold">⏱️ Session Expired</h2>
        <p className="mt-2 text-gray-500">Please sign in again to continue.</p>
        <a href="/auth/signin" className="mt-4 inline-block px-6 py-2 bg-blue-600 text-white rounded-lg">
          Sign In
        </a>
      </div>
    );
  }

  return (
    <div>
      <p className="text-sm text-gray-400">
        Signed in as {session?.user?.name} ✅
      </p>
      {/* campaign content */}
    </div>
  );
}
```

---

## 8. Session Lifecycle Flow

```
Login session expires?
  → Token refresh fails (401 SESSION_EXPIRED)
  → Client redirects to /auth/signin
  → All action tokens invalidated

Action token expires + user idle?
  → "Session Expired" shown
  → Login session still alive
  → User clicks "Continue" → new action token issued

Action token about to expire + user active?
  → Auto-refresh at last 1 min
  → Refresh re-validates login session
  → Picks up any role/group changes
  → Seamless ✅

Admin revokes user access mid-session?
  → Next refresh: server checks session → still valid
  → Next API call: canAccessCampaign → 403
  → Immediate effect ✅
```

---

## 9. Environment Variables

```env
TOKEN_SECRET=your-random-64-char-string
```

---

## 10. Security Summary

| Scenario | What Happens |
|---|---|
| Login session valid + token valid | Full access ✅ |
| Login session valid + token expired | Prompt to continue (re-issue token) |
| Login session expired + token valid | Token refresh fails → redirect to login |
| Login session expired + token expired | Redirect to login |
| Token stolen, used from another session | `SESSION_MISMATCH` → 403 |
| Role changed mid-session | Picked up on next token refresh |
| User removed from campaign | Next API call → `canAccessCampaign` → 403 |

---

## 11. Unified Auth Decision Flow Diagram

```
┌─────────────────────────────────────┐
│         Incoming Request            │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│   Has login session cookie?         │
│                                     │
│   NO ──► 401 "NO_SESSION"           │
│          redirect → /auth/signin    │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   Is login session valid?           │
│   (not expired, not revoked)        │
│                                     │
│   NO ──► 401 "SESSION_EXPIRED"      │
│          redirect → /auth/signin    │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   Has x-action-token header?        │
│                                     │
│   NO ──► Return session-only auth   │
│          (for non-token routes)     │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   Is token signature valid?         │
│   (HMAC verification)               │
│                                     │
│   NO ──► 403 "TOKEN_INVALID"        │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   Is token expired?                 │
│   (now > payload.exp)               │
│                                     │
│   YES ──► 401 "TOKEN_EXPIRED"       │
└──────────────┬──────────────────────┘
               ▼ NO
┌─────────────────────────────────────┐
│   Does token.userId match           │
│   session.userId?                   │
│                                     │
│   NO ──► 403 "SESSION_MISMATCH"     │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   Does token.sessionId match        │
│   session.sessionId?                │
│                                     │
│   NO ──► 403 "SESSION_MISMATCH"     │
│          (session was re-created)   │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   Route has campaign access check?  │
│                                     │
│   NO ──► ✅ AUTHORIZED             │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   canAccessCampaign(userId, id)?    │
│                                     │
│   Check: is user owner?             │
│   NO ──▼                            │
│   Check: is user a member?          │
│   NO ──▼                            │
│   Check: is campaign PUBLIC?        │
│   NO ──▼                            │
│   Check: is campaign UNLISTED?      │
│   NO ──► 403 "NO_ACCESS"            │
│                                     │
│   ANY YES ──► ✅ AUTHORIZED        │
└─────────────────────────────────────┘
```

---

## 12. Token Refresh Decision Flow Diagram

```
┌─────────────────────────────────────┐
│   Refresh timer fires (every 10s)   │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│   Has current token?                │
│                                     │
│   NO ──► Skip (no active session)   │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   Token remaining time <= 60s?      │
│                                     │
│   NO ──► Skip (not yet)             │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   Call /api/token/refresh           │
│   (sends session cookie + token)    │
└──────────────┬──────────────────────┘
               ▼
┌─────────────────────────────────────┐
│   Is login session still valid?     │
│                                     │
│   NO ──► "SESSION_EXPIRED"          │
│          redirect → /auth/signin    │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   Is old token still valid?         │
│                                     │
│   NO ──► "TOKEN_EXPIRED"            │
│          dispatch token-expired     │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   Does token.userId match session?  │
│                                     │
│   NO ──► "SESSION_MISMATCH"         │
│          dispatch token-expired     │
└──────────────┬──────────────────────┘
               ▼ YES
┌─────────────────────────────────────┐
│   Issue new token                   │
│   (inherits CURRENT session data)   │
│   (fresh 15 min TTL)                │
│                                     │
│   ✅ Token refreshed seamlessly    │  
└─────────────────────────────────────┘
```

---

## 13. Session Expiry Scenarios

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  Scenario 1: Active User                                    │
│  ─────────────────────────                                  │
│  0:00    Token issued ──────────────────────────────┐       │
│  14:00   Remaining <= 60s                           │       │
│          ├─ Login session valid? ── YES             │       │
│          ├─ Old token valid? ── YES                 │       │
│          └─ ✅ New token issued  ───────────────┐   │       │
│  28:00   Remaining <= 60s                       │   │       │
│          ├─ Login session valid? ── YES         │   │       │
│          └─ ✅ New token issued ───────────┐    │   │       │
│          ... continues indefinitely         │   │   │       │
│                                                             │
│  Scenario 2: Idle User (missed refresh)                     │
│  ──────────────────────────────────────                     │
│  0:00    Token issued ──────────────────────────────┐       │
│  14:00   Remaining <= 60s                           │       │
│          ├─ Browser tab inactive / JS paused        │       │
│  15:01   Token EXPIRED                              │       │
│          ├─ Next action triggers fetch              │       │
│          ├─ Server returns 401 TOKEN_EXPIRED        │       │
│          ├─ Login session still valid?              │       │
│          │  ├─ YES → Show "Continue" button         │       │
│          │  │        Click → new token issued ✅    │       │
│          │  └─ NO  → Redirect to /auth/signin       │       │
│                                                             │
│  Scenario 3: Login Session Expires                          │
│  ─────────────────────────────────                          │
│  0:00    Token issued ──────────────────────────────┐       │
│  14:00   Remaining <= 60s                           │       │
│          ├─ Call /api/token/refresh                 │       │
│          ├─ Login session expired                   │       │
│          ├─ Server returns 401 SESSION_EXPIRED      │       │
│          └─ Redirect to /auth/signin                │       │
│                                                             │
│  Scenario 4: Token Stolen                                   │
│  ────────────────────────                                   │
│  Attacker uses token from different browser/session         │
│          ├─ token.userId ≠ session.userId           │       │
│          │  OR                                      │       │
│          ├─ token.sessionId ≠ session.sessionId     │       │
│          └─ 403 SESSION_MISMATCH 🚫                 │       │
│                                                             │
│  Scenario 5: Role/Group Changed Mid-Session                 │
│  ──────────────────────────────────────────                 │
│  Admin changes user role from REGISTERED → PREMIUM          │
│  14:00   Token refresh triggered                    │       │
│          ├─ Server reads CURRENT session            │       │
│          ├─ New token gets role: PREMIUM            │       │
│          └─ ✅ User sees premium content on refresh │       │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```
