---
title: "Zenn 2025年版：Next.js App Router × Middleware で実現する5パターンの認証ガード完全解説"
emoji: "📝"
type: "tech"
topics: ["claude", "ai", "openrouter"]
published: true
---


## TL;DR

- Next.js 15 の `middleware.ts` を使うと **サーバーサイドで認証チェック → リダイレクト** をエッジで完結できる
- Cookie / JWT / Session の検証をパターン別に解説
- `matcher` の設定ミスで「全ページが認証不要になる」罠を回避する実践的なチェックリスト付き

---

## はじめに：なぜ Middleware で認証するのか

Next.js App Router では、認証ガードの実装方法が複数あります。

| 場所 | タイミング | 欠点 |
|---|---|---|
| `layout.tsx` の `useEffect` | クライアントレンダリング後 | 一瞬コンテンツが見えるフラッシュが発生 |
| `page.tsx` の Server Component | サーバーレンダリング時 | ページごとに冗長実装になりやすい |
| **`middleware.ts`** | **エッジ・リクエスト時** | **一元管理・フラッシュなし・高速** |

Middleware はリクエストが Next.js サーバーに到達する前、または CDN エッジで実行されます。つまり **未認証ユーザーに HTML の欠片も渡さず** リダイレクトできます。

この記事では 5 つの実装パターンを、コピペで使えるコードと共に紹介します。

---

## 前提・環境

- Next.js 15.x (App Router)
- Node.js 20.x
- `jose` パッケージ (JWT 検証用・Edge Runtime 対応)
- 認証プロバイダは問わず汎用的に書きます

```bash
npm install jose
```

---

## パターン 1：Cookie の存在チェック（最小実装）

最もシンプルなパターン。`session` Cookie があれば認証済みとみなします。

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const session = request.cookies.get('session')

  if (!session) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/dashboard/:path*', '/settings/:path*'],
}
```

### ⚠️ この実装の注意点

Cookie の「存在チェックだけ」では **偽造・改ざんを検知できません**。開発初期のプロトタイプ段階のみ使用し、本番では次のパターンに移行してください。

---

## パターン 2：JWT 署名検証（本番向け基本構成）

`jose` を使って JWT の署名を Edge Runtime で検証します。

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { jwtVerify } from 'jose'

const SECRET = new TextEncoder().encode(process.env.JWT_SECRET!)

export async function middleware(request: NextRequest) {
  const token = request.cookies.get('access_token')?.value

  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  try {
    await jwtVerify(token, SECRET)
    return NextResponse.next()
  } catch {
    // 署名不正・期限切れ
    const response = NextResponse.redirect(new URL('/login', request.url))
    response.cookies.delete('access_token')
    return response
  }
}

export const config = {
  matcher: [
    /*
     * 以下を除くすべてのパスにマッチ:
     *  - api (API routes)
     *  - _next/static (静的ファイル)
     *  - _next/image (画像最適化)
     *  - favicon.ico
     *  - /login, /signup (認証不要ページ)
     */
    '/((?!api|_next/static|_next/image|favicon.ico|login|signup).*)',
  ],
}
```

### ポイント：`JWT_SECRET` の管理

`process.env.JWT_SECRET` は `.env.local` に記述し、**絶対にコードにハードコードしない**ことが鉄則です。

```bash
# .env.local (git commit 禁止)
JWT_SECRET=your-256-bit-random-secret
```

---

## パターン 3：ロールベースアクセス制御（RBAC）

JWT のペイロードに `role` クレームを持たせ、管理画面へのアクセスを制限します。

```typescript
// middleware.ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'
import { jwtVerify } from 'jose'

const SECRET = new TextEncoder().encode(process.env.JWT_SECRET!)

interface JwtPayload {
  sub: string
  role: 'user' | 'admin' | 'superadmin'
  exp: number
}

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl
  const token = request.cookies.get('access_token')?.value

  if (!token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  try {
    const { payload } = await jwtVerify(token, SECRET)
    const claims = payload as unknown as JwtPayload

    // /admin/* は admin 以上のみ
    if (pathname.startsWith('/admin') && claims.role !== 'admin' && claims.role !== 'superadmin') {
      return NextResponse.redirect(new URL('/403', request.url))
    }

    // /superadmin/* は superadmin のみ
    if (pathname.startsWith('/superadmin') && claims.role !== 'superadmin') {
      return NextResponse.redirect(new URL('/403', request.url))
    }

    // ヘッダーに role を付与 (Server Component で参照可能)
    const requestHeaders = new Headers(request.headers)
    requestHeaders.set('x-user-role', claims.role)
    requestHeaders.set('x-user-id', claims.sub)

    return NextResponse.next({ request: { headers: requestHeaders } })
  } catch {
    return NextResponse.redirect(new URL('/login', request.url))
  }
}

export const config = {
  matcher: ['/dashboard/:path*', '/admin/:path*', '/superadmin/:path*'],
}
```

### Server Component でロールを受け取る

Middleware でセットしたカスタムヘッダーは、Server Component から `headers()` で読み取れます。

```typescript
// app/admin/page.tsx
import { headers } from 'next/headers'

export default async function AdminPage() {
  const headerList = await headers()
  const role = headerList.get('x-user-role')
  const userId = headerList.get('x-user-id')

  return (
    <div>
      <p>ロール: {role}</p>
      <p>ユーザーID: {userId}</p>
    </div>
  )
}
```

---

## パターン 4：リフレッシュトークンによる自動更新

アクセストークンの有効期限が近い場合、Middleware で自動リフレッシュするパターンです。

```typescript
// lib/auth/refresh.ts
export async function refreshAccessToken(
  refreshToken: string
): Promise<string | null> {
  try {
    const res = await fetch(`${process.env.NEXT_PUBLIC_API_URL}/auth/refresh`, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ refresh_token: refreshToken }),
    })

    if (!res.ok) return null

    const data = await res.json()
    return data.access_token as string
  } catch {
    return null
  }
}
```

```typescript
// middleware.ts (一部抜粋)
import { jwtVerify } from 'jose'
import { refreshAccessToken } from './lib/auth/refresh'

const SECRET = new TextEncoder().encode(process.env.JWT_SECRET!)

// トークンの残り有効期限が 5 分以下なら更新
const REFRESH_THRESHOLD_SECONDS = 300

export async function middleware(request: NextRequest) {
  const accessToken = request.cookies.get('access_token')?.value
  const refreshToken = request.cookies.get('refresh_token')?.value

  if (!accessToken && !refreshToken) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  try {
    const { payload } = await jwtVerify(accessToken!, SECRET)
    const exp = payload.exp as number
    const now = Math.floor(Date.now() / 1000)

    // まだ有効期限が十分残っている
    if (exp - now > REFRESH_THRESHOLD_SECONDS) {
      return NextResponse.next()
    }

    // 期限切れが近い → リフレッシュ
    throw new Error('near expiry')
  } catch {
    if (!refreshToken) {
      return NextResponse.redirect(new URL('/login', request.url))
    }

    const newAccessToken = await refreshAccessToken(refreshToken)
    if (!newAccessToken) {
      const res = NextResponse.redirect(new URL('/login', request.url))
      res.cookies.delete('access_token')
      res.cookies.delete('refresh_token')
      return res
    }

    const response = NextResponse.next()
    response.cookies.set('access_token', newAccessToken, {
      httpOnly: true,
      secure: process.env.NODE_ENV === 'production',
      sameSite: 'lax',
      path: '/',
    })
    return response
  }
}
```

---

## パターン 5：NextAuth.js v5 (Auth.js) との統合

Auth.js (旧 NextAuth.js) v5 は Middleware との統合が公式でサポートされています。

```typescript
// auth.ts (プロジェクトルート)
import NextAuth from 'next-auth'
import GitHub from 'next-auth/providers/github'

export const { handlers, auth, signIn, signOut } = NextAuth({
  providers: [GitHub],
  callbacks: {
    authorized({ auth, request: { nextUrl } }) {
      const isLoggedIn = !!auth?.user
      const isOnDashboard = nextUrl.pathname.startsWith('/dashboard')

      if (isOnDashboard) {
        if (isLoggedIn) return true
        return false // ログイン画面へリダイレクト
      } else if (isLoggedIn) {
        return Response.redirect(new URL('/dashboard', nextUrl))
      }
      return true
    },
  },
})
```

```typescript
// middleware.ts
export { auth as middleware } from './auth'

export const config = {
  matcher: ['/((?!api/auth|_next/static|_next/image|favicon.ico).*)'],
}
```

### Auth.js v5 の最大の強み

`authorized` コールバック 1 つにロジックを集約できるため、**Middleware のコードがほぼゼロ行**になります。OAuthプロバイダ切り替えも `providers` 配列を変えるだけです。

---

## matcher の正規表現：よくある罠 3 選

### 罠 1：`/api` への認証チェックを忘れる

```typescript
// ❌ /api/user などの API エンドポイントが保護されない
matcher: ['/dashboard/:path*']

// ✅ API も保護する
matcher: ['/dashboard/:path*', '/api/protected/:path*']
```

### 罠 2：`_next/static` を除外し忘れて無限ループ

Middleware が `_next/static` にもマッチすると、静的ファイルの取得がリダイレクトループに陥ります。

```typescript
// ✅ 正しい除外パターン
matcher: ['/((?!_next/static|_next/image|favicon.ico).*)']
```

### 罠 3：`/:path*` と `/:path+` の違い

| パターン | `/dashboard` | `/dashboard/settings` |
|---|---|---|
| `/dashboard/:path*` | ✅ マッチ | ✅ マッチ |
| `/dashboard/:path+` | ❌ マッチしない | ✅ マッチ |

`*` は「0 個以上」、`+` は「1 個以上」のセグメントにマッチします。`/dashboard` 自体を守りたいなら `*` を使いましょう。

---

## セキュリティ強化：Cookie のオプション設定

Middleware でレスポンスに Cookie をセットする際、以下のオプションは**必ず設定**してください。

```typescript
response.cookies.set('access_token', token, {
  httpOnly: true,          // JavaScript からアクセス不可 (XSS 対策)
  secure: process.env.NODE_ENV === 'production',  // HTTPS のみ送信
  sameSite: 'lax',         // CSRF 対策 ('strict' はさらに厳格)
  maxAge: 60 * 60,         // 1時間 (秒単位)
  path: '/',
})
```

| オプション | 目的 | 推奨値 |
|---|---|---|
| `httpOnly` | XSS でのトークン盗取防止 | `true` |
| `secure` | 通信経路での盗聴防止 | 本番 `true` |
| `sameSite` | CSRF 対策 | `'lax'` 以上 |
| `maxAge` | 有効期限 | 用途に応じて |

---

## 実装チェックリスト（本番投入前）

```
[ ] JWT_SECRET は環境変数で管理しており、コードにハードコードされていない
[ ] Cookie に httpOnly / secure / sameSite が設定されている
[ ] matcher が意図したパスのみにマッチしている (_next/static 等が除外されている)
[ ] 認証失敗時にトークン Cookie を削除している
[ ] /403 や /login ページ自体が matcher に含まれていない (無限リダイレクト防止)
[ ] Edge Runtime で使用できないモジュールを誤って import していない
[ ] リフレッシュトークンの実装がある場合、失敗時の fallback ロジックがある
```

---

## まとめ

| パターン | 用途 | 難易度 |
|---|---|---|
| Cookie 存在チェック | プロトタイプ | ⭐ |
| JWT 署名検証 | 本番基本構成 | ⭐⭐ |
| RBAC | 権限管理 | ⭐⭐⭐ |
| 自動リフレッシュ | UX 改善 | ⭐⭐⭐⭐ |
| Auth.js v5 統合 | OAuth / ソーシャル認証 | ⭐⭐ |

Next.js App Router の Middleware は「エッジで動く薄いゲートキーパー」として非常に優秀です。複雑なビジネスロジックを詰め込みすぎず、**認証チェックとリダイレクトに特化**させるのが設計の鉄則です。

重い処理（DB アクセス・複雑な認可ロジック）は Server Component や Route Handler に委譲し、Middleware は軽量に保ちましょう。

---

## 参考リンク

- [Next.js 公式ドキュメント - Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware)
- [Auth.js v5 公式ドキュメント](https://authjs.dev/)
- [jose - npm (Edge Runtime 対応 JWT ライブラリ)](https://www.npmjs.com/package/jose)
- [Next.js - Setting Cookies in Middleware](https://nextjs.org/docs/app/building-your-application/routing/middleware#setting-cookies)

---

✍️ 本記事の著者: **合同会社ジモラボ**

ジモラボは、八王子を拠点に AI を活用した SaaS を多数開発しています。本記事の技術検証もそうした開発過程の副産物です。

- 🌐 公式サイト: https://locallab.jp
- 🔍 AI SEO 最適化 SaaS: [lookupai.jp](https://lookupai.jp)
- 📺 YouTube: [@locallab_llc](https://www.youtube.com/@locallab_llc)
- ✉️ お問い合わせ: info@locallab.jp

> 興味を持っていただけたら、ぜひ各 SNS のフォローもお願いします！
