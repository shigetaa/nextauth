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

## カスタムログイン画面
カスタムログインページの構成コンポーネントとして、`MUI`デザインコンポーネントを使用しますので、使用できるように設定していきます。
```bash
npm install @mui/material @emotion/react @emotion/styled @mui/icons-material
```
マテリアルUIは、Roboto フォントを使用しますので以下の 外部CDNファイルを読み込みます。
```html
<link
  rel="stylesheet"
  href="https://fonts.googleapis.com/css?family=Roboto:300,400,500,700&display=swap"
/>
```
カスタムログインのフォーム情報を管理する為、`react-hook-form`を設定します。
```bash
npm install react-hook-form
```
サイト共通設定として、`MUI`を使用しますので`_app.tsx`を変更します。
```bash
vi pages/_app.tsx
```
```typescript
import type { AppProps } from 'next/app'
import { SessionProvider } from "next-auth/react"
import { Session } from 'next-auth'
import Head from 'next/head'
import CssBaseline from '@mui/material/CssBaseline'
import { createTheme, ThemeProvider } from '@mui/material/styles'
const theme = createTheme();

const App = ({ Component, pageProps }: AppProps<{ session: Session }>) => {
  return (
    <SessionProvider session={pageProps.session}>
      <Head>
        <meta name="viewport" content="initial-scale=1, width=device-width" />
        <link rel="stylesheet" href="https://fonts.googleapis.com/css?family=Roboto:300,400,500,700&display=swap" />
      </Head>
      <ThemeProvider theme={theme}>
        <CssBaseline />
        <Component {...pageProps} />
      </ThemeProvider>
    </SessionProvider>
  )
}

export default App
```
カスタムログインページを作成していきます。
```bash
vi pages/login.tsx
```
```typescript
import React, { useState } from "react"
import { Button, Container, Box, Avatar, Typography, TextField, Alert, FormControl, InputLabel, OutlinedInput, IconButton, InputAdornment } from "@mui/material"
import LockOutlinedIcon from '@mui/icons-material/LockOutlined'
import { Visibility, VisibilityOff } from '@mui/icons-material'
import { getCsrfToken, signIn } from "next-auth/react"
import { useRouter } from "next/router"
import { CtxOrReq } from "next-auth/client/_utils"
import { useForm } from "react-hook-form"

// POSTリクエスト（サインイン・サインアウトなど）に必要なCSRFトークンを返却する
export const getServerSideProps = async (context: CtxOrReq | undefined) => {
	return {
		props: {
			title: "login",
			csrfToken: await getCsrfToken(context),
		},
	};
};

interface IFormValues {
	email?: string;
	password?: string;
}
const Login = ({ csrfToken }: { csrfToken: string | undefined }) => {
	const router = useRouter()
	const [error, setError] = useState('')
	const [showPassword, setShowPassword] = useState(false)
	const handleClickShowPassword = () => {
		setShowPassword(!showPassword)
	}
	const handleMouseDownPassword = (event: React.MouseEvent<HTMLButtonElement>) => {
		event.preventDefault()
	}
	const { register, handleSubmit } = useForm<IFormValues>()
	const signInUser = async (data: IFormValues) => {
		await signIn<any>("credentials", {
			redirect: false,
			email: data.email,
			password: data.password,
			callbackUrl: `${window.location.origin}`,
		}).then((res) => {
			if (res?.error) {
				setError('Email , Password を正しく入力して下さい。')
			} else {
				router.push('/')
			}
		})
	}

	return (

		<Container component="main" maxWidth="xs">
			<Box
				sx={{
					marginTop: 8,
					display: 'flex',
					flexDirection: 'column',
					alignItems: 'center',
				}}
			>
				<Avatar sx={{ m: 1, bgcolor: 'secondary.main' }}>
					<LockOutlinedIcon />
				</Avatar>
				<Typography component="h1" variant="h5">Sign in</Typography>
				<Box component="form" noValidate sx={{ mt: 1 }} onSubmit={handleSubmit(signInUser)}>
					<input name="csrfToken" type="hidden" defaultValue={csrfToken} />
					{error && <Alert severity="error">{error}</Alert>}
					<TextField
						margin="normal"
						required
						fullWidth
						id="email"
						label="Email Address"
						autoComplete="email"
						autoFocus
						{...register('email')}
					/>
					<FormControl
						margin="normal"
						required
						fullWidth>
						<InputLabel htmlFor="outlined-adornment-password">Password</InputLabel>
						<OutlinedInput
							id="outlined-adornment-password"
							label="Password"
							type={showPassword ? 'text' : 'password'}
							autoComplete="current-password"
							{...register('password')}
							endAdornment={
								<InputAdornment position="end">
									<IconButton
										edge="end"
										aria-label="toggle password visibility"
										onClick={handleClickShowPassword}
										onMouseDown={handleMouseDownPassword}
									>
										{showPassword ? <VisibilityOff /> : <Visibility />}
									</IconButton>
								</InputAdornment>
							}
						/>
					</FormControl>
					<Button
						type="submit"
						fullWidth
						variant="contained"
						sx={{ mt: 3, mb: 2 }}
					>
						Sign In
					</Button>
				</Box>
			</Box>
		</Container>
	)
}

export default Login
```
カスタムログインページの指定するため、`[..nextauth].ts`ファイルに`pages`プロパティを追加してしていします。
```bash
vim pages/api/auth/[...nextauth].ts
```
`authOptions` オブジェクト内に追記
```typescript
	pages: {
		signIn: '/login',
		signOut: '/login'
	},
```
開発サーバーを起動
```bash
npm run dev
```
http://localhost:3000/login にアクセスすると無事ログインフォームが表示して、認証が出来たら完成です。