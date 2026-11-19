# 精算ノート（ふたり精算ノート）

2人で使う立て替え・割り勘の精算アプリです。カレンダーで支出を記録し、折半／個別負担の精算、現金・クレジットの内訳、ルーム共有ができます。データは Firebase Firestore（プロジェクト: `app-team-hiro`、コレクション: `warikan_rooms`）と同期されます。

AIアシスタントは **Firebase AI Logic 経由で Gemini** に接続し、アプリ内の会計データ（精算サマリー・今月の集計・明細一覧）を参照して回答します。

## セキュリティ設計（APIキーを公開しない）

- **Gemini APIキーはコードに一切含まれません。** ブラウザからは Firebase AI Logic の専用プロキシ経由でのみ Gemini を呼び出すため、HTMLをGitHubで公開してもキーは流出しません。
- HTML内にある Firebase の設定値（`apiKey` を含む）は、Firebase のウェブ用「プロジェクト識別子」であり、Gemini のキーとは別物です。Firebase 公式も公開前提の値としています。
- **悪用対策は Firebase App Check（reCAPTCHA Enterprise）** です。コンソールで AI Logic に対して App Check を「適用（Enforce）」すると、正規のアプリ（登録ドメイン）以外からの呼び出しは拒否され、第三者が勝手に Gemini の利用枠を使うことはできません。
- App Check のサイトキーは公開キーなので、コードに埋め込んでも問題ありません。
- 追加の防御として、Google Cloud / AI Studio 側で Gemini API の割り当て上限（クォータ）や利用上限を設定しておくことをおすすめします。

## セットアップ手順

### 1. Firebase AI Logic を有効にする

1. [Firebase コンソール](https://console.firebase.google.com/) でプロジェクト `app-team-hiro` を開く
2. 左メニューの「AI サービス」>「AI Logic」を開き、「使ってみる（Get started）」をクリック
3. Gemini API プロバイダの選択では **「Gemini Developer API」** を選ぶ（無料枠で開始できます）
4. 画面の案内に従って必要な API を有効化する（この過程で AI Logic への App Check 適用が自動で有効になります）

### 2. App Check（reCAPTCHA Enterprise）を登録する

1. [Google Cloud コンソール](https://console.cloud.google.com/) でプロジェクト `app-team-hiro` を開き、「Fraud Defense（reCAPTCHA Enterprise）」で **Web 用のスコアベースキー** を作成する
   - ドメインには、アプリを公開するドメインを登録します:
     - `app-team-hiro.web.app`
     - `app-team-hiro.firebaseapp.com`
     - GitHub Pages を使う場合は `<GitHubユーザー名>.github.io`（後から追加も可能）
   - 「チェックボックス チャレンジを使う」は **オフ** のまま
2. Firebase コンソールの「セキュリティ」>「App Check」>「アプリ」タブで、このウェブアプリを reCAPTCHA Enterprise プロバイダで登録し、作成したキーを入力する
3. 発行された **サイトキー** を `index.html` の `RECAPTCHA_ENTERPRISE_SITE_KEY` に貼り付ける
4. App Check の画面で **AI Logic に対して「適用（Enforce）」がオン** になっていることを確認する

※ サイトキーが空欄の間、AIアシスタントは「ローカル集計モード」で動作します（差額・今月のまとめ・LINE文面などの基本集計はそのまま使えます）。

※ ローカル開発（localhost）ではコードが自動でデバッグトークン方式に切り替わります。ブラウザのコンソールに表示されるデバッグトークンを、Firebase コンソールの App Check「デバッグトークンの管理」に登録してください。

### 3. GitHub で公開する（GitHub Pages）

1. GitHub で新しい **Public リポジトリ** を作成する
2. このフォルダのファイルを push する
3. リポジトリの Settings > Pages で、公開元を `main` ブランチ / ルートに設定する
4. 公開URLは `https://<GitHubユーザー名>.github.io/<リポジトリ名>/` になります
5. そのドメイン（`<GitHubユーザー名>.github.io`）を手順2の reCAPTCHA キーのドメインに追加するのを忘れずに

### 4.（任意）Firebase Hosting にデプロイする

```bash
npm install -g firebase-tools
firebase login
firebase deploy --only hosting
```

公開URLは `https://app-team-hiro.web.app/` です（`.firebaserc` にプロジェクト設定済み）。

## 使い方（AIアシスタント）

- 画面右下などにあるAIアシスタントボタンからチャットを開きます
- チャット上部のステータス表示で、Gemini連携モードかローカル集計モードかを確認できます
- 質問のたびに、最新の精算サマリー・今月の集計・明細一覧（新しい順・最大200件）と直近の会話履歴が Gemini に送られます
- 「LINE文面を作って」と聞くと、精算文面のコピーボタン付きで回答します

## ファイル構成

- `index.html` — アプリ本体（単一ファイル）
- `firebase.json` / `.firebaserc` — Firebase Hosting 用設定（プロジェクト: `app-team-hiro`）
- `README.md` — このファイル
