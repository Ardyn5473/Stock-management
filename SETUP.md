# セットアップ完全ガイド ― AI 次フェーズ投資レジャー（Supabase同期版）

このアプリを「複数端末で同期・株価自動更新・基準超えの自動抽出・LINE通知」付きで動かすための手順書です。
構成は DAC備品アプリと同じ「Supabase（DB＋認証＋関数）＋ 静的ホスティング（index.html 1枚）」。ビルド不要です。

- 所要時間: 株価まで約15分／LINE通知まで追加で約15分
- 必要なもの: Supabaseアカウント、メールアドレス、（任意で）LINEアカウントとPC（CLI用）
- 難易度の山は2か所だけ: ①関数のデプロイ（CLI）と ②LINEのuserId取得。ここを丁寧に書きます。

---

## 全体像（先に把握）

```
[あなたのスマホ/PC] ──ログイン(メール)──▶ [Supabase: holdings / settings テーブル]
        ▲  同期                                   ▲ 価格・シグナル書き込み
        │                                         │
   index.html(静的サイト)                  [Edge Function: fetch-prices]
                                                   │  毎日(cron)or 手動↻
                                          ┌────────┴─────────┐
                                     [Stooq] 株価終値      [LINE Messaging API] 通知
```

やることは5つ（＋任意でLINE）:
1. Supabaseプロジェクトを作る
2. データベースを作る（SQLを1回貼る）
3. メールログインをONにする
4. 株価更新の関数をデプロイ＋毎日自動実行(cron)
5. index.htmlに接続情報を入れてホスティング
6.（任意）LINE通知を設定

---

## 同梱ファイルの役割

| ファイル | 役割 | いつ使う |
|---|---|---|
| `index.html` | アプリ本体 | ホスティングする |
| `schema.sql` | テーブル＋RLS（全部入り） | **新規**はこれを1回実行 |
| `migration_v2.sql` | 列追加（保有株数・LINE） | **旧版から更新**する人だけ |
| `supabase/functions/fetch-prices/index.ts` | 株価更新＋LINE通知の関数 | デプロイする |
| `SETUP.md` | この手順書 | 読む |

> すでに前バージョンを動かしている人は「STEP 2」で `schema.sql` の代わりに `migration_v2.sql` を実行し、`index.html` と関数を入れ替えるだけでOKです。

---

## STEP 1. Supabaseプロジェクトを作る

1. https://supabase.com にアクセスし、サインイン（GitHubアカウント等でOK）。
2. 「New project」をクリック。Organizationを選び、次を入力:
   - Name: 任意（例 `invest-ledger`）
   - Database Password: 強いパスワードを生成して**控える**（後で使う可能性あり）
   - Region: `Northeast Asia (Tokyo)` を推奨
3. 「Create new project」。プロビジョニングに1〜2分かかります。

### 接続情報を控える（重要）

左下の歯車 **Project Settings → API** を開き、次の2つをメモ:
- **Project URL**: `https://xxxxxxxx.supabase.co`
- **Project API keys → anon public**: `eyJhbGci...`（長い文字列）

> anonキーは公開して安全な鍵です（後述のRLSで各ユーザーのデータが守られます）。
> 同じ画面にある **service_role** キーは**秘密鍵**。index.htmlには絶対に入れません（STEP 4のcronとSTEP 6でのみサーバー側に使用）。

---

## STEP 2. データベースを作る

1. 左メニュー **SQL Editor** を開き、「New query」。
2. **新規の人**: `schema.sql` の中身を全部貼り付けて右下 **Run**。
   **旧版から更新の人**: 代わりに `migration_v2.sql` を貼って Run。
3. 「Success. No rows returned」と出ればOK。
4. 左メニュー **Table Editor** に `holdings` と `settings` の2テーブルができていることを確認。

### （推奨）リアルタイム同期を有効化

**Database → Replication**（または Publications）で `holdings` テーブルの Realtime をON。
→ 1台で編集すると他端末に即反映されます。OFFでも「画面を開き直す／フォーカスが戻る」と同期するので必須ではありません。

---

## STEP 3. メールログインを有効化

1. 左メニュー **Authentication → Sign In / Providers**（または Providers）。
2. **Email** が有効になっていることを確認（既定でON）。
3. このアプリは「メールに届く6桁コード」でログインする方式です。**リダイレクトURLの設定は不要**。
   同じメールでログインすれば、PC・スマホで同じ台帳が共有されます。

> 補足: 無料枠のメール送信はテスト用で送信数に上限があります。本格運用で届きにくい場合は、Authentication → Emails でSMTP（独自メール送信）を設定できますが、個人利用なら既定のままで十分です。

---

## STEP 4. 株価更新の関数をデプロイ＋自動実行

ここはPCのターミナルを使います。

### 4-1. Supabase CLI を入れる

```bash
npm install -g supabase
supabase --version   # 出ればOK
```

### 4-2. プロジェクトに接続

```bash
supabase login                       # ブラウザが開いて認証
supabase link --project-ref xxxxxxxx # ← Project URL の xxxxxxxx 部分
```

### 4-3. 関数をデプロイ

同梱の `supabase/functions/fetch-prices/` フォルダ構成のまま、プロジェクト直下で:

```bash
supabase functions deploy fetch-prices
```

- 関数内では `SUPABASE_URL` と `SUPABASE_SERVICE_ROLE_KEY` が**自動で使える**ので、環境変数の追加設定は不要です。
- これでアプリの「↻（株価を更新）」ボタンが動くようになります（手動更新）。

### 4-4. 毎日自動で更新（cron）

毎日張り付かない運用に合わせ、**平日の取引終了後に1回**だけ自動更新します。
SQL Editor で、`<project-ref>` と `<SERVICE_ROLE_KEY>` を置き換えて実行:

```sql
create extension if not exists pg_cron;
create extension if not exists pg_net;

-- 平日 16:10 JST（= 07:10 UTC）に株価更新＋シグナル通知を実行
select cron.schedule(
  'refresh-prices-eod',
  '10 7 * * 1-5',
  $$
  select net.http_post(
    url     := 'https://<project-ref>.functions.supabase.co/fetch-prices',
    headers := jsonb_build_object(
      'Content-Type','application/json',
      'Authorization','Bearer <SERVICE_ROLE_KEY>'
    )
  );
  $$
);
```

- `<SERVICE_ROLE_KEY>` は Project Settings → API の **service_role**。これはDB内のcronだけに置く秘密鍵で、index.htmlには入れません。
- 時刻調整は `'分 時 * * 1-5'`（UTC）で。停止は `select cron.unschedule('refresh-prices-eod');`、確認は `select * from cron.job;`。
- ダッシュボードに **Integrations → Cron** のUIがある場合は、そこから同じURLを叩くスケジュールをGUIで作ってもOKです。

---

## STEP 5. index.htmlに接続情報を入れてホスティング

### 5-1. 接続情報を記入

`index.html` を開き、上部のここを STEP 1 で控えた値に置き換え:

```js
const SUPABASE_URL = "https://YOUR-PROJECT.supabase.co";  // ← Project URL
const SUPABASE_ANON_KEY = "YOUR-ANON-KEY";                // ← anon public
```

（未設定のまま開くと、アプリが「セットアップ未完了」の案内を表示します。）

### 5-2. デプロイ（どれか1つ）

**A. Cloudflare Pages（DAC備品アプリと同じ）**
- Cloudflare ダッシュボード → Workers & Pages → Create → Pages → 「Upload assets」で `index.html` をアップ、または Git連携。
- ビルド設定は不要（フレームワークなし／出力はそのまま）。

**B. GitHub Pages**
- リポジトリに `index.html` を置く → Settings → Pages → Source を「Deploy from a branch」、ブランチ/ルートを選択 → 数十秒で公開URLが出ます。

**C. ローカルで動作確認だけ**
```bash
python3 -m http.server 8080
# ブラウザで http://localhost:8080/index.html
```

### 5-3. 初回ログイン

公開URLを開く → メールアドレス入力 → 届いた6桁コードを入力 → 台帳が開きます。
別の端末でも**同じメール**でログインすれば同じデータが見えます。

---

## STEP 6.（任意）LINE通知をセットアップ

シグナル（押し目買い・利確・損切りの到達）が出たらLINEに届くようにします。
**注意**: かつての「LINE Notify」は2025年3月31日に終了済みです。現行の **LINE Messaging API**（月200通まで無料）を使います。

### 6-1. LINE公式アカウント＆チャネルを作る
1. https://entry.line.biz/start/jp/ で**LINE公式アカウント**を無料作成。
2. https://developers.line.biz/ の **LINE Developers コンソール**にログイン → 対象チャネルで **Messaging API** を有効化。
3. 「Messaging API設定」タブ下部の **チャネルアクセストークン（長期）** を発行して控える。

### 6-2. 自分を友だち追加
作った公式アカウントのQR/IDから、通知を受け取りたい自分のLINEで**友だち追加**します（push通知は友だちにしか送れません）。

### 6-3. 自分のuserId（`U`で始まる）を取得 ← ここが一番つまずく所
push通知の宛先はメールやLINE IDではなく **userId** です。確実な取り方:

**方法A（コード不要・おすすめ）: 一時Webhookで受け取る**
1. https://webhook.site を開くと、自分専用の受信URL（`https://webhook.site/xxxx`）が出る。
2. LINE Developers コンソール → Messaging API設定 → **Webhook URL** にそのURLを設定し、**Webhookの利用をON**。
3. 友だち追加した公式アカウントに、自分のLINEから**何かメッセージを1通送信**。
4. webhook.site の受信データに `"source": { "userId": "Uxxxxxxxx..." }` が表示される。これをコピー。
5. 確認できたら、Webhook URLは消すか無効化してOK。

**方法B: 応答設定から確認**
LINE Official Account Manager の設定画面でフォロワーの情報から確認できる場合もありますが、確実なのは方法Aです。

### 6-4. トークンをサーバー側に登録（index.htmlには入れない）
```bash
supabase secrets set LINE_CHANNEL_ACCESS_TOKEN=（6-1で発行したトークン）
supabase functions deploy fetch-prices   # 反映のため再デプロイ
```

### 6-5. アプリで有効化＆テスト
1. アプリ右上の ⚙（配点・基準）→ 下部の **LINE通知**。
2. トグルをON、**送信先 LINE userId** に 6-3 の `U...` を貼り付け。
3. 「テスト通知を送る」→ LINEに届けば成功。「適用して保存」。

以後の動作: 株価更新（手動↻ or 日次cron）のたびに、**新たにシグナルが出た銘柄だけ**通知されます。同じ状態が続く間は重複送信しません（`last_signal`で管理）。

---

## 動作確認チェックリスト

- [ ] 公開URLでログインできる（6桁コードが届く）
- [ ] 「＋追加」で銘柄を保存でき、リロードしても残る
- [ ] 別端末で同じメールにログインすると同じ銘柄が見える
- [ ] コード（例 6981）と市場を入れた銘柄で「↻」を押すと現在値が入る（「自動更新」バッジが付く）
- [ ] スクリーナーで「基準クリア」「売買シグナル」が表示される
- [ ] 資産タブで保有株数・平均取得単価を入れた銘柄の評価損益が出る
- [ ] CSVボタンでファイルが落ちる
- [ ](任意)LINEのテスト通知が届く

---

## つまずきやすい点（トラブルシュート）

| 症状 | 原因/対処 |
|---|---|
| 「セットアップ未完了」と出る | index.html上部のURL/anonキーが未置換。STEP 5-1を確認 |
| ログインコードが届かない | 迷惑メール確認。無料枠の送信制限の可能性 → 時間を置く or SMTP設定 |
| 「↻」で価格が入らない | コード/市場が空、または関数未デプロイ。`supabase functions deploy fetch-prices` を確認 |
| 米国株が取れない | 銘柄の「市場」を「米国」にする（`.us`で取得）。日本株は数字コード＋東証 |
| 価格が古い | Stooqは終値・遅延ベース。cron時刻（UTC）を見直す |
| LINEテストが失敗 | トークン未設定/再デプロイ忘れ/userId誤り。`supabase secrets set`→再デプロイ→`U...`を再確認 |
| 通知が来すぎ/来ない | 通知は「状態が変わった時だけ」。同じシグナルが続く間は送りません（仕様） |
| 他端末に即反映されない | STEP 2のRealtimeをON（OFFでも開き直せば同期） |

---

## 日々の使い方（運用フロー）

1. 平日夕方、cronが自動で終値を取得 → 計画ラインに到達した銘柄はLINE通知。
2. 通知が来たら（または週次で）アプリの**スクリーナー**を開く＝「今アクションが要る銘柄」だけ並ぶ。
3. 押し目買い／利確は、証券会社の**指値・アラート**に登録しておく（毎日見なくてよい）。
4. 四半期決算ごとに、銘柄の仮説（前提）が生きているか見直す。崩れていれば淡々と外す。
5. **資産タブ**で評価損益・含み益率・構成比を確認。偏りすぎたテーマがあれば調整を検討。

---

## セキュリティの要点（必ず守る）

- **service_role キー**と **LINEチャネルトークン**は秘密鍵。index.htmlやGit公開リポジトリに**絶対に置かない**。サーバー側（cronのSQL内 / `supabase secrets`）だけに置く。
- index.htmlに入れてよいのは **Project URL** と **anon public** キーのみ（RLSで保護）。
- 株価はStooqの無料・終値・遅延データ。中長期・指値運用には十分ですが、短期売買には不向き。データ源を変えたい場合は関数の `quote()` を差し替え（Alpha Vantage / Twelve Data 等）。

---

## 補足: このツールについて

本ツールは分析を整理するための雛形で、投資助言ではありません。スコア・株価・各数値はご自身で最新の一次情報を確認のうえ入力してください。最終的な投資判断はご自身の責任で行ってください。
