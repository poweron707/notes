# JWT Token & Campaign Access Control Design

**Date:** 2026-03-24

---

## Overview

Design for handling JWT token expiry race conditions during request submission, and the recommended approach for campaign-level access control using session tokens instead of per-campaign tokens.

---

## Part 1: JWT Token Expiry Race Condition

### The Problem

A user submits a request with a valid token, but by the time the server verifies it, the token has just expired during the network round-trip.

---

### Solution 1: Grace Period (Simplest)

Add a buffer window when verifying:

```ts
import jwt from "jsonwebtoken";

const GRACE_PERIOD_SECONDS = 30;

export function verifyToken(token: string): jwt.JwtPayload | null {
  try {
    return jwt.verify(token, process.env.JWT_SECRET!, {
      clockTolerance: GRACE_PERIOD_SECONDS, // built-in support
    }) as jwt.JwtPayload;
  } catch (err) {
    return null;
  }
}
```

`clockTolerance` is a native `jsonwebtoken` option — allows tokens up to N seconds past their `exp` to still pass verification.

---

### Solution 2: Pre-Submit Token Check + Auto-Refresh (Best UX)

Check the token before submitting. If it's about to expire, refresh it first.

#### Client-Side Token Manager

```ts
// lib/token-manager.ts

const REFRESH_THRESHOLD = 60; // refresh if < 60s remaining

function getTokenExpiry(token: string): number {
  const payload = JSON.parse(atob(token.split(".")[1]));
  return payload.exp;
}

function isTokenExpiringSoon(token: string): boolean {
  const exp = getTokenExpiry(token);
  const now = Math.floor(Date.now() / 1000);
  return exp - now < REFRESH_THRESHOLD;
}

async function refreshToken(): Promise<string> {
  const res = await fetch("/api/auth/refresh", {
    method: "POST",
    credentials: "include",
  });

  if (!res.ok) throw new Error("Refresh failed");
  const { accessToken } = await res.json();
  return accessToken;
}

export async function authenticatedFetch(
  url: string,
  options?: RequestInit
): Promise<Response> {
  let token = getStoredToken();

  // auto-refresh if expiring soon
  if (!token || isTokenExpiringSoon(token)) {
    try {
      token = await refreshToken();
      storeToken(token);
    } catch {
      window.location.href = "/auth/signin";
      throw new Error("Session expired");
    }
  }

  const res = await fetch(url, {
    ...options,
    headers: {
      ...options?.headers,
      Authorization: `Bearer ${token}`,
    },
  });

  // if still got 401 (edge case), try one more refresh
  if (res.status === 401) {
    try {
      token = await refreshToken();
      storeToken(token);
      return fetch(url, {
        ...options,
        headers: {
          ...options?.headers,
          Authorization: `Bearer ${token}`,
        },
      });
    } catch {
      window.location.href = "/auth/signin";
      throw new Error("Session expired");
    }
  }

  return res;
}

function getStoredToken(): string | null {
  return sessionStorage.getItem("accessToken");
}

function storeToken(token: string) {
  sessionStorage.setItem("accessToken", token);
}
```

#### Server-Side Refresh Endpoint

```ts
// app/api/auth/refresh/route.ts
import { NextRequest, NextResponse } from "next/server";
import jwt from "jsonwebtoken";

const ACCESS_SECRET = process.env.JWT_SECRET!;
const REFRESH_SECRET = process.env.JWT_REFRESH_SECRET!;

export async function POST(req: NextRequest) {
  const refreshToken = req.cookies.get("refreshToken")?.value;

  if (!refreshToken) {
    return NextResponse.json({ error: "No refresh token" }, { status: 401 });
  }

  try {
    const payload = jwt.verify(refreshToken, REFRESH_SECRET) as jwt.JwtPayload;

    const accessToken = jwt.sign(
      { userId: payload.userId, role: payload.role },
      ACCESS_SECRET,
      { expiresIn: "15m" }
    );

    return NextResponse.json({ accessToken });
  } catch {
    return NextResponse.json({ error: "Refresh expired" }, { status: 401 });
  }
}
```

---

### Solution 3: Dual Token Strategy (Recommended)

```
┌─────────────────────────────────────────────┐
│  Access Token (short-lived: 15 min)         │
│  • Stored in memory/sessionStorage          │
│  • Sent as Bearer header                    │
│  • Used for API requests                    │
│                                             │
│  Refresh Token (long-lived: 7 days)         │
│  • Stored in httpOnly secure cookie         │
│  • Only sent to /api/auth/refresh           │
│  • Used to get new access tokens            │
└─────────────────────────────────────────────┘
```

#### Login — Issue Both Tokens

```ts
// app/api/auth/login/route.ts
import { NextRequest, NextResponse } from "next/server";
import jwt from "jsonwebtoken";

export async function POST(req: NextRequest) {
  const { email, password } = await req.json();

  const user = await validateUser(email, password);
  if (!user) {
    return NextResponse.json({ error: "Invalid credentials" }, { status: 401 });
  }

  const accessToken = jwt.sign(
    { userId: user.id, role: user.role },
    process.env.JWT_SECRET!,
    { expiresIn: "15m" }
  );

  const refreshToken = jwt.sign(
    { userId: user.id, role: user.role },
    process.env.JWT_REFRESH_SECRET!,
    { expiresIn: "7d" }
  );

  const response = NextResponse.json({ accessToken });

  response.cookies.set("refreshToken", refreshToken, {
    httpOnly: true,
    secure: true,
    sameSite: "strict",
    path: "/api/auth/refresh",
    maxAge: 7 * 24 * 60 * 60,
  });

  return response;
}
```

#### Server-Side Verification with Grace Period

```ts
// lib/auth-verify.ts
import jwt from "jsonwebtoken";

export function verifyAccessToken(token: string) {
  return jwt.verify(token, process.env.JWT_SECRET!, {
    clockTolerance: 30, // 30s grace for race conditions
  }) as jwt.JwtPayload;
}
```

---

### Complete Flow

```
User logs in
  → Gets access token (15 min) + refresh token (7 day cookie)

User browses for 14 minutes, clicks submit
  → authenticatedFetch checks: token expires in < 60s
  → Auto-refreshes → gets new access token
  → Submits with fresh token ✅

User idle for 16 minutes, clicks submit
  → Token expired, but server has 30s grace
  → If within grace → request succeeds ✅
  → If past grace → 401 → client tries refresh
  → Refresh token still valid (7 days) → new access token → retry ✅

User idle for 8 days
  → Access token expired, refresh token expired
  → Redirect to login
```

---

### Approach Comparison

| Approach | Complexity | UX | Security |
|---|---|---|---|
| Grace period only | Low | OK — edge cases still fail | Slightly relaxed window |
| Pre-submit check + refresh | Medium | Seamless — user never sees expiry | Strong |
| Dual token + grace + auto-refresh | Higher | Best — fully transparent | Best |

**Recommendation:** Dual tokens + auto-refresh + 30s grace period.

---

## Part 2: Campaign Access Control — Per-Campaign Token vs Session Token

### The Question

Should we create a JWT token per campaign or per session, when each campaign has a unique `campaign_id`?

---

### Comparison

| | Token Per Campaign | Token Per Session |
|---|---|---|
| Token creation | 1 token per campaign click | 1 token at login |
| API overhead | N tokens for N campaigns | 1 token for everything |
| Performance | Slower — sign token on every click | Fast — token already exists |
| Security | Granular — token only works for that campaign | Broader — works for any accessible campaign |
| Complexity | Higher | Lower |

---

### Recommendation: Session Token + Server-Side Campaign Access Check

One token for auth, server validates campaign access per request.

```
Per-campaign token (50 campaigns):
  List page:  DB query + sign 50 tokens = ~80ms
  Click one:  verify campaign token + DB query = ~15ms
  Total:      ~95ms

Hybrid (session token):
  List page:  DB query only = ~10ms
  Click one:  verify session token + access check = ~5ms (cached)
  Total:      ~15ms

~6x faster.
```

---

### Campaign List API

```ts
// app/api/campaigns/route.ts
import { NextRequest, NextResponse } from "next/server";
import { verifyAccessToken } from "@/lib/auth-verify";
import { prisma } from "@/lib/prisma";

export async function GET(req: NextRequest) {
  const token = req.headers.get("authorization")?.replace("Bearer ", "");
  if (!token) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const payload = verifyAccessToken(token);
  if (!payload) return NextResponse.json({ error: "Invalid token" }, { status: 401 });

  const campaigns = await prisma.campaign.findMany({
    where: {
      OR: [
        { visibility: "PUBLIC" },
        { ownerId: payload.userId },
        { members: { some: { userId: payload.userId } } },
      ],
    },
    select: {
      id: true,
      name: true,
      thumbnail: true,
      status: true,
      createdAt: true,
    },
    orderBy: { createdAt: "desc" },
  });

  return NextResponse.json({ campaigns });
}
```

---

### Individual Campaign API

```ts
// app/api/campaigns/[campaignId]/route.ts
import { NextRequest, NextResponse } from "next/server";
import { verifyAccessToken } from "@/lib/auth-verify";
import { canAccessCampaign } from "@/lib/campaign-access";

export async function GET(
  req: NextRequest,
  { params }: { params: { campaignId: string } }
) {
  const token = req.headers.get("authorization")?.replace("Bearer ", "");
  if (!token) return NextResponse.json({ error: "Unauthorized" }, { status: 401 });

  const payload = verifyAccessToken(token);
  if (!payload) return NextResponse.json({ error: "Invalid token" }, { status: 401 });

  const hasAccess = await canAccessCampaign(payload.userId, params.campaignId);
  if (!hasAccess) {
    return NextResponse.json({ error: "Not found or no access" }, { status: 403 });
  }

  const campaign = await prisma.campaign.findUnique({
    where: { id: params.campaignId },
  });

  return NextResponse.json({ campaign });
}
```

---

### Campaign Access Utility with Redis Cache

```ts
// lib/campaign-access.ts
import { prisma } from "@/lib/prisma";
import { redis } from "@/lib/redis";

export async function canAccessCampaign(
  userId: string,
  campaignId: string
): Promise<boolean> {
  const cacheKey = `access:${userId}:${campaignId}`;
  const cached = await redis.get(cacheKey);
  if (cached !== null) return cached === "1";

  const campaign = await prisma.campaign.findFirst({
    where: {
      id: campaignId,
      OR: [
        { visibility: "PUBLIC" },
        { ownerId: userId },
        { members: { some: { userId } } },
      ],
    },
    select: { id: true },
  });

  const hasAccess = !!campaign;
  await redis.set(cacheKey, hasAccess ? "1" : "0", "EX", 60);

  return hasAccess;
}
```

---

### Batch Prefetch on Campaign List Load

```ts
export async function prefetchUserAccess(userId: string): Promise<Set<string>> {
  const accessible = await prisma.campaign.findMany({
    where: {
      OR: [
        { visibility: "PUBLIC" },
        { ownerId: userId },
        { members: { some: { userId } } },
      ],
    },
    select: { id: true },
  });

  const ids = new Set(accessible.map((c) => c.id));

  const pipeline = redis.pipeline();
  for (const id of ids) {
    pipeline.set(`access:${userId}:${id}`, "1", "EX", 300);
  }
  await pipeline.exec();

  return ids;
}
```

---

### Database Schema

```prisma
model Campaign {
  id         String   @id @default(cuid())
  name       String
  thumbnail  String?
  status     String   @default("active")
  visibility String   @default("PRIVATE")
  ownerId    String
  owner      User     @relation(fields: [ownerId], references: [id])
  members    CampaignMember[]
  createdAt  DateTime @default(now())

  @@index([ownerId])
  @@index([visibility])
}

model CampaignMember {
  campaign   Campaign @relation(fields: [campaignId], references: [id])
  campaignId String
  user       User     @relation(fields: [userId], references: [id])
  userId     String
  role       String   @default("viewer") // viewer, editor, admin

  @@id([campaignId, userId])
  @@index([userId])
}
```

---

### URL Structure

```
/campaigns                    → list all accessible campaigns
/campaigns/[campaignId]       → individual campaign page
```

No tokens in URLs. Clean, fast, secure.

---

### Security Comparison

| Concern | Per-Campaign Token | Hybrid (Session Token) |
|---|---|---|
| Token leaked → access scope | 1 campaign only | All user's campaigns |
| Mitigated by | — | Short-lived access token (15 min) + refresh |
| Access revoked mid-session | Token still works until expiry | Server check catches immediately |
| Admin removes user from campaign | Must invalidate specific token | Next request → 403 automatically |

The hybrid approach is **more secure** for access revocation since it checks the DB/cache on every request rather than trusting a pre-signed token.

---

## Environment Variables

```env
JWT_SECRET=your-random-64-char-string
JWT_REFRESH_SECRET=another-random-64-char-string
```

---

## Summary

```
Session token (already have from login)
  ↓
Campaign list → server filters by access → return IDs
  ↓
Click campaign → server verifies access(userId, campaignId) → cached
  ↓
Fast, secure, simple
```

**Don't create tokens per campaign.** One session token + server-side access checks = faster response times and better security.
