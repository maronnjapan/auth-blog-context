---
title: "どん兵衛の出来上がりを待っている間にBetter Auth+Neon+VikeのアプリをCloudflare Workersにデプロイする"
emoji: "🦔"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["BetterAuth","Neon","Vike","CloudflareWorkers"]
published: true
published_at: 2025-12-25 12:15
---

## はじめに
「あー、どん兵衛ができあがりを待っている間に、Typescript作られている認証、認可フレームワークである[Better Auth](https://www.better-auth.com/)のセットアップからデプロイまで試したいなー」  
こんな風に思ったことはありませんか？  
私はないですが、誰かは思っているかもしれません。
今回はそういった方を対象に、どん兵衛ができあがるくらいの時間で、Better AuthをNeon、Vike、Cloudflare Workersを使ってセットアップからデプロイまで行うスクリプトを紹介します。
どん兵衛ができあがるまでの時間でBetter Authを試してみましょう！

## 前提条件
流石にまっさらな状態から、どん兵衛ができあがる時間でセットアップからデプロイを行うのは難しいため、以下の設定が行われていることが前提となります。
- Cloudflare アカウント作成済
- Neon アカウント作成済
- Node.js インストール済
    
また、以下のコマンドも使用可能である必要があります。
- neon
    - インストール方法: https://neon.com/docs/reference/neon-cli#install
- psql (PostgreSQLクライアント)
    - Ubuntuの場合` sudo apt install postgresql postgresql-contrib`でインストール可能
- jq
- pnpm

ここまでは準備しておいてください。
では実際にスクリプトを実行して、デプロイまで行ってみましょう！

## セットアップからデプロイまでの実行手順
まずはそれぞれ任意のディレクトリ配下で、`setup.sh`を作成し、以下のコードを貼り付けてください。
:::details setup.shのコード

```bash
#!/bin/bash
set -e

# 以下のパッケージがインストールされていることを前提とする
# - neon-cli
# - jq
# - psql (PostgreSQLクライアント)
# - Node.js
# - pnpm

# Neonにログイン
neon auth
# cloudflareにログイン
pnpm dlx wrangler login

# Neon組織IDを取得して.neonファイルを生成（対話モード回避のため）
ORG_ID=$(neon orgs list --output json | jq -r '.[0].id')
if [ -z "$ORG_ID" ]; then
    echo "Failed to get organization ID" >&2
    exit 1
fi

# .neonファイルを生成（プロジェクトルートディレクトリに作成）
cat > .neon <<EOF
{
  "orgId": "$ORG_ID"
}
EOF

echo "Created .neon file with orgId: $ORG_ID"

# プロジェクト作成し、IDを変数に格納
PROJECT_ID=$(neon projects create --name authAppDb --output json | jq -r '.project.id')

# アプリのセットアップと必要なパッケージのインストール
pnpm create vike@latest --react --tailwindcss --shadcn-ui --hono --drizzle --cloudflare --eslint --prettier
mv my-app sample-auth-app
cp ./setup-neon-better-auth.cjs ./sample-auth-app/
cd sample-auth-app
pnpm install
pnpm add better-auth @neondatabase/serverless @radix-ui/react-label @radix-ui/react-slot

node ./setup-neon-better-auth.cjs

# DBへの接続情報取得
CONNECTION_STRING=$(neon connection-string --project-id $PROJECT_ID)

# ローカル実行用に.envファイルを生成
echo "DATABASE_URL=$CONNECTION_STRING" > .env
echo "BETTER_AUTH_SECRET=$(LC_ALL=C tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c 32)" >> .env
echo "BETTER_AUTH_URL=\"http://localhost:3000\"" >> .env

# Drizzle ORMのスキーマと型を生成
pnpm dlx @better-auth/cli@latest generate --config ./better-auth.config.ts --output ./database/drizzle/schema/auth.ts
# テーブル設定が記載されたマイグレーションファイル(SQLファイル)を生成
pnpm drizzle:generate
# Cloudflare Workers用の型を生成
pnpm generate-types

# SQLファイルをpsqlコマンド経由で実行してDBにテーブルを作成
for sql_file in ./database/migrations/*.sql; do
    [ -f "$sql_file" ] && psql "$CONNECTION_STRING" -f "$sql_file"
done

# Cloudflare Workers を初回デプロイしてサービスを作成し、URLを取得
DEPLOY_LOG=$(pnpm run deploy 2>&1)
WORKER_URL=$(printf "%s\n" "$DEPLOY_LOG" | grep -oE 'https://[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.workers\.dev' | tail -n1)

if [ -n "$WORKER_URL" ]; then
    # Better Authの実行に必要なシークレット、DB接続情報、デプロイしたURLを環境変数に設定して再デプロイ
    BETTER_AUTH_SECRET=$(LC_ALL=C tr -dc 'a-zA-Z0-9' < /dev/urandom | head -c 32)
    printf "%s" "$CONNECTION_STRING" | pnpm dlx wrangler secret put DATABASE_URL
    printf "%s" "$BETTER_AUTH_SECRET" | pnpm dlx wrangler secret put BETTER_AUTH_SECRET
    # アプリURLは暗号化する必要がないので、平文で設定
    pnpm dlx wrangler deploy --var "BETTER_AUTH_URL:${WORKER_URL}"
    # DB接続の遅延対策にHyperDriveを設定
    pnpm dlx wrangler hyperdrive create sampleauthapphyperdrive --connection-string="$CONNECTION_STRING"
    # アプリとHyperDriveを紐付けるために再度デプロイ
    pnpm run deploy
else
    echo "Failed to determine Workers URL" >&2
fi
```

:::

次に、同じ階層で`setup-neon-better-auth.cjs`を作成し、以下のコードを貼り付けてください。  
ちなみに、Vikeの設定の関係上、ファイル拡張子は`.cjs`でないと実行できません。
なので、拡張子は必ずcjsにしてください。
:::details setup-neon-better-auth.cjsのコード

```javascript
#!/usr/bin/env node

/**
 * Neon + Better Auth セットアップスクリプト
 *
 * このスクリプトは、Cloudflare Workers + Drizzle + Vikeで構成されたアプリにNeon + Better Authを導入するために必要なコードを生成します。
 * 以下の内容のコードを生成、書き込みしています:
 * - NeonをDrizzle ORM経由で実行するための設定
 * - Better Auth実行に必要なDB構造を生成するための設定
 * - Better Authのサーバー・クライアント設定
 * - Better Authを利用するためのUIコンポーネント（サインイン、サインアップ、ユーザープロフィール）
 * - Cloudflare D1の設定をNeonに差し替える設定
 * - Better Auth+Neon実行に必要な最低限のwrangler.jsoncの設定
 *
 * 反対に以下の内容は行っていません:
 * - 環境変数ファイル(.env)の作成
 * - 必要なパッケージのインストール
 * - worker-configuration.d.ts (自動生成ファイル)
 *
 */

const fs = require('fs');
const path = require('path');

// ベースディレクトリ（このスクリプトがあるディレクトリ）
const baseDir = __dirname;

/**
 * ディレクトリを再帰的に作成
 */
function ensureDirectoryExists(filePath) {
  const dirname = path.dirname(filePath);
  if (fs.existsSync(dirname)) {
    return;
  }
  ensureDirectoryExists(dirname);
  fs.mkdirSync(dirname);
}

/**
 * ファイルを書き込む（上書きまたは新規作成）
 */
function writeFile(relativePath, content) {
  const fullPath = path.join(baseDir, relativePath);
  ensureDirectoryExists(fullPath);

  const exists = fs.existsSync(fullPath);
  fs.writeFileSync(fullPath, content, 'utf8');

  console.log(`${exists ? '上書き' : '新規作成'}: ${relativePath}`);
}

// ファイル定義
const files = {
  // wrangler.jsonc (d1_databases と hyperdrive を除外)
  'wrangler.jsonc': `{
\t"$schema": "./node_modules/wrangler/config-schema.json",
\t"name": "sample-auth-app",
\t"compatibility_date": "2025-09-06",
\t// Node.jsを有効化
\t"compatibility_flags": [
\t\t"nodejs_compat"
\t],
\t"main": "virtual:photon:cloudflare:server-entry",
\t// ログを有効化
\t"observability": {
\t\t"logs": {
\t\t\t"enabled": true
\t\t}
\t}
}
`,

  // Better Auth設定
  'better-auth.config.ts': `import { neon } from '@neondatabase/serverless';
import { drizzle } from 'drizzle-orm/neon-http';
import { drizzleAdapter } from 'better-auth/adapters/drizzle';
import { betterAuth } from 'better-auth';
import { betterAuthOptions } from './lib/better-auth/server/options';

/**
 * Better Authで使用するデータのスキーマ生成用設定ファイル
 * @better-auth/cliを使い、Drizzle経由でDBを操作できるコードを生成するために使用する
 */

const { DATABASE_URL, BETTER_AUTH_URL, BETTER_AUTH_SECRET } = process.env;

// NeonをDBとし、Drizzleのクライアント作成
const sql = neon(DATABASE_URL!);
const db = drizzle(sql);

export const auth: ReturnType<typeof betterAuth> = betterAuth({
    database: drizzleAdapter(db, { provider: 'pg' }), 
});
`,

  // Better Auth クライアント
  'lib/better-auth/client/index.ts': `import { createAuthClient } from "better-auth/react"

/** クライアント側で使用する機能を定義する */

export const authClient = createAuthClient({
    /** 
     * フロントとサーバーのドメインが異なる場合は設定する。
     * 今回は同じURLなのでコメントアウトしている
     */
    // baseURL: "http://localhost:3000"
})

export const { signIn, signOut, signUp, useSession } = authClient
`,

  // Better Auth サーバー
  'lib/better-auth/server/index.ts': `import { neon } from "@neondatabase/serverless";
import { drizzle } from "drizzle-orm/neon-http";
import { drizzleAdapter } from "better-auth/adapters/drizzle";
import { betterAuth } from "better-auth";
import { betterAuthOptions } from "./options";
import * as schema from "../../../database/drizzle/schema/auth"
import { dbNeon } from "@/database/drizzle/db";

/**
 * サーバー側でBetter Authの機能を使用するための設定
 * Cloudflare Workersで動かす想定なので、process.envではなくBinding経由で環境変数を設定する
 */
export const auth = (env: CloudflareBindings): ReturnType<typeof betterAuth> => {
    const db = dbNeon(env.DATABASE_URL)

    return betterAuth({
        ...betterAuthOptions,
        /** 
         * better-auth.config.tsではスキーマを生成するための設定だったのでschemaプロパティは不要
         * しかし、アプリで実際に動かす場合はスキーマがないとDB操作ができないため、必ず設定すること
         */ 
        database: drizzleAdapter(db, { provider: "pg", schema }),
        baseURL: env.BETTER_AUTH_URL,
        secret: env.BETTER_AUTH_SECRET,
    });
};
`,

  // Better Auth オプション
  'lib/better-auth/server/options.ts': `import { BetterAuthOptions } from "better-auth";

/**
 * Better Authのオプション
 *
 * Docs: https://www.better-auth.com/docs/reference/options
 */
export const betterAuthOptions: BetterAuthOptions = {
    /**
     * The name of the application.
     */
    appName: "AuthApp",

    /**
     * 今回はユーザー名、パスワード認証のみなので有効にする
     */
    emailAndPassword: {
        enabled: true,
    }
};
`,

  // Drizzle設定（Neon）
  'drizzle.config.ts': `import "dotenv/config";
import { defineConfig } from "drizzle-kit";

/**
 * Drizzleの設定ファイル
 * pnpm drizzle:generateを実行したときテーブルの生成などを行うマイグレーションファイルを生成する際に使用する
 */

if (!process.env.DATABASE_URL) {
  throw new Error("DATABASE_URL is not set");
}

export default defineConfig({
  schema: "./database/drizzle/schema/*",
  dialect: "postgresql",
  out: "./database/migrations",
  dbCredentials: {
    // アプリ実行用のコードではないので、process.env経由で接続用URLを取得している。
    url: process.env.DATABASE_URL!,
  },
});
`,

  // データベース抽象化
  'database/drizzle/db.ts': `import { neon } from "@neondatabase/serverless";
import { drizzle as drizzleNeon } from "drizzle-orm/neon-http";

export function dbNeon(databaseUrl: string) {
  const neonClient = neon(databaseUrl);

  return drizzleNeon(neonClient);
}
`,

  // サーバーエントリー
  'server/entry.ts': `import { dbMiddleware } from "./db-middleware";
import { apply, serve } from "@photonjs/hono";
import { Hono } from "hono";
import { auth } from "@/lib/better-auth/server";

/**
 * アプリケーションを起動するエントリーファイル
 * ここではsetup.shで「pnpm create vike@latest --react --tailwindcss --shadcn-ui --hono --drizzle --cloudflare --eslint --prettier」で作成されたファイルに以下の変更を加えている。
 * HonoにBetter Authの設定をマウント
 */

const port = process.env.PORT ? parseInt(process.env.PORT, 10) : 3000;
type Bindings = {
  DATABASE_URL: string;
} & CloudflareBindings;

export default startApp() as unknown;

function startApp() {
  const app = new Hono<{ Bindings: Bindings }>();


  app.on(['GET', 'POST'], '/api/auth/*', (c) => {
    return auth(c.env).handler(c.req.raw);
  });

  apply(app, [
    // Make database available in Context as \`context.db\`
    dbMiddleware,
  ]);

  return serve(app, {
    port,
  });
}
`,

  // DBミドルウェア
  'server/db-middleware.ts': `import { dbNeon } from "../database/drizzle/db";
import { enhance, type UniversalMiddleware } from "@universal-middleware/core";

/**
 * Drizzleクライアントをリクエストコンテキストに注入している&補完が効くように型定義を行っている
 * Cloudflare Workersはサーバーレス環境のため、サーバーが常時稼働しているわけではない
 * そのため、ミドルウェアを用いて都度リクエストのコンテキスト内にDrizzleクライアントを含めるようにしている
 */

declare global {
  namespace Universal {
    interface Context {
      db: ReturnType<typeof dbNeon>;
    }
  }
}

export const dbMiddleware: UniversalMiddleware = enhance(
  async (_request, context, _runtime) => {
    // Cloudflare Workersの場合のみ、ランタイムの中にDBとの接続情報があるので取得している
    const databaseUrl =
      _runtime.runtime === "workerd" && _runtime?.env?.DATABASE_URL ? (_runtime.env.DATABASE_URL as string) : undefined;

    // Cloudflare Workers以外の実行は今回許容しないようにしている
    if (!databaseUrl) {
      throw new Error("DATABASE_URL is not set in environment variables");
    }
    const db = dbNeon(databaseUrl);

    return {
      ...context,
      // Sets pageContext.db
      db: db,
    };
  },
  {
    name: "my-app:db-middleware",
    immutable: false,
  },
);
`,

  // グローバル型定義
  'global.d.ts': `import { dbNeon } from "./database/drizzle/db";

// contextにアクセスする際、補完が効くようにした。
declare global {
  namespace Vike {
    interface PageContextServer {
      env: Env;
      db: ReturnType<typeof dbNeon>;
    }
  }
}

export { };
`,

  // トップページ
  'pages/index/+Page.tsx': `import { useSession } from "@/lib/better-auth/client";
import { SignInForm } from "./SignInForm";
import { SignUpForm } from "./SignUpForm";
import { UserProfile } from "./UserProfile";

export default function Page() {
  const { data: session, isPending } = useSession();

  return (
    <div className="container mx-auto px-4 py-8">
      <div className="max-w-4xl mx-auto">
        <h1 className="text-4xl font-bold mb-2">Better Auth Demo</h1>
        <p className="text-muted-foreground mb-8">
          サインイン、サインアップ、サインアウト機能のデモ
        </p>

        {isPending ? (
          <div className="flex justify-center items-center py-12">
            <p className="text-muted-foreground">読み込み中...</p>
          </div>
        ) : session ? (
          <div className="flex justify-center">
            <UserProfile />
          </div>
        ) : (
          <div className="grid md:grid-cols-2 gap-6">
            <SignInForm />
            <SignUpForm />
          </div>
        )}
      </div>
    </div>
  );
}
`,

  // サインインフォーム
  'pages/index/SignInForm.tsx': `import { signIn } from "@/lib/better-auth/client";
import { useState } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";

export function SignInForm() {
    const [email, setEmail] = useState("");
    const [password, setPassword] = useState("");
    const [error, setError] = useState("");
    const [loading, setLoading] = useState(false);

    const handleSubmit = async (e: React.FormEvent) => {
        e.preventDefault();
        setError("");
        setLoading(true);

        const result = await signIn.email({
            email,
            password,
        });

        setLoading(false);

        if (result.error) {
            setError(result.error.message ?? "ログインに失敗しました");
            return;
        }

        // ログイン成功時はページをリロードしてセッションを反映
        window.location.reload();
    };

    return (
        <Card className="w-full max-w-md">
            <CardHeader>
                <CardTitle>サインイン</CardTitle>
                <CardDescription>アカウントにサインインしてください</CardDescription>
            </CardHeader>
            <CardContent>
                <form onSubmit={handleSubmit} className="space-y-4">
                    {error && (
                        <div className="p-3 text-sm text-red-600 bg-red-50 rounded-md">
                            {error}
                        </div>
                    )}

                    <div className="space-y-2">
                        <Label htmlFor="signin-email">メールアドレス</Label>
                        <Input
                            id="signin-email"
                            type="email"
                            value={email}
                            onChange={(e) => setEmail(e.target.value)}
                            placeholder="example@email.com"
                            required
                        />
                    </div>

                    <div className="space-y-2">
                        <Label htmlFor="signin-password">パスワード</Label>
                        <Input
                            id="signin-password"
                            type="password"
                            value={password}
                            onChange={(e) => setPassword(e.target.value)}
                            placeholder="パスワード"
                            required
                        />
                    </div>

                    <Button type="submit" className="w-full" disabled={loading}>
                        {loading ? "サインイン中..." : "サインイン"}
                    </Button>
                </form>
            </CardContent>
        </Card>
    );
}
`,

  // サインアップフォーム
  'pages/index/SignUpForm.tsx': `import { signUp } from "@/lib/better-auth/client";
import { useState } from "react";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";

export function SignUpForm() {
    const [name, setName] = useState("");
    const [email, setEmail] = useState("");
    const [password, setPassword] = useState("");
    const [error, setError] = useState("");
    const [loading, setLoading] = useState(false);
    const [success, setSuccess] = useState(false);

    const handleSubmit = async (e: React.FormEvent) => {
        e.preventDefault();
        setError("");
        setSuccess(false);
        setLoading(true);

        const result = await signUp.email({
            email,
            password,
            name,
        });

        setLoading(false);

        if (result.error) {
            setError(result.error.message ?? "新規登録に失敗しました");
            return;
        }

        setSuccess(true);
        setName("");
        setEmail("");
        setPassword("");
    };

    return (
        <Card className="w-full max-w-md">
            <CardHeader>
                <CardTitle>新規登録</CardTitle>
                <CardDescription>アカウントを作成してください</CardDescription>
            </CardHeader>
            <CardContent>
                <form onSubmit={handleSubmit} className="space-y-4">
                    {error && (
                        <div className="p-3 text-sm text-red-600 bg-red-50 rounded-md">
                            {error}
                        </div>
                    )}
                    {success && (
                        <div className="p-3 text-sm text-green-600 bg-green-50 rounded-md">
                            アカウントが作成されました。サインインしてください。
                        </div>
                    )}

                    <div className="space-y-2">
                        <Label htmlFor="signup-name">名前</Label>
                        <Input
                            id="signup-name"
                            type="text"
                            value={name}
                            onChange={(e) => setName(e.target.value)}
                            placeholder="山田太郎"
                            required
                        />
                    </div>

                    <div className="space-y-2">
                        <Label htmlFor="signup-email">メールアドレス</Label>
                        <Input
                            id="signup-email"
                            type="email"
                            value={email}
                            onChange={(e) => setEmail(e.target.value)}
                            placeholder="example@email.com"
                            required
                        />
                    </div>

                    <div className="space-y-2">
                        <Label htmlFor="signup-password">パスワード</Label>
                        <Input
                            id="signup-password"
                            type="password"
                            value={password}
                            onChange={(e) => setPassword(e.target.value)}
                            placeholder="8文字以上"
                            required
                            minLength={8}
                        />
                    </div>

                    <Button type="submit" className="w-full" disabled={loading}>
                        {loading ? "登録中..." : "新規登録"}
                    </Button>
                </form>
            </CardContent>
        </Card>
    );
}
`,

  // ユーザープロフィール
  'pages/index/UserProfile.tsx': `import { signOut, useSession } from "@/lib/better-auth/client";
import { Button } from "@/components/ui/button";
import { Card, CardContent, CardDescription, CardHeader, CardTitle } from "@/components/ui/card";
import { useState } from "react";

export function UserProfile() {
    const { data: session, isPending } = useSession();
    const [loading, setLoading] = useState(false);

    const handleSignOut = async () => {
        setLoading(true);
        await signOut();
        setLoading(false);
        // サインアウト後にページをリロード
        window.location.reload();
    };

    if (isPending) {
        return (
            <Card className="w-full max-w-md">
                <CardContent className="pt-6">
                    <p className="text-center text-muted-foreground">読み込み中...</p>
                </CardContent>
            </Card>
        );
    }

    if (!session) {
        return null;
    }

    return (
        <Card className="w-full max-w-md">
            <CardHeader>
                <CardTitle>ユーザープロフィール</CardTitle>
                <CardDescription>サインイン済み</CardDescription>
            </CardHeader>
            <CardContent className="space-y-4">
                <div className="space-y-2">
                    <div>
                        <p className="text-sm font-medium text-muted-foreground">名前</p>
                        <p className="text-base">{session.user.name}</p>
                    </div>
                    <div>
                        <p className="text-sm font-medium text-muted-foreground">メールアドレス</p>
                        <p className="text-base">{session.user.email}</p>
                    </div>
                </div>

                <Button
                    onClick={handleSignOut}
                    variant="destructive"
                    className="w-full"
                    disabled={loading}
                >
                    {loading ? "サインアウト中..." : "サインアウト"}
                </Button>
            </CardContent>
        </Card>
    );
}
`,

  // UI コンポーネント: Button
  'components/ui/button.tsx': `import * as React from "react"
import { Slot } from "@radix-ui/react-slot"
import { cva, type VariantProps } from "class-variance-authority"

import { cn } from "@/lib/utils"

const buttonVariants = cva(
  "inline-flex items-center justify-center gap-2 whitespace-nowrap rounded-md text-sm font-medium ring-offset-background transition-colors focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:pointer-events-none disabled:opacity-50 [&_svg]:pointer-events-none [&_svg]:size-4 [&_svg]:shrink-0",
  {
    variants: {
      variant: {
        default: "bg-primary text-primary-foreground hover:bg-primary/90",
        destructive:
          "bg-destructive text-destructive-foreground hover:bg-destructive/90",
        outline:
          "border border-input bg-background hover:bg-accent hover:text-accent-foreground",
        secondary:
          "bg-secondary text-secondary-foreground hover:bg-secondary/80",
        ghost: "hover:bg-accent hover:text-accent-foreground",
        link: "text-primary underline-offset-4 hover:underline",
      },
      size: {
        default: "h-10 px-4 py-2",
        sm: "h-9 rounded-md px-3",
        lg: "h-11 rounded-md px-8",
        icon: "h-10 w-10",
      },
    },
    defaultVariants: {
      variant: "default",
      size: "default",
    },
  }
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : "button"
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    )
  }
)
Button.displayName = "Button"

export { Button, buttonVariants }
`,

  // UI コンポーネント: Input
  'components/ui/input.tsx': `import * as React from "react"

import { cn } from "@/lib/utils"

const Input = React.forwardRef<HTMLInputElement, React.ComponentProps<"input">>(
  ({ className, type, ...props }, ref) => {
    return (
      <input
        type={type}
        className={cn(
          "flex h-10 w-full rounded-md border border-input bg-background px-3 py-2 text-base ring-offset-background file:border-0 file:bg-transparent file:text-sm file:font-medium file:text-foreground placeholder:text-muted-foreground focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-ring focus-visible:ring-offset-2 disabled:cursor-not-allowed disabled:opacity-50 md:text-sm",
          className
        )}
        ref={ref}
        {...props}
      />
    )
  }
)
Input.displayName = "Input"

export { Input }
`,

  // UI コンポーネント: Card
  'components/ui/card.tsx': `import * as React from "react"

import { cn } from "@/lib/utils"

const Card = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn(
      "rounded-lg border bg-card text-card-foreground shadow-sm",
      className
    )}
    {...props}
  />
))
Card.displayName = "Card"

const CardHeader = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("flex flex-col space-y-1.5 p-6", className)}
    {...props}
  />
))
CardHeader.displayName = "CardHeader"

const CardTitle = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn(
      "text-2xl font-semibold leading-none tracking-tight",
      className
    )}
    {...props}
  />
))
CardTitle.displayName = "CardTitle"

const CardDescription = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("text-sm text-muted-foreground", className)}
    {...props}
  />
))
CardDescription.displayName = "CardDescription"

const CardContent = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div ref={ref} className={cn("p-6 pt-0", className)} {...props} />
))
CardContent.displayName = "CardContent"

const CardFooter = React.forwardRef<
  HTMLDivElement,
  React.HTMLAttributes<HTMLDivElement>
>(({ className, ...props }, ref) => (
  <div
    ref={ref}
    className={cn("flex items-center p-6 pt-0", className)}
    {...props}
  />
))
CardFooter.displayName = "CardFooter"

export { Card, CardHeader, CardFooter, CardTitle, CardDescription, CardContent }
`,

  // UI コンポーネント: Label
  'components/ui/label.tsx': `"use client"

import * as React from "react"
import * as LabelPrimitive from "@radix-ui/react-label"
import { cva, type VariantProps } from "class-variance-authority"

import { cn } from "@/lib/utils"

const labelVariants = cva(
  "text-sm font-medium leading-none peer-disabled:cursor-not-allowed peer-disabled:opacity-70"
)

const Label = React.forwardRef<
  React.ElementRef<typeof LabelPrimitive.Root>,
  React.ComponentPropsWithoutRef<typeof LabelPrimitive.Root> &
    VariantProps<typeof labelVariants>
>(({ className, ...props }, ref) => (
  <LabelPrimitive.Root
    ref={ref}
    className={cn(labelVariants(), className)}
    {...props}
  />
))
Label.displayName = LabelPrimitive.Root.displayName

export { Label }
`,
};

// メイン処理
console.log('=== Neon + Better Auth セットアップスクリプト開始 ===\n');

try {
  let createdCount = 0;
  let overwrittenCount = 0;

  for (const [relativePath, content] of Object.entries(files)) {
    const fullPath = path.join(baseDir, relativePath);
    const exists = fs.existsSync(fullPath);

    writeFile(relativePath, content);

    if (exists) {
      overwrittenCount++;
    } else {
      createdCount++;
    }
  }

  console.log('\n=== セットアップ完了 ===');
  console.log(`新規作成: ${createdCount}ファイル`);
  console.log(`上書き: ${overwrittenCount}ファイル`);
  console.log(`合計: ${createdCount + overwrittenCount}ファイル\n`);

  console.log('次のステップ:');
  console.log('1. .env ファイルを作成・編集してください');
  console.log('2. package.json に必要なパッケージを手動で追加してください');
  console.log('3. wrangler.jsonc に必要に応じて hyperdrive を追加してください');
  console.log('4. `pnpm generate-types` を実行して worker-configuration.d.ts を生成してください');
  console.log('5. `pnpm auth:schema:generate` を実行して Better Auth スキーマを最新化してください');
  console.log('6. `pnpm drizzle:generate` を実行してマイグレーションを生成してください');
  console.log('7. `マイグレーションファイルをNeonに適用してください');

} catch (error) {
  console.error('\nエラーが発生しました:', error.message);
  process.exit(1);
}

```

:::

この二つが準備できれば後は`setup.sh`を実行すれば、自動的にセットアップとデプロイが進みます。  
途中で、NeonやCloudflareへのログインや権限付与の同意を求められますので、指示に従ってログインしてください。  
マイグレーションの生成も求められますので、`y`を入力して進めてください。  
データベースとのアクセスを高速化するために、HyperDriveの設定について聞かれますが、こちらも`y`を入力して進めてください。  
HyperDriveをアプリと紐づけるためにBinding名が聞かれますが、特にこだわりがなければデフォルト値で進めてください。 
以上で、セットアップとデプロイが完了します。
ここまでで、私の環境では大体5分くらいとなります。  
後は、Cloudflare WorkersのURLにアクセスすれば、以下のようなサインイン、サインアップができるアプリが動作しているはずです。  

色々動かして見て、動作を確認してみてください。

## アプリの構成図
ここでは、今回作成したアプリの構成図を示します。
以下のような関係でアプリは構成されています。
![構成図](/images/deploy-ba-neon-cw-donbei/architecture-diagram.jpg)  

API部分はHonoで構築されており、`/api/auth/*`に対するリクエストはBetter Authのハンドラにルーティングされます。
フロント部分はVikeで構築されており、Better Authのクライアントを利用してサインイン、サインアップ、サインアウトの機能を実装しています。
Better Authはユーザー情報など認証、認可に関係するデータを保存する必要があり、DBを必要とします。
ただ、直接DBにアクセスするのではなく、Drizzle ORMを介してDBにアクセスしています。
ここまでがアプリの話であり、Cloudflare Workers上で動作しています。
次にDBの話ですが、今回はNeonを使用しています。
NeonはPostgreSQL互換のサーバーレスデータベースです。  
Cloudflare D1でも良いのですが、認証、認可を扱うのならトランザクションや複雑なクエリが必要になることが多いと考え、PostgreSQLで動作するものを選びました。  
なのですが、今回については複雑な処理はないので、Cloudflare D1でも問題なく動作すると思います。  
以上が構成図です。  
構成図を構築するのに必要なコードやコマンドは全て`setup.sh`と`setup-neon-better-auth.cjs`に含まれています。  
それぞれのファイルの内容については、コードコメントに詳細を記載していますので、そちらを参照してください。  

## 得られた知見
今回、Neon + Better Auth + Cloudflare Workers + Vike + Drizzle ORMで構成されたアプリを構築、デプロイしてみて得られた知見を以下に示します。
### Better Authを試すのに必要な要素の把握
Better Authの存在自体は知っていたのですが、実際に試すにはどのような要素が必要なのか把握できていませんでした。  
今回実際にセットアップしてみて、Better Authを試すために必要な要素が把握できました。
具体的には以下の要素が必要であることが分かりました。
- DB (今回はNeon)
- ORM (今回はDrizzle ORM)
- サーバー (今回はCloudflare Workers + Hono)
- フロントエンド (今回はVike + React + shadcn/ui)

セットアップを億劫に思って試すのを避けていましたが、実際に行い構成を理解できたので、今後Better Authを試す際のハードルが下がりました。
### Vikeの便利さ
Vike自体は以前から存在を知っており、興味も持っていましたが実際に使ったことはありませんでした。  
今回初めてVikeを使ってみて、その便利さを実感しました。  
特に以下の点が便利だと感じました。
- プロジェクト作成時の選択肢の多さ
- Viteベースなため、ローカルの起動が高速
- SSR/SSG対応が簡単

特に恩恵を感じたのは、プロジェクト作成時の選択肢の多さです。
[プロジェクト作成画面](https://vike.dev/new)では、様々なフレームワークやライブラリを選択できます。
![プロジェクト作成画面](/images/deploy-ba-neon-cw-donbei/setup-vike.png)
選択し、画面に表示されたコマンドを実行すれば、選択した構成がある程度設定された状態でプロジェクトが作成されます。
今回のように、Cloudflare Workers + Hono + Drizzle ORM + Reactのような構成も、`pnpm create vike@latest --react --tailwindcss --shadcn-ui --hono --drizzle --cloudflare --eslint --prettier`を実行するだけで、ある程度設定された状態でプロジェクトが作成されました。  
これがかなり便利で、今回Better Authを試すためのハードルをかなり下げてくれました。
他の点もよく、ビルドの速さやSSR/SSG対応の容易さも、Vikeを使うメリットだと感じました。  
SSRは認証、認可機能搭載するには欲しい機能なので、VikeがSSRに対応しているのも将来的には大きなメリットになりえそうです。
### HyperDriveの必要性
Cloudflareには、DBとのやり取りを高速化するための仕組みとして[HyperDrive](https://www.cloudflare.com/ja-jp/developer-platform/products/hyperdrive/)があることも知れました。
今回使用したDBは、アメリカリージョンのNeonでした。  
そのため、通常通りにCloudflare Workersから接続すると、処理時間がかかりタイムアウトが起きました。  
HyperDriveを使えばコネクションをプールしたり、結果をキャッシュしたりできるので、タイムアウトを防げる可能性があるようです。
実際に、HyperDriveを有効化してみたところ、タイムアウトが発生せずに正常に動作しました。  
Cloudflare Workersはタイムアウトが結構早いので、リージョンを意識しないといけないこと。
そして、リージョンが離れている時はHyperDriveを使うことで、タイムアウトを防げる可能性があることが分かったのは大きな知見でした。

## おわりに
今回は、Neon + Better Auth + Cloudflare Workers + Vike + Drizzle ORMで構成された認証、認可機能を持つアプリをどん兵衛にお湯を入れてから、できあがるまでの間にデプロイできるスクリプトを紹介しました。  
そして、その中で得られた知見も共有しました。  
個人的にはBetter Authを試すためのハードルが下げられたと思うので、是非皆さんもどん兵衛用意する間にセットアップして、Better Authを試してみてください。
ここまで読んでいただき、ありがとうございました。

## 参考資料
NeonとDrizzle ORMの設定
https://zenn.dev/daidev/books/5b00c234f09a48/viewer/78f0bf
Better Authの導入資料
https://www.better-auth.com/docs/installation
Better Auth + Honoの例
https://hono.dev/examples/better-auth-on-cloudflare
Cloudflare Workersにおけるnode.jsの有効化
https://developers.cloudflare.com/workers/runtime-apis/nodejs/
HyperDriveの資料
https://zenn.dev/kameoncloud/articles/217263d6c87815
https://developers.cloudflare.com/hyperdrive/
https://www.cloudflare.com/ja-jp/developer-platform/products/hyperdrive/
NeonとCloudflare Workersの接続
https://developers.cloudflare.com/workers/databases/third-party-integrations/neon/
neon-cliインストール
https://neon.com/docs/reference/neon-cli#install
wslでpsqlコマンドを使う方法
https://qiita.com/nanbuwks/items/846cf3536a82a2798555