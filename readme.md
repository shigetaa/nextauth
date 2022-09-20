# Next-Auth
```bash
npx create-next-app@latest nextauth --ts
cd nextauth
```
```bash
npm install next-auth @next-auth/prisma-adapter
npm install nodemailer
npm install sqlite3
npm install prisma @prisma/client
```
以下のコマンドを実行しPrismaを初期化します。
`.env`と`/prisma/prisma/schema.prisma`が作成されます。
```bash
npx prisma init
```
`.env` 内は以下のように記述します。
開発環境は`.env.local`にも記述します。
```bash
vi .env
```
```env
NEXTAUTH_URL=
NEXTAUTH_SECRET=
# Linux: `openssl rand -hex 32` or go to https://generate-secret.now.sh/32

AUTH0_ID=
AUTH0_SECRET=
AUTH0_ISSUER=

FACEBOOK_ID=
FACEBOOK_SECRET=

GITHUB_ID=
GITHUB_SECRET=

GOOGLE_ID=
GOOGLE_SECRET=

TWITTER_ID=
TWITTER_SECRET=

LINE_CLIENT_ID=
LINE_CLIENT_SECRET=

EMAIL_SERVER=smtp://username:password@smtp.example.com:587
EMAIL_FROM=NextAuth <noreply@example.com>

DATABASE_URL="file:./data.db"
```

`prisma/schema.prisma`内は以下のように記述します。
```bash
vi prisma/schema.prisma
```
```prisma
// This is your Prisma schema file,
// learn more about it in the docs: https://pris.ly/d/prisma-schema

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "sqlite"
  url      = env("DATABASE_URL")
}

model Account {
  id                 String    @id @default(cuid())
  userId             String
  providerType       String
  providerId         String
  providerAccountId  String
  refreshToken       String?
  accessToken        String?
  accessTokenExpires DateTime?
  createdAt          DateTime  @default(now())
  updatedAt          DateTime  @updatedAt
  user               User      @relation(fields: [userId], references: [id])

  @@unique([providerId, providerAccountId])
}

model Session {
  id           String   @id @default(cuid())
  userId       String
  expires      DateTime
  sessionToken String   @unique
  accessToken  String   @unique
  createdAt    DateTime @default(now())
  updatedAt    DateTime @updatedAt
  user         User     @relation(fields: [userId], references: [id])
}

model User {
  id            String    @id @default(cuid())
  name          String?
  email         String?   @unique
  emailVerified DateTime?
  image         String?
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  accounts      Account[]
  sessions      Session[]
}

model VerificationRequest {
  id         String   @id @default(cuid())
  identifier String
  token      String   @unique
  expires    DateTime
  createdAt  DateTime @default(now())
  updatedAt  DateTime @updatedAt

  @@unique([identifier, token])
}
```
スキーマファイルを作成したら、PrismaCLIを使いPrisma Clientを生成します。
```bash
npx prisma generate
```
今回記載したスキーマを元にデータベースを構成します。
```bash
npx prisma migrate dev
```


```bash
mkdir pages/api/auth
vi pages/api/auth/[...nextauth].ts
```
```typescript
import NextAuth, { NextAuthOptions } from "next-auth"
import CredentialsProvider from "next-auth/providers/credentials"
import GoogleProvider from "next-auth/providers/google"
import FacebookProvider from "next-auth/providers/facebook"
import GithubProvider from "next-auth/providers/github"
import TwitterProvider from "next-auth/providers/twitter"
import Auth0Provider from "next-auth/providers/auth0"
import EmailProvider from "next-auth/providers/email"
import LineProvider from "next-auth/providers/line"
import { PrismaAdapter } from "@next-auth/prisma-adapter"
import { PrismaClient } from "@prisma/client"

const prisma = new PrismaClient()
// For more information on each option (and a full list of options) go to
// https://next-auth.js.org/configuration/options
export const authOptions: NextAuthOptions = {
	// https://next-auth.js.org/configuration/providers/oauth
	providers: [
		CredentialsProvider({
			name: "User Acount",
			credentials: {
				username: { label: "Acount", type: "text", placeholder: "User Acount" },
				password: { label: "Password", type: "password" }
			},
			async authorize(credentials, req) {
				const user = { id: 1, name: "Akira", email: "akira@example.com" }
				if (user) {
					return user
				} else {
					return null
				}
			}
		}),
		EmailProvider({
			server: process.env.EMAIL_SERVER || "",
			from: process.env.EMAIL_FROM || "",
		}),
		// https://developers.line.biz/ja/docs/line-login/integrate-line-login/
		LineProvider({
			clientId: process.env.LINE_CLIENT_ID || "",
			clientSecret: process.env.LINE_CLIENT_SECRET || "",
		}),
		FacebookProvider({
			clientId: process.env.FACEBOOK_ID || "",
			clientSecret: process.env.FACEBOOK_SECRET || "",
		}),
		GithubProvider({
			clientId: process.env.GITHUB_ID || "",
			clientSecret: process.env.GITHUB_SECRET || "",
		}),
		GoogleProvider({
			clientId: process.env.GOOGLE_ID || "",
			clientSecret: process.env.GOOGLE_SECRET || "",
		}),
		TwitterProvider({
			clientId: process.env.TWITTER_ID || "",
			clientSecret: process.env.TWITTER_SECRET || "",
		}),
		Auth0Provider({
			clientId: process.env.AUTH0_ID || "",
			clientSecret: process.env.AUTH0_SECRET || "",
			issuer: process.env.AUTH0_ISSUER,
		}),
	],
	theme: {
		colorScheme: "light",
	},
	callbacks: {
		async jwt({ token }) {
			console.log('callbacks jwt')
			console.log(token)
			token.userRole = "admin"
			return token
		},
	},
	adapter: PrismaAdapter(prisma)
}

export default NextAuth(authOptions)
```

```bash
vi pages/_app.tsx
```
```typescript
import type { AppProps } from 'next/app'
import { SessionProvider } from "next-auth/react"
import { Session } from 'next-auth'

const App = ({ Component, pageProps }: AppProps<{ session: Session }>) => {
  return (
    <SessionProvider session={pageProps.session} refetchInterval={0}>
      <Component {...pageProps} />
    </SessionProvider>
  )
}

export default App

```

```bash
vi pages/index.tsx
```
```typescript
import { useSession, signIn, signOut } from "next-auth/react"

const IndexPage = () => {
  const { data: session } = useSession()
  console.log(session)
  return <>
    {!session && <>
      Not signed in <br />
      <button onClick={() => signIn()}>Sign in</button>
    </>}
    {session && <>
      Signed in as {session.user?.name}<br />
      <button onClick={() => signOut()}>Sign out</button>
    </>}
  </>
}

export default IndexPage
```