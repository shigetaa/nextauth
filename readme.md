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
npm install bcrypt
npm install -D @types/bcrypt
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
  crypted_password String?
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
テストユーザーをデータベースに登録する`Seed`ファイルを実装
```bash
vi prisma/seed.ts
```
```typescript
import { Prisma, PrismaClient } from '@prisma/client'
import bcrypt from 'bcrypt'

const prisma = new PrismaClient()

const main = async () => {
	const saltRounds = 10
	const password = "test"
	const hashedPassword = await bcrypt.hash(password, saltRounds)

	const testUser = await prisma.user.upsert({
		where: { email: 'akira@example.com' },
		update: {},
		create: {
			email: 'akira@example.com',
			name: 'akira',
			cryptedPassword: hashedPassword
		},
	})
	console.log(testUser)
}

main()
	.catch((e) => {
		console.error(e)
		process.exit(1)
	}).finally(async () => {
		await prisma.$disconnect()
	})
```
実行コマンドを`package.json`に設定
```bash
vi package.json
```
以下を追記する。
```json
  "prisma": {
    "seed": "npx ts-node --compiler-options {\"module\":\"CommonJS\"} prisma/seed.ts"
  },
```
Seed を実行して、テストデータを登録する
```bash
npx prisma db seed
```
データベースに登録されているかブラウザで確認する為
以下のコマンドで **Prisma** のGUIを起動して、テストデータを確認する。
```bash
npx prisma studio
```
認証処理を実装していく
```bash
mkdir pages/api/auth
vi pages/api/auth/[...nextauth].ts
```
```typescript
import NextAuth, { NextAuthOptions } from "next-auth"
import CredentialsProvider from "next-auth/providers/credentials"
import { PrismaAdapter } from "@next-auth/prisma-adapter"
import { PrismaClient } from "@prisma/client"
import bcrypt from 'bcrypt'

const prisma = new PrismaClient()
export const authOptions: NextAuthOptions = {
	providers: [
		CredentialsProvider({
			name: "User Acount",
			credentials: {
				email: { label: "Email", type: "text", placeholder: "user@example.com" },
				password: { label: "Password", type: "password" }
			},
			async authorize(credentials, req) {
				const { email, password } = credentials as { email: string; password: string }
				const user = await prisma.user.findFirst({
					where: { email: email }
				})
				if (!user) {
					console.log('not email')
					return null
				}
				const isPassword = await bcrypt.compare(password, user.cryptedPassword || '')
				if (!isPassword) {
					console.log('not password')
					return null
				}
				return user
			}
		}),
	],
	theme: {
		colorScheme: "light",
	},
	session: {
		strategy: 'jwt'
	},
	secret: process.env.NEXTAUTH_SECRET,
	adapter: PrismaAdapter(prisma)
}

export default NextAuth(authOptions)
```
SessionProvider を実装する。
```bash
vi pages/_app.tsx
```
```typescript
import type { AppProps } from 'next/app'
import { SessionProvider } from "next-auth/react"
import { Session } from 'next-auth'

const App = ({ Component, pageProps }: AppProps<{ session: Session }>) => {
  return (
    <SessionProvider session={pageProps.session}>
      <Component {...pageProps} />
    </SessionProvider>
  )
}

export default App

```
トップページを作成
```bash
vi pages/index.tsx
```
```typescript
import { useSession, signIn, signOut } from "next-auth/react"

const IndexPage = () => {
  const { data: session } = useSession()
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
開発サーバを起動してブラウザで認証確認
```bash
npm run dev
```
ブラウザーで http://localhost:3000 にアクセスして認証出来るか確認する。