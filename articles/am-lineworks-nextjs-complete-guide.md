---
title: "Next.jsでLINE WORKS OAuth・Bot・Webhook・定期通知まで実装する完全ガイド"
emoji: "🔍"
type: "tech"
topics: ["lineworks", "nextjs", "oauth", "webhook", "bot"]
published: true
---

LINE WORKS API は、単にログイン連携するだけでも便利ですが、実務ではそこから先の **Bot通知**、**Webhook受信**、**会話状態管理**、**定期バッチ連携** まで実装してはじめて価値が出ます。  
一方で、公式ドキュメントを読み進めるだけでは、`OAuth 2.0` と `Service Account` の役割分担、秘密鍵の扱い、Webhook署名検証、Botの公開設定などで詰まりやすいのも事実です。

この記事では、**Next.js App Router + TypeScript** を使って、LINE WORKS を業務システムへ実装するところまでを一気通しで整理します。対象は、社内SaaSや業務アプリに LINE WORKS を組み込みたいエンジニアです。

---

## TL;DR

- LINE WORKS 連携は、**OAuth 2.0** と **Service Account** を明確に分けて考えると実装しやすい
- ユーザーログインは OAuth、Bot送信や定期通知は Service Account が基本
- Webhook は `x-works-signature` の検証が必須
- 実務では `sub` の保存先、Bot公開設定、秘密鍵の改行、Cookieの `SameSite` でハマりやすい
- Next.js App Router でも、ログイン、Bot送信、Webhook、Cron通知まで十分構築できる

---

## この記事の対象読者

このガイドは主に次のような人向けです。

- LINE WORKS で社内ログインを実装したい
- LINE WORKS Bot から業務通知を送りたい
- Webhook を受けてユーザー入力や画像送信を処理したい
- Next.js / TypeScript / Prisma 構成で実装したい
- 公式情報だけだと埋まらない実務上のハマりどころも知りたい

---

## 前提条件

この記事では以下の構成を前提にします。

| 項目 | 内容 |
|---|---|
| 確認日 | 2026年4月29日 |
| フロントエンド / API | Next.js 16 + React 19 + TypeScript |
| 実行環境 | Node.js / Vercel |
| DB | PostgreSQL + Prisma ORM |
| 認証 | LINE WORKS OAuth 2.0 |
| Bot送信 | Service Account + JWT Bearer Grant |
| 主な用途 | 社内システムへの SSO、Bot通知、Webhook連携 |

> この記事中の環境変数名は `LINEWORKS_*` に統一しています。既存コードで `LINE_WORKS_*` を使っている場合は、読み替えてください。

---

## まず理解したい全体像

実装全体を先に掴んでおくと、細部で迷いにくくなります。

> 以下の図は **Mermaid記法** です。

```mermaid
flowchart TD
  U["ユーザー"] --> L["/api/auth/login"]
  L --> O["LINE WORKS OAuth認可"]
  O --> C["/api/auth/callback"]
  C --> P["プロフィール取得(users/me)"]
  P --> D["自アプリDBにsubを紐付け"]
  D --> S["セッションCookie発行"]

  CR["Cron / Server"] --> J["Service Account JWT生成"]
  J --> T["アクセストークン取得"]
  T --> B["Bot APIでメッセージ送信"]
  B --> U

  W["LINE WORKS"] --> H["/api/bot/callback"]
  H --> V["署名検証"]
  V --> E["message/postbackを処理"]
  E --> R["DB更新 / 応答 / 後続処理"]
```

このとき、重要なのは次の切り分けです。

- **OAuth 2.0**: ユーザーが自分でログインするための仕組み
- **Service Account**: サーバーが Bot API を呼ぶための仕組み
- **Webhook**: LINE WORKS からイベントを受ける入口
- **DB**: `sub` や会話状態を保持する場所

---

## LINE WORKS連携で最初に決めるべき設計

実装より先に、まずこの3点を決めておくと後戻りが減ります。

### 1. `sub` をどこに保存するか

OAuth のプロフィール取得で受け取る `sub` は、LINE WORKS ユーザーを識別する重要な値です。  
業務システム側では、最低でも次のどちらかで扱えるようにしておくのがおすすめです。

- アプリ側ユーザーの外部IDとして持つ
- LINE WORKS 専用の連携テーブルを持つ

たとえば Prisma ではこういう形にすると扱いやすいです。

```prisma
model AppUser {
  id           String   @id @default(cuid())
  email        String?  @unique
  name         String
  role         String
  lineWorksId  String?  @unique
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
}

model BotSession {
  lineWorksId String  @id
  status      String  @default("IDLE")
  context     String?
  updatedAt   DateTime @updatedAt
}
```

### 2. 自動プロビジョニングするか

LINE WORKS ログイン時に、存在しないユーザーをどうするかも先に決める必要があります。

- 既存ユーザーのみログイン許可
- 初回ログイン時に仮ユーザー作成
- 招待済みユーザーのみ紐付け
- 管理者承認待ちにする

社内システムなら、最初は **既存ユーザーとの照合方式** の方が運用しやすいことが多いです。  
一方で、Bot中心のシステムでは初回ログイン時に作る方が自然なケースもあります。

### 3. OAuth と Service Account を混ぜない

これはかなり大事です。

- ログイン: OAuth 2.0
- サーバーからのBot通知: Service Account
- Webhook受信: Bot Secret で署名検証

この責務分離が曖昧だと、権限不足の原因が追いにくくなります。

---

## 事前準備

記事に入る前に、最低限ここは済ませておきます。

### 必要な設定

- LINE WORKS Developer Console でアプリ登録
- OAuth 用の Client ID / Client Secret 発行
- Service Account の設定
- Private Key の発行
- Bot の作成
- 管理画面で Bot を追加・公開
- Redirect URI の設定
- Webhook URL の設定

### 環境変数

`.env.local` の例です。

```bash
LINEWORKS_CLIENT_ID=
LINEWORKS_CLIENT_SECRET=
LINEWORKS_REDIRECT_URI=http://localhost:3000/api/auth/callback
LINEWORKS_SERVICE_ACCOUNT=
LINEWORKS_PRIVATE_KEY="-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----"
LINEWORKS_BOT_ID=
LINEWORKS_BOT_SECRET=

NEXT_PUBLIC_APP_URL=http://localhost:3000
DATABASE_URL=postgresql://...
CRON_SECRET=
```

### 依存パッケージ

```bash
npm install jsonwebtoken prisma @prisma/client
npm install -D typescript @types/jsonwebtoken
```

---

## Step 1. OAuth 2.0 で LINE WORKS ログインを実装する

まずはログインです。  
流れは次の通りです。

1. `/api/auth/login` で認可URLへリダイレクト
2. `state` を Cookie に保存
3. `/api/auth/callback` で `code` を受け取る
4. `code` をアクセストークンへ交換
5. `users/me` でプロフィール取得
6. `sub` を使ってアプリ側ユーザーと紐付け
7. セッションCookieを発行

---

## OAuth用ヘルパーを作る

`lib/lineworks-auth.ts`

```ts
export interface LineWorksAuthConfig {
  clientId: string;
  clientSecret: string;
  redirectUri: string;
}

export interface LineWorksTokenResponse {
  access_token: string;
  token_type: string;
  expires_in: number;
  refresh_token?: string;
  scope: string;
}

export interface LineWorksUserProfile {
  sub: string;
  name: string | {
    lastName: string;
    firstName: string;
    phoneticLastName?: string;
    phoneticFirstName?: string;
  };
  email?: string;
  picture?: string;
}

export function getLineWorksAuthConfig(): LineWorksAuthConfig {
  return {
    clientId: process.env.LINEWORKS_CLIENT_ID || "",
    clientSecret: process.env.LINEWORKS_CLIENT_SECRET || "",
    redirectUri:
      process.env.LINEWORKS_REDIRECT_URI ||
      `${process.env.NEXT_PUBLIC_APP_URL}/api/auth/callback`,
  };
}

export function getAuthorizationUrl(config: LineWorksAuthConfig, state: string): string {
  const params = new URLSearchParams({
    client_id: config.clientId,
    redirect_uri: config.redirectUri,
    response_type: "code",
    scope: "openid profile email user.read",
    state,
  });

  return `https://auth.worksmobile.com/oauth2/v2.0/authorize?${params.toString()}`;
}

export async function getAccessToken(
  code: string,
  config: LineWorksAuthConfig
): Promise<LineWorksTokenResponse> {
  const response = await fetch("https://auth.worksmobile.com/oauth2/v2.0/token", {
    method: "POST",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: new URLSearchParams({
      code,
      client_id: config.clientId,
      client_secret: config.clientSecret,
      redirect_uri: config.redirectUri,
      grant_type: "authorization_code",
    }),
  });

  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`Failed to get access token: ${response.status} ${errorText}`);
  }

  return response.json();
}

export async function getUserProfile(accessToken: string): Promise<LineWorksUserProfile> {
  const response = await fetch("https://www.worksapis.com/v1.0/users/me", {
    headers: {
      Authorization: `Bearer ${accessToken}`,
    },
  });

  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`Failed to get user profile: ${response.status} ${errorText}`);
  }

  const data = await response.json();

  return {
    sub: data.userId || data.sub,
    name: data.userName || data.name,
    email: data.email,
    picture: data.photoUrl || data.picture,
  };
}
```

### ここでのポイント

- 認可URLは `authorize`
- code交換は `token`
- プロフィールは `users/me`
- `sub` 相当は API の返却形式差分を吸収できるようにしておくと安全
- 返却される `name` は文字列とは限らないため、オブジェクトも考慮する

---

## ログイン開始エンドポイント

`app/api/auth/login/route.ts`

```ts
import { randomBytes } from "crypto";
import { NextRequest, NextResponse } from "next/server";
import { getAuthorizationUrl, getLineWorksAuthConfig } from "@/lib/lineworks-auth";

export async function GET(request: NextRequest) {
  try {
    const config = getLineWorksAuthConfig();
    const state = randomBytes(24).toString("hex");
    const authUrl = getAuthorizationUrl(config, state);

    const isProduction = process.env.NODE_ENV === "production";

    const response = NextResponse.redirect(authUrl);
    response.cookies.set("oauth_state", state, {
      path: "/",
      httpOnly: true,
      secure: isProduction,
      sameSite: isProduction ? "none" : "lax",
      maxAge: 60 * 10,
    });

    return response;
  } catch (error) {
    console.error("[Auth Login] Failed to start OAuth flow:", error);
    return NextResponse.json(
      { error: "Failed to initiate login" },
      { status: 500 }
    );
  }
}
```

### なぜ `state` を Cookie に入れるのか

これは **CSRF対策** です。  
callback 側で受け取った `state` と、事前に保存した `state` を比較することで、不正なリクエスト混入を防げます。

### `SameSite` の罠

LINE WORKS の OAuth は外部サイトから戻ってくるため、本番 HTTPS 環境では `SameSite=None; Secure` にしないと Cookie が返ってこないことがあります。  
一方、ローカルの `http://localhost` では `Secure` を付けられないため、開発時は `lax` に切り替える方が現実的です。

---

## callback でログインを完了する

`app/api/auth/callback/route.ts`

```ts
import { NextRequest, NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";
import {
  getAccessToken,
  getUserProfile,
  getLineWorksAuthConfig,
} from "@/lib/lineworks-auth";

function normalizeUserName(name: string | { lastName: string; firstName: string }) {
  if (typeof name === "string") return name;
  return `${name.lastName || ""} ${name.firstName || ""}`.trim() || "Unknown User";
}

export async function GET(request: NextRequest) {
  try {
    const { searchParams } = new URL(request.url);
    const code = searchParams.get("code");
    const state = searchParams.get("state");
    const error = searchParams.get("error");

    if (error) {
      return NextResponse.redirect(new URL(`/auth/error?error=${encodeURIComponent(error)}`, request.url));
    }

    if (!code) {
      return NextResponse.redirect(new URL("/auth/error?error=missing_code", request.url));
    }

    const savedState = request.cookies.get("oauth_state")?.value;
    if (!savedState || savedState !== state) {
      const response = NextResponse.redirect(new URL("/auth/error?error=invalid_state", request.url));
      response.cookies.delete("oauth_state");
      return response;
    }

    const config = getLineWorksAuthConfig();
    const token = await getAccessToken(code, config);
    const profile = await getUserProfile(token.access_token);

    const lineWorksId = profile.sub;
    const userName = normalizeUserName(profile.name);

    let user = await prisma.appUser.findUnique({
      where: { lineWorksId },
    });

    if (!user) {
      user = await prisma.appUser.create({
        data: {
          lineWorksId,
          name: userName,
          email: profile.email,
          role: "USER",
        },
      });
    } else {
      user = await prisma.appUser.update({
        where: { id: user.id },
        data: {
          name: userName,
          email: profile.email,
        },
      });
    }

    const isProduction = process.env.NODE_ENV === "production";

    const response = NextResponse.redirect(new URL("/dashboard", request.url));
    response.cookies.delete("oauth_state");
    response.cookies.set("session_user_id", user.id, {
      path: "/",
      httpOnly: true,
      secure: isProduction,
      sameSite: isProduction ? "none" : "lax",
      maxAge: 60 * 60 * 24 * 30,
    });

    return response;
  } catch (error) {
    console.error("[OAuth Callback] Fatal error:", error);
    return NextResponse.redirect(new URL("/auth/error?error=callback_error", request.url));
  }
}
```

### 実務上の論点

この callback で本当に重要なのは、コード交換そのものよりも **アプリ側ユーザーにどう紐付けるか** です。

- `sub` を唯一キーとして使うのか
- 既存メールアドレスと突き合わせるのか
- 初回ログイン時に自動作成するのか
- 事前登録済みユーザーだけ通すのか

たとえば社内システムなら、`sub` と `employeeNumber` や `email` を紐付けておくことで、組織データと綺麗に接続できます。  
ここはアプリの権限モデルにも直結するため、記事で必ず触れておく価値があります。

---

## Step 2. Service Account で Bot API を呼ぶ

OAuth が「人のログイン」なら、Bot API の送信は「サーバーの権限」で行います。  
このとき使うのが **Service Account + JWT Bearer Grant** です。

流れはこうです。

1. Private Key で JWT を署名
2. token endpoint へ送る
3. access token を受け取る
4. Bot API でメッセージ送信

---

## Bot API 用の認証ヘルパー

`lib/lineworks-bot.ts`

```ts
import jwt from "jsonwebtoken";

export interface LineWorksBotConfig {
  clientId: string;
  clientSecret: string;
  serviceAccount: string;
  privateKey: string;
  botId: string;
  botSecret: string;
}

export function getLineWorksBotConfig(): LineWorksBotConfig {
  const clientId = process.env.LINEWORKS_CLIENT_ID;
  const clientSecret = process.env.LINEWORKS_CLIENT_SECRET;
  const serviceAccount = process.env.LINEWORKS_SERVICE_ACCOUNT;
  const privateKey = process.env.LINEWORKS_PRIVATE_KEY;
  const botId = process.env.LINEWORKS_BOT_ID;
  const botSecret = process.env.LINEWORKS_BOT_SECRET;

  const missing: string[] = [];
  if (!clientId) missing.push("LINEWORKS_CLIENT_ID");
  if (!clientSecret) missing.push("LINEWORKS_CLIENT_SECRET");
  if (!serviceAccount) missing.push("LINEWORKS_SERVICE_ACCOUNT");
  if (!privateKey) missing.push("LINEWORKS_PRIVATE_KEY");
  if (!botId) missing.push("LINEWORKS_BOT_ID");
  if (!botSecret) missing.push("LINEWORKS_BOT_SECRET");

  if (missing.length > 0) {
    throw new Error(`Missing required env vars: ${missing.join(", ")}`);
  }

  return {
    clientId,
    clientSecret,
    serviceAccount,
    privateKey,
    botId,
    botSecret,
  };
}

function sanitizePrivateKey(raw: string): string {
  if (!raw.includes("\n") && raw.includes("\\n")) {
    return raw.replace(/\\n/g, "\n");
  }
  return raw;
}

export async function getBotAccessToken(config?: LineWorksBotConfig): Promise<string> {
  const botConfig = config || getLineWorksBotConfig();
  const privateKey = sanitizePrivateKey(botConfig.privateKey);

  const now = Math.floor(Date.now() / 1000);
  const assertion = jwt.sign(
    {
      iss: botConfig.clientId,
      sub: botConfig.serviceAccount,
      iat: now,
      exp: now + 3600,
    },
    privateKey,
    {
      algorithm: "RS256",
      header: {
        alg: "RS256",
        typ: "JWT",
      },
    }
  );

  const body = new URLSearchParams({
    assertion,
    grant_type: "urn:ietf:params:oauth:grant-type:jwt-bearer",
    client_id: botConfig.clientId,
    client_secret: botConfig.clientSecret,
    scope: "bot user.read",
  });

  const response = await fetch("https://auth.worksmobile.com/oauth2/v2.0/token", {
    method: "POST",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
    },
    body: body.toString(),
  });

  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`Failed to get bot access token: ${response.status} ${errorText}`);
  }

  const data = await response.json();
  return data.access_token;
}
```

### ここでハマりやすい点

#### 1. Private Key の改行

Vercel や `.env` に秘密鍵を入れると、改行が `\n` の文字列として入ってしまうことがあります。  
そのまま `jwt.sign()` に渡すと壊れるので、実運用では高確率で `replace(/\\n/g, "\n")` が必要になります。

#### 2. Client Secret と Bot Secret を混同しやすい

名前が似ているので、かなりハマります。

- `LINEWORKS_CLIENT_SECRET`: OAuth / JWT Bearer Grant 用
- `LINEWORKS_BOT_SECRET`: Webhook署名検証用

この2つは用途がまったく違います。

#### 3. 毎回トークンを取り直すか、キャッシュするか

最小実装では毎回取得しても構いません。  
ただし通知量が増えると無駄が大きいので、実運用では短時間キャッシュした方が良いです。

---

## Botからメッセージを送る

```ts
export async function sendMessage(
  userId: string,
  text: string,
  config?: LineWorksBotConfig
): Promise<void> {
  const botConfig = config || getLineWorksBotConfig();
  const accessToken = await getBotAccessToken(botConfig);

  const response = await fetch(
    `https://www.worksapis.com/v1.0/bots/${botConfig.botId}/users/${userId}/messages`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${accessToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        content: {
          type: "text",
          text,
        },
      }),
    }
  );

  if (!response.ok) {
    const errorText = await response.text();
    throw new Error(`Failed to send message: ${response.status} ${errorText}`);
  }
}
```

最初はこれで十分です。  
まずは **1:1のテキスト送信を成功させる** ことを目標にすると進めやすいです。

---

## 送信テスト用エンドポイント

ローカル検証しやすいように、管理者だけ叩けるテストエンドポイントを作っておくと便利です。

`app/api/test-bot/route.ts`

```ts
import { NextRequest, NextResponse } from "next/server";
import { sendMessage } from "@/lib/lineworks-bot";
import { getCurrentUser } from "@/lib/auth";

export async function GET(request: NextRequest) {
  try {
    const user = await getCurrentUser();
    if (!user) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }

    if (user.role !== "ADMIN") {
      return NextResponse.json({ error: "Forbidden" }, { status: 403 });
    }

    const targetUserId = process.env.LINEWORKS_ADMIN_USER_ID || user.lineWorksId;
    if (!targetUserId) {
      return NextResponse.json({ error: "No target user id" }, { status: 400 });
    }

    await sendMessage(targetUserId, "LINE WORKS Bot の接続テストです");

    return NextResponse.json({
      success: true,
      message: `Message sent to ${targetUserId}`,
    });
  } catch (error) {
    console.error("[Test Bot] Failed:", error);
    return NextResponse.json(
      {
        error: "Failed to send message",
        details: error instanceof Error ? error.message : String(error),
      },
      { status: 500 }
    );
  }
}
```

この手の疎通用エンドポイントは、本番では閉じるか管理者権限に限定してください。

---

## Step 3. Webhook を受けてイベントを処理する

Bot を業務に使うなら、送信だけで終わらず Webhook が必要になります。  
ユーザーがメッセージを送ったとき、画像を投げたとき、ポストバックを押したときなどのイベントがここに飛んできます。

最初に必ずやるべきことは **署名検証** です。

---

## Webhook署名を検証する

`lib/lineworks-signature.ts`

```ts
import crypto from "crypto";
import { getLineWorksBotConfig } from "@/lib/lineworks-bot";

export function verifyBotSignature(body: string, signature: string): boolean {
  const { botSecret } = getLineWorksBotConfig();

  const hmac = crypto.createHmac("sha256", botSecret);
  hmac.update(body);
  const expected = hmac.digest("base64");

  const expectedBuffer = Buffer.from(expected);
  const actualBuffer = Buffer.from(signature);

  if (expectedBuffer.length !== actualBuffer.length) {
    return false;
  }

  return crypto.timingSafeEqual(expectedBuffer, actualBuffer);
}
```

### なぜ body を文字列で扱うのか

署名検証は **受信した生のボディ** を元に計算しないといけません。  
`await request.json()` してから再シリアライズすると、改行や順序の違いで一致しないことがあります。

そのため、Next.js では先に `await request.text()` で生の文字列を取り、その文字列で署名検証するのが安全です。

---

## Webhook の最小実装

`app/api/bot/callback/route.ts`

```ts
import { NextRequest, NextResponse } from "next/server";
import { verifyBotSignature } from "@/lib/lineworks-signature";
import { sendMessage } from "@/lib/lineworks-bot";

interface LineWorksEvent {
  type: string;
  source?: {
    userId?: string;
    type?: string;
  };
  message?: {
    type?: string;
    text?: string;
    id?: string;
    resourceId?: string;
  };
  postback?: {
    data?: string;
  };
  content?: {
    type?: string;
    text?: string;
    id?: string;
    resourceId?: string;
    fileId?: string;
    postback?: {
      data?: string;
    };
  };
  data?: string;
  timestamp?: number;
}

export async function POST(request: NextRequest) {
  try {
    const bodyText = await request.text();
    const signature = request.headers.get("x-works-signature");

    if (!signature || !verifyBotSignature(bodyText, signature)) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }

    let event: LineWorksEvent;
    try {
      event = JSON.parse(bodyText);
    } catch (error) {
      console.error("[Bot Webhook] Invalid JSON:", error);
      return NextResponse.json({ ok: true });
    }

    const userId = event.source?.userId;
    if (!userId) {
      return NextResponse.json({ ok: true });
    }

    if (event.type === "message") {
      const message = event.message || event.content;

      if (message?.type === "text" && message.text) {
        await sendMessage(userId, `受信しました: ${message.text}`);
      }

      if (message?.type === "image") {
        await sendMessage(userId, "画像を受信しました。後続処理を開始します。");
      }
    }

    if (event.type === "postback") {
      const data = event.postback?.data || event.data || event.content?.postback?.data;

      if (data === "help") {
        await sendMessage(userId, "利用できるコマンドを表示します。");
      }
    }

    return NextResponse.json({ ok: true });
  } catch (error) {
    console.error("[Bot Webhook] Error:", error);
    return NextResponse.json({ ok: true });
  }
}
```

### なぜエラー時でも `200` を返す設計が多いのか

Webhook は再送制御や外部サービスとの相性があるため、業務処理で失敗しても入口では `200` を返す設計が現実的なことがあります。  
ただしこれは「失敗を無視する」という意味ではありません。

必要なのは次の3点です。

- 署名不正は `401`
- 業務処理失敗はログへ記録
- 必要なら内部通知や再試行機構へ回す

---

## Step 4. 会話状態をDBで持つ

Webhook で単発の応答を返すだけなら、ここまでで十分です。  
しかし、業務Bot はしばしば多段階入力になります。

たとえば次のようなフローです。

1. メニューを表示
2. ユーザーが「経費登録」を選ぶ
3. 車両を選ぶ
4. 金額を入力
5. 日付を入力
6. 確認して保存

このとき、HTTP はステートレスなので、途中状態を自前で保持しないと会話が崩れます。  
そのため、実務では `botStatus` と `botContext` のような形で DB に持つ設計が扱いやすいです。

```ts
type BotStatus =
  | "IDLE"
  | "WAIT_MENU"
  | "WAIT_VEHICLE_SELECT"
  | "WAIT_AMOUNT"
  | "WAIT_DATE"
  | "WAIT_CONFIRM";

interface BotContext {
  vehicleId?: string;
  amount?: number;
  date?: string;
}
```

更新ヘルパーはこんな形です。

```ts
import { prisma } from "@/lib/prisma";

export async function updateBotState(
  lineWorksId: string,
  status: BotStatus,
  context?: BotContext
) {
  await prisma.botSession.upsert({
    where: { lineWorksId },
    create: {
      lineWorksId,
      status,
      context: context ? JSON.stringify(context) : null,
    },
    update: {
      status,
      context: context ? JSON.stringify(context) : null,
    },
  });
}

export async function getBotState(lineWorksId: string) {
  const row = await prisma.botSession.findUnique({
    where: { lineWorksId },
  });

  return {
    status: (row?.status || "IDLE") as BotStatus,
    context: row?.context ? JSON.parse(row.context) : {},
  };
}
```

### これを入れるメリット

- 会話の途中再開ができる
- キャンセル処理を実装しやすい
- 画像送信待ちなどの分岐が作れる
- Bot が単なる通知役ではなく業務UIになる

公式サンプルだけだとここまで踏み込んだ例は少ないので、記事の価値が出やすいポイントです。

---

## Step 5. 画像受信から後続処理につなぐ

実務で Bot を使うなら、テキストだけでなく画像受信も重要です。  
領収書、点検記録、現場写真、OCR前提の帳票など、Botが“入力チャネル”になるケースは多いです。

まずはイベント側で画像を検知します。

```ts
if (event.type === "message") {
  const message = event.message || event.content;

  if (message?.type === "image") {
    const fileId = (message as any).fileId || message.id || message.resourceId;
    if (fileId) {
      await queueImageProcessing({
        lineWorksUserId: userId,
        fileId,
      });
      await sendMessage(userId, "画像を受信しました。解析を開始します。");
    }
  }
}
```

### 実装の考え方

ここで全部同期処理しない方が安全です。

- Webhook では受信だけする
- DB や Queue にジョブを積む
- 非同期で画像取得、保存、OCR、AI解析を走らせる

最小構成なら同期でも動きますが、タイムアウトや失敗時再試行を考えると分離した方が運用しやすいです。

---

## Step 6. 定期通知を実装する

業務システムで LINE WORKS が一番効くのは、実は通知です。  
ログインよりも「必要な人に必要な情報を届ける」方が日常的な価値を出しやすいです。

たとえば次のような通知です。

- 日次サマリー
- 締切リマインド
- 期限切れアラート
- 承認依頼
- エラー通知

ここでは Cron から LINE WORKS Bot で通知する例を示します。

---

## Cronエンドポイントの例

`app/api/cron/daily-summary/route.ts`

```ts
import { NextRequest, NextResponse } from "next/server";
import { prisma } from "@/lib/prisma";
import { sendMessage } from "@/lib/lineworks-bot";

function verifyCronSecret(request: NextRequest) {
  const expected = process.env.CRON_SECRET;
  const actual = request.headers.get("x-cron-secret");

  if (!expected || actual !== expected) {
    return false;
  }
  return true;
}

export async function GET(request: NextRequest) {
  try {
    if (!verifyCronSecret(request)) {
      return NextResponse.json({ error: "Unauthorized" }, { status: 401 });
    }

    const today = new Date();

    const managers = await prisma.appUser.findMany({
      where: {
        role: "MANAGER",
        lineWorksId: { not: null },
      },
      select: {
        id: true,
        name: true,
        lineWorksId: true,
      },
    });

    const summaryText = [
      `日次サマリー (${today.toLocaleDateString("ja-JP")})`,
      "",
      "- 新規申請: 12件",
      "- 承認待ち: 4件",
      "- 差し戻し: 1件",
      "",
      "詳細はダッシュボードを確認してください。",
    ].join("\n");

    const results = [];

    for (const manager of managers) {
      if (!manager.lineWorksId) continue;

      try {
        await sendMessage(manager.lineWorksId, summaryText);
        results.push({
          userId: manager.id,
          name: manager.name,
          success: true,
        });
      } catch (error) {
        console.error(`[Daily Summary] Failed for ${manager.name}:`, error);
        results.push({
          userId: manager.id,
          name: manager.name,
          success: false,
        });
      }
    }

    return NextResponse.json({
      success: true,
      sent: results.filter((x) => x.success).length,
      failed: results.filter((x) => !x.success).length,
      results,
    });
  } catch (error) {
    console.error("[Daily Summary] Fatal error:", error);
    return NextResponse.json(
      {
        error: "Internal server error",
        details: error instanceof Error ? error.message : String(error),
      },
      { status: 500 }
    );
  }
}
```

### 実務でさらに欲しくなること

- 部門や支店ごとの配信先解決
- 通知対象ロールの絞り込み
- 通知失敗時の管理者アラート
- 送信履歴の監査ログ
- 再送制御

たとえば「支店長 + 同支店フォロワーに送る」といったルールは、配信先解決関数に閉じ込めると整理しやすいです。

```ts
export async function getNotificationRecipients(branchId: string): Promise<string[]> {
  const users = await prisma.appUser.findMany({
    where: {
      branchId,
      lineWorksId: { not: null },
      role: { in: ["BRANCH_MANAGER", "FOLLOWER"] },
    },
    select: { lineWorksId: true },
  });

  return users
    .map((u) => u.lineWorksId)
    .filter((id): id is string => !!id);
}
```

---

## 運用時に欲しいエラー通知

定期処理が落ちたとき、ログだけでは気づかないことがあります。  
そのため、失敗時は管理者へ LINE WORKS で通知する設計も有効です。

```ts
export async function sendSystemErrorNotification(source: string, error: unknown) {
  const admins = await prisma.appUser.findMany({
    where: {
      role: "SYSTEM_ADMIN",
      lineWorksId: { not: null },
    },
    select: {
      name: true,
      lineWorksId: true,
    },
  });

  const message = [
    "システムエラー通知",
    "",
    `機能: ${source}`,
    `時刻: ${new Date().toLocaleString("ja-JP", { timeZone: "Asia/Tokyo" })}`,
    `内容: ${error instanceof Error ? error.message : "Unknown error"}`,
  ].join("\n");

  for (const admin of admins) {
    if (!admin.lineWorksId) continue;

    try {
      await sendMessage(admin.lineWorksId, message);
    } catch (notifyError) {
      console.error(`[System Error Notification] Failed for ${admin.name}:`, notifyError);
    }
  }
}
```

このパターンはかなり実務的で、Bot が単なるフロント機能ではなく、運用インフラの一部になります。

---

## Flex Message や Quick Reply はいつ使うべきか

テキストだけでも Bot は成立しますが、多段階入力や業務選択が増えると UI 的に苦しくなります。  
その場合は、Quick Reply や Flex Message を使うと体験が一気に改善します。

たとえば「入力メニューを出す」だけでも、選択肢が明確になります。

```ts
interface QuickReply {
  items: Array<{
    type: "action";
    action: {
      type: "message" | "postback";
      label: string;
      displayText: string;
      text?: string;
      data?: string;
    };
  }>;
}
```

送信時に `quickReply` を付ける形です。

```ts
export async function sendTextWithQuickReply(
  userId: string,
  text: string,
  quickReply: QuickReply
) {
  const botConfig = getLineWorksBotConfig();
  const accessToken = await getBotAccessToken(botConfig);

  const response = await fetch(
    `https://www.worksapis.com/v1.0/bots/${botConfig.botId}/users/${userId}/messages`,
    {
      method: "POST",
      headers: {
        Authorization: `Bearer ${accessToken}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({
        content: {
          type: "text",
          text,
          quickReply,
        },
      }),
    }
  );

  if (!response.ok) {
    throw new Error(await response.text());
  }
}
```

ただし記事としては、最初からリッチUIに寄せすぎるよりも、まずはテキストの最小実装を通してから発展編として触れる方が読みやすいです。

---

## LINE WORKS 実装で本当にハマりやすいポイント

ここは公式の説明だけでは埋まりにくいので、独立して整理しておきます。

---

### 1. Botを作っただけでは送れない

Developer Console で Bot を作成しただけでは、実際にメッセージ送信できないことがあります。  
管理画面側で Bot を追加・公開し、利用対象を整えてはじめて動くケースがあるため、API実装より先に設定を疑うべきです。

---

### 2. Client Secret と Bot Secret を混同する

これは本当に多いです。

- `Client Secret`: OAuth / JWTトークン発行
- `Bot Secret`: Webhook署名検証

認証失敗が出たときは、まずここを見直してください。

---

### 3. Private Key の改行が壊れる

環境変数に入れた秘密鍵が、改行を含まず `\n` の文字列として保存される問題です。  
署名エラーが出たらまず疑ってよいポイントです。

```ts
const privateKey = process.env.LINEWORKS_PRIVATE_KEY?.replace(/\\n/g, "\n");
```

---

### 4. `state` Cookie が返ってこない

本番環境で `state mismatch` が起きる場合、OAuth処理が悪いのではなく、Cookie の属性で落ちていることがあります。

- 本番: `SameSite=None; Secure`
- 開発: `SameSite=Lax`

この切り替えはかなり重要です。

---

### 5. Webhook署名検証を JSON パース後にやってしまう

これは危険です。  
署名検証は生の body 文字列でやります。

```ts
const bodyText = await request.text();
const signature = request.headers.get("x-works-signature");
verifyBotSignature(bodyText, signature);
```

---

### 6. `sub` の保存先を後回しにする

後から設計すると、ユーザー同期や権限制御が崩れます。  
最初に「LINE WORKS ユーザーを自アプリでどう識別するか」を決めるべきです。

---

### 7. OAuth と Service Account の役割を混ぜる

ユーザーログインの文脈と、サーバーが通知を送る文脈は別です。  
これを混ぜると、必要スコープや失敗原因の切り分けが難しくなります。

---

### 8. Webhook処理で同期処理を詰め込みすぎる

画像解析やOCR、AI処理をその場で全部やると、タイムアウトや再送リスクが増えます。  
Webhook は受付、重い処理は非同期に分けるのが安全です。

---

## 動作確認の進め方

技術記事として信頼されるには、単にコードを載せるだけでなく、**動作確認済みであることの証拠** が重要です。  
最低限、次の確認は記事に入れたいです。

### OAuth

- `/api/auth/login` で LINE WORKS の認可画面へ遷移する
- callback で `code` を受け取れる
- `users/me` のプロフィール取得に成功する
- DB に `lineWorksId` が保存される
- セッションCookieが発行される

### Bot送信

- テストエンドポイントから 1:1 メッセージ送信できる
- Bot未公開時に失敗することを確認できる
- 権限不足時のエラーメッセージを採取できる

### Webhook

- 署名が正しい場合に受信できる
- 署名が不正な場合に `401` を返す
- text / image / postback を最低1回ずつ確認する

### Cron通知

- Secret付きでエンドポイントを叩ける
- 送信対象者の解決が正しい
- 送信成功件数と失敗件数が取れる
- エラー時通知が機能する

---

## 記事に載せると強いスクリーンショット / レスポンス例

公開記事にするなら、次の素材を入れると一気に説得力が出ます。

- Developer Console の OAuth 設定画面
- Bot 作成 / 公開画面
- LINE WORKS ログイン画面
- callback 後のダッシュボード画面
- `users/me` のレスポンス例
- LINE WORKS に届いた Bot 通知画面
- Webhook 受信ログ
- 署名不正時のレスポンス
- Cron 実行結果 JSON

レスポンス例のサンプルはこんな形です。

```json
{
  "success": true,
  "sent": 3,
  "failed": 0,
  "results": [
    {
      "userId": "clx123...",
      "name": "営業部 部門長",
      "success": true
    }
  ]
}
```

---

## 実運用に入るなら追加で考えたいこと

ここまでで MVP としては十分ですが、本番運用ではさらに次を検討したくなります。

### 1. RBAC との接続

LINE WORKS ログイン後、アプリ側のロールにどう繋ぐかです。

- 一律 `USER`
- 部門 / 支店 / 管理者ロールへ紐付け
- LINE WORKS 側の属性や事前登録情報で制御

### 2. 監査ログ

誰がいつログインし、誰に何を送ったか、Webhook で何を登録したかを残したい場面は多いです。

### 3. リトライ戦略

外部APIなので、瞬間的な失敗は起こり得ます。  
少なくとも次は欲しくなります。

- 429 / 5xx のリトライ
- バックオフ
- 送信ジョブ化
- Dead Letter Queue 的な退避

### 4. PII を含むログ出力の整理

ローカルでは詳細ログが便利ですが、本番では生のメッセージ本文や画像関連IDの出し方に注意が必要です。

### 5. 画像処理の非同期化

OCR や AI 解析までつなぐ場合、Webhook 同期処理ではなく Queue 化した方が安定します。

---

## サンプル構成まとめ

この記事のコードをまとめると、プロジェクトの構成は大体こうなります。

```text
app/
  api/
    auth/
      login/route.ts
      callback/route.ts
    bot/
      callback/route.ts
    cron/
      daily-summary/route.ts
    test-bot/
      route.ts

lib/
  lineworks-auth.ts
  lineworks-bot.ts
  lineworks-signature.ts
  prisma.ts
  auth.ts
  bot-state.ts
```

ここまで切り出しておくと、責務が比較的明快です。

- `lineworks-auth.ts`: OAuth
- `lineworks-bot.ts`: Service Account / Bot送信
- `lineworks-signature.ts`: Webhook署名
- `bot-state.ts`: 会話状態
- `route.ts`: HTTP入口

---

## ここまでの実装を通して見えてくる設計指針

LINE WORKS API は、見た目以上に「認証の種類」と「権限の境界」が大事です。  
このガイドで一貫している設計指針は次の通りです。

1. ログインは OAuth
2. 通知は Service Account
3. Webhook は署名検証を最優先
4. `sub` は早めに永続化設計を決める
5. 会話は DB に状態を持つ
6. 重い処理は Webhook と分離する
7. 通知失敗も運用対象として扱う

この7点が揃うと、単なる“API接続”ではなく、業務システムの一部として LINE WORKS を扱えるようになります。

---

## まとめ

Next.js で LINE WORKS を使う実装は、表面的には OAuth と API 呼び出しだけに見えますが、実務ではその先の設計が重要です。

- ユーザーログインは OAuth 2.0
- Bot送信は Service Account
- Webhook は `x-works-signature` を検証
- `sub` をアプリ側ユーザーにどう結び付けるかを先に決める
- 業務Botなら会話状態を DB に持つ
- 定期通知やエラー通知まで入れると、LINE WORKS が本当に業務に効いてくる

LINE WORKS は、SSO、通知、入力チャネル、業務オペレーションの接点としてかなり強いプラットフォームです。  
ただし、設定と実装の責務分離を曖昧にすると途端に複雑になります。

だからこそ、最初に全体像を整理し、  
**OAuth、Bot、Webhook、状態管理、運用通知** をそれぞれの責務で分けて実装するのがいちばんきれいです。

---

## 付録: 最小動作チェックリスト

```md
### OAuth
- [ ] /api/auth/login で認可画面へ遷移する
- [ ] callback で code を受け取れる
- [ ] users/me 取得に成功する
- [ ] lineWorksId(sub) を保存できる
- [ ] session cookie が発行される

### Bot送信
- [ ] Service Account で access token を取得できる
- [ ] 1:1 テキスト送信できる
- [ ] Bot が管理画面で公開済みである

### Webhook
- [ ] x-works-signature を検証している
- [ ] text イベントを処理できる
- [ ] postback イベントを処理できる
- [ ] image イベントを受け取れる

### 運用
- [ ] Cron secret で保護している
- [ ] 定期通知を送信できる
- [ ] エラー時に管理者通知できる
- [ ] PII を本番ログに出しすぎていない
```

---

## 付録: cURL での疎通確認例

### Cron エンドポイント

```bash
curl -i \
  -H "x-cron-secret: YOUR_CRON_SECRET" \
  http://localhost:3000/api/cron/daily-summary
```

### Webhook ローカル確認の考え方

Webhook は本来 LINE WORKS から飛んできますが、ローカルでは署名つきリクエストを再現しないと本番同様の確認になりません。  
そのため、まずは次の段階で検証するのがおすすめです。

1. 署名検証を一時的に開発モードだけ簡略化してイベント構造を確認
2. 署名付きのテスト送信に切り替える
3. 本番相当環境で再検証する

---

## 付録: この構成をさらに発展させるなら

この記事の延長線上で次に書けるテーマもかなりあります。

- LINE WORKS OAuth + Prisma RBAC 設計
- LINE WORKS Bot による業務入力フロー設計
- 画像受信から OCR / AI 解析までの実装
- LINE WORKS 通知を中心にした運用監視設計
- Flex Message と Quick Reply を使った業務UI化

この完全ガイドを土台にしておけば、そのまま発展記事にもつなげやすいはずです。