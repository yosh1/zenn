---
title: "Expo Go で Google サインイン + Firebase Auth を動かすまでにハマったこと"
emoji: "🔐"
type: "tech"
topics: ["expo", "reactnative", "firebase", "google", "oauth"]
published: true
---

Expo Go で Google サインインを実装しようとして、半日溶かした記録と解決策です。

結論から言うと、**iOS クライアント ID + PKCE + `WebBrowser.openAuthSessionAsync`** の組み合わせで動きます。`auth.expo.io` プロキシは使いません。

## なぜハマるのか

Google サインインの情報を調べると、だいたい3パターン出てきます。

1. `@react-native-google-signin/google-signin` — ネイティブモジュールが必要。**Expo Go では動かない**
2. `expo-auth-session` の Google Provider — SDK 49 で非推奨。`auth.expo.io` プロキシも不安定
3. `signInWithPopup` — Web 専用。**React Native では動かない**

2026年4月時点の Expo SDK 54 では、どれもそのままでは使えません。

## 動く構成

使うもの:
- `expo-web-browser` — ブラウザで OAuth 画面を開く
- `expo-crypto` — PKCE の code challenge を生成
- `firebase/auth` — Firebase の `signInWithCredential`
- GCP の **iOS 用 OAuth クライアント ID**

ポイントは iOS クライアント ID を使うことで **reversed client ID スキームのリダイレクト** が使えるところ。`WebBrowser.openAuthSessionAsync` は内部で `ASWebAuthenticationSession` を使っていて、Expo Go でもカスタムスキームへのリダイレクトをキャッチできます。

## 前提

- Expo SDK 54（managed workflow）
- Firebase プロジェクト作成済み
- Firebase Authentication で Google プロバイダーを有効化済み

## GCP の設定

Google Cloud Console で **iOS 用の OAuth クライアント ID** を作成します。

1. https://console.cloud.google.com/apis/credentials にアクセス
2. 「+ 認証情報を作成」→「OAuth クライアント ID」
3. アプリケーションの種類: **iOS**
4. バンドル ID: `com.yourapp.bundleid`（app.json の `ios.bundleIdentifier` と合わせる）
5. 作成されたクライアント ID をメモ

OAuth 同意画面の設定も忘れずに。ユーザーの種類を「外部」にして、テストユーザーを追加しておくとスムーズです。

## パッケージのインストール

```bash
npx expo install expo-web-browser expo-crypto firebase
```

## Firebase の初期化

```ts:services/firebase.ts
import { initializeApp, getApps, getApp } from "firebase/app";
import { initializeAuth, getReactNativePersistence } from "firebase/auth";
import AsyncStorage from "@react-native-async-storage/async-storage";

const firebaseConfig = {
  apiKey: process.env.EXPO_PUBLIC_FIREBASE_API_KEY,
  authDomain: process.env.EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN,
  projectId: process.env.EXPO_PUBLIC_FIREBASE_PROJECT_ID,
  storageBucket: process.env.EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET,
  messagingSenderId: process.env.EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID,
  appId: process.env.EXPO_PUBLIC_FIREBASE_APP_ID,
};

const app = getApps().length === 0 ? initializeApp(firebaseConfig) : getApp();

export const auth = initializeAuth(app, {
  persistence: getReactNativePersistence(AsyncStorage),
});
```

## Google サインインの実装

ここが肝心なところです。

```tsx:app/auth/login.tsx
import { useState } from "react";
import { Alert } from "react-native";
import * as WebBrowser from "expo-web-browser";
import * as Crypto from "expo-crypto";
import { GoogleAuthProvider, signInWithCredential } from "firebase/auth";
import { auth } from "@/services/firebase";

WebBrowser.maybeCompleteAuthSession();

const IOS_CLIENT_ID = process.env.EXPO_PUBLIC_GOOGLE_IOS_CLIENT_ID!;
const REVERSED_CLIENT_ID = IOS_CLIENT_ID.split(".").reverse().join(".");

const handleGoogleSignIn = async () => {
  // 1. PKCE の code_verifier と code_challenge を生成
  const codeVerifier =
    Math.random().toString(36).substring(2, 15) +
    Math.random().toString(36).substring(2, 15) +
    Math.random().toString(36).substring(2, 15);

  const digest = await Crypto.digestStringAsync(
    Crypto.CryptoDigestAlgorithm.SHA256,
    codeVerifier,
    { encoding: Crypto.CryptoEncoding.BASE64 },
  );
  const codeChallenge = digest
    .replace(/\+/g, "-")
    .replace(/\//g, "_")
    .replace(/=+$/, "");

  // 2. reversed client ID をリダイレクト URI として使う
  const redirectUri = `${REVERSED_CLIENT_ID}:/oauthredirect`;

  const authUrl =
    `https://accounts.google.com/o/oauth2/v2/auth` +
    `?client_id=${IOS_CLIENT_ID}` +
    `&redirect_uri=${encodeURIComponent(redirectUri)}` +
    `&response_type=code` +
    `&scope=${encodeURIComponent("openid profile email")}` +
    `&code_challenge=${codeChallenge}` +
    `&code_challenge_method=S256` +
    `&prompt=select_account`;

  // 3. ブラウザで認証画面を開く
  //    ASWebAuthenticationSession がリダイレクトをキャッチしてくれる
  const result = await WebBrowser.openAuthSessionAsync(authUrl, redirectUri);

  if (result.type === "success" && result.url) {
    const url = new URL(result.url);
    const code = url.searchParams.get("code");

    if (code) {
      // 4. authorization code を token に交換
      const tokenResponse = await fetch(
        "https://oauth2.googleapis.com/token",
        {
          method: "POST",
          headers: { "Content-Type": "application/x-www-form-urlencoded" },
          body: new URLSearchParams({
            client_id: IOS_CLIENT_ID,
            code,
            code_verifier: codeVerifier,
            grant_type: "authorization_code",
            redirect_uri: redirectUri,
          }).toString(),
        },
      );

      const tokens = await tokenResponse.json();

      // 5. id_token で Firebase にサインイン
      if (tokens.id_token) {
        const credential = GoogleAuthProvider.credential(tokens.id_token);
        await signInWithCredential(auth, credential);
      }
    }
  }
};
```

## なぜこれで動くのか

`WebBrowser.openAuthSessionAsync` は iOS だと `ASWebAuthenticationSession` を使います。この API は、指定したスキームへのリダイレクトを**アプリに URL Scheme が登録されていなくてもキャッチ**できます。

Expo Go のように URL Scheme を自由に追加できない環境でも、`reversed client ID` スキームへのリダイレクトを受け取れる。ここがミソです。

フローをまとめると:

1. `openAuthSessionAsync` で Google の認証画面を開く
2. ユーザーがログインすると `com.googleusercontent.apps.XXXXX:/oauthredirect?code=...` にリダイレクト
3. `ASWebAuthenticationSession` がこれをキャッチしてアプリに返す
4. authorization code を Google の token endpoint で id_token に交換
5. id_token を Firebase の `signInWithCredential` に渡してログイン完了

## ハマりどころまとめ

| やったこと | 結果 |
|---|---|
| `signInWithPopup` | React Native では undefined |
| `expo-auth-session` の Google Provider + `expoClientId` | `iosClientId` が優先されて Web client ID が使われない |
| `auth.expo.io` プロキシ | "Something went wrong" で失敗 |
| Web client ID + `exp://` リダイレクト | Google が `exp://` スキームを拒否 |
| Web client ID + `response_type=id_token` | プロキシがフラグメントを処理できない |
| **iOS client ID + PKCE + reversed client ID** | **動いた** |

## .env の設定

```env
EXPO_PUBLIC_FIREBASE_API_KEY=your-api-key
EXPO_PUBLIC_FIREBASE_AUTH_DOMAIN=your-project.firebaseapp.com
EXPO_PUBLIC_FIREBASE_PROJECT_ID=your-project-id
EXPO_PUBLIC_FIREBASE_STORAGE_BUCKET=your-project.firebasestorage.app
EXPO_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=your-sender-id
EXPO_PUBLIC_FIREBASE_APP_ID=your-app-id
EXPO_PUBLIC_GOOGLE_IOS_CLIENT_ID=your-ios-client-id.apps.googleusercontent.com
```

## 本番ビルドへの移行

Expo Go での開発が終わったら、本番では `@react-native-google-signin/google-signin` に切り替えるのが安定します。ネイティブの Google サインインダイアログが使えるので UX もよくなります。

この記事のアプローチは **Expo Go での開発フェーズを快適にする** ためのもの、という位置づけです。

## おわりに

ここにたどり着くまでに `invalid_request`、`redirect_uri_mismatch`、プロキシのエラーを何十回見たかわかりません。Expo Go + Google サインインは公式ドキュメントも情報が散らばっていて、正直かなりつらかった。

同じところで詰まっている方の参考になればうれしいです。
