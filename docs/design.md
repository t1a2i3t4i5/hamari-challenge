# ハマりチャレンジ 設計書

パチンコ風の確率（1/319 など）で「当たるまで何回転ハマったか」を競う Web アプリ。
抽選はサーバ側で行い、分母ごとの自己ベストをリセットなしのランキングで表示する。

- 版: v0.1（2026-11-28）
- 状態: 設計中

---

## 1. 要件

### 1.1 機能

| # | 機能 | 内容 |
|---|------|------|
| F1 | 抽選 | 分母を選んで「回す」。サーバで当たるまでの回転数を算出して返す |
| F2 | 分母 | 2 / 99 / 199 / 319 / 349 / 399 の固定 6 種 |
| F3 | ユーザー | 匿名。初回に名前を入力し、サーバがトークンを発行する |
| F4 | 名前 | 1〜12 文字、重複可。表示は `名前#1234`（4 桁タグ）。後から変更可 |
| F5 | 自己ベスト | 分母ごとに最大ハマり回転数を保持。上回ったときだけ更新 |
| F6 | ランキング | 分母ごと、1 人 1 件、上位 100 件 + 自分の順位。リセットなし |
| F7 | 珍しさ表示 | その回転数以上ハマる確率 `(1 - 1/d)^(n-1)` を併記（例: 0.18%） |

### 1.2 非機能

- 無料枠（Cloudflare Workers Free / D1 Free）で運用できること
- 抽選結果をクライアントで改ざんできないこと
- ボットによる大量試行を、人が手で回すペースまで抑えること

### 1.3 スコープ外（v1 ではやらない）

- ログイン（Google 等）
- 端末間の記録引き継ぎ（引き継ぎコードは v2 候補）
- 日別・期間別ランキング
- ネイティブアプリ

---

## 2. 構成

```
[Browser] ── 静的ファイル ──> Cloudflare Pages
    │
    └─ fetch /api/* ───────> Cloudflare Workers (TypeScript)
                                 ├─ D1 (SQLite)            … ユーザー・自己ベスト
                                 ├─ Rate Limiting binding  … 回転の頻度制限
                                 └─ Turnstile 検証          … 登録時のボット判定
```

| 項目 | 採用 | 理由 |
|------|------|------|
| フロント | 素の HTML/CSS/JS（後で必要なら Vite） | 画面が 2 つだけで、フレームワーク不要 |
| API | Workers + Hono | ルーティングとミドルウェアが軽い |
| DB | D1 | 無料枠あり、SQL が使える |
| 開発 | wrangler（ローカルで D1 も動く） | |

---

## 3. データモデル（D1）

```sql
-- ユーザー
CREATE TABLE users (
  id            TEXT PRIMARY KEY,          -- UUID
  token_hash    TEXT NOT NULL UNIQUE,      -- SHA-256(token) の hex。トークン本体は保存しない
  name          TEXT NOT NULL,             -- 1〜12 文字
  tag           INTEGER NOT NULL,          -- 0〜9999、表示は #0042
  ip_hash       TEXT NOT NULL,             -- 登録元 IP のハッシュ（登録数制限用）
  created_at    INTEGER NOT NULL,          -- unix ms
  UNIQUE (name, tag)
);
CREATE INDEX idx_users_ip_created ON users (ip_hash, created_at);

-- 自己ベスト（ユーザー × 分母で 1 行）
CREATE TABLE best_records (
  user_id       TEXT NOT NULL REFERENCES users(id),
  denominator   INTEGER NOT NULL,          -- 2/99/199/319/349/399
  hamari        INTEGER NOT NULL,          -- 当たりを含む回転数
  achieved_at   INTEGER NOT NULL,          -- unix ms、同記録の順位決定に使う
  PRIMARY KEY (user_id, denominator)
);
CREATE INDEX idx_best_rank ON best_records (denominator, hamari DESC, achieved_at ASC);

-- 分母ごとの累計（任意。サイトの「総回転数」表示用）
CREATE TABLE stats (
  denominator   INTEGER PRIMARY KEY,
  total_spins   INTEGER NOT NULL DEFAULT 0, -- 全ユーザーの累計回転数
  total_plays   INTEGER NOT NULL DEFAULT 0  -- 「回す」が押された回数
);
```

### 自己ベスト更新（1 クエリ）

```sql
INSERT INTO best_records (user_id, denominator, hamari, achieved_at)
VALUES (?1, ?2, ?3, ?4)
ON CONFLICT (user_id, denominator) DO UPDATE
SET hamari = excluded.hamari, achieved_at = excluded.achieved_at
WHERE excluded.hamari > best_records.hamari;
```

### 自分の順位

```sql
SELECT COUNT(*) + 1 AS rank
FROM best_records
WHERE denominator = ?1
  AND (hamari > ?2 OR (hamari = ?2 AND achieved_at < ?3));
```

---

## 4. API

共通
- ベース: `/api`
- 認証: `Authorization: Bearer <token>`（登録以外すべて）
- エラー: `{ "error": { "code": "RATE_LIMITED", "message": "..." } }`

### 4.1 `POST /api/users` — 登録

リクエスト
```json
{ "name": "タイチ", "turnstileToken": "..." }
```
レスポンス `201`
```json
{ "userId": "uuid", "token": "base64url(32bytes)", "name": "タイチ", "tag": 3821 }
```
処理
1. Turnstile を検証
2. 名前を検証（長さ、NG ワード、前後の空白を除去、NFKC 正規化）
3. 同一 `ip_hash` の直近 1 時間の登録数が上限（仮: 5）なら `429`
4. タグを乱数で採番（`UNIQUE(name, tag)` に衝突したら再試行、最大 5 回）
5. トークンを生成し、ハッシュだけ保存。本体はこのレスポンスでしか返さない

### 4.2 `PATCH /api/users/me` — 名前変更

```json
{ "name": "新しい名前" }
```
→ `200 { "name": "...", "tag": 1234 }`（タグは振り直し）

### 4.3 `POST /api/spin` — 抽選

リクエスト
```json
{ "denominator": 319 }
```
レスポンス `200`
```json
{
  "denominator": 319,
  "hamari": 842,
  "probability": 0.0715,
  "isNewBest": true,
  "best": 842,
  "rank": 12
}
```
処理
1. トークン認証
2. 分母が許可リストにあるか確認、なければ `400`
3. レート制限（user_id と IP の両方）。超過したら `429`
4. 抽選（§5）
5. 自己ベスト更新（§3）
6. `stats` を加算
7. 更新があれば順位を計算して返す

### 4.4 `GET /api/ranking?d=319` — ランキング

レスポンス `200`
```json
{
  "denominator": 319,
  "top": [
    { "rank": 1, "name": "タイチ", "tag": 3821, "hamari": 2410, "probability": 0.00052, "achievedAt": 1764300000000 }
  ],
  "me": { "rank": 12, "hamari": 842 },
  "totalSpins": 1234567
}
```
- `me` はトークンがあるときだけ返す
- `top` は Cache API で 30 秒キャッシュする（D1 の読み取りを節約）

### 4.5 `GET /api/me` — 自分の情報

→ `200 { "name", "tag", "bests": { "2": 9, "319": 842, ... } }`

---

## 5. 抽選ロジック

- 乱数は `crypto.getRandomValues`（Workers 標準）
- 剰余による偏りを避けるため、棄却サンプリングを使う

```ts
function hitOnce(d: number): boolean {
  const limit = Math.floor(0x1_0000_0000 / d) * d; // d の倍数で切る
  const buf = new Uint32Array(1);
  for (;;) {
    crypto.getRandomValues(buf);
    if (buf[0] < limit) return buf[0] % d === 0;
  }
}

function spinUntilHit(d: number): number {
  let n = 0;
  do { n++; } while (!hitOnce(d));
  return n;
}
```

- 1/399 で数千回転のループになっても CPU 時間は数 ms で、Free の 10ms 制限に収まる見込み（要計測）
- 安全のため上限を設ける（仮: 100,000 回転。超えたら打ち切ってその値を返す。発生確率は事実上ゼロ）
- 1 回あたり 1 乱数で済ませたい場合は、幾何分布の逆関数法 `ceil(log(U) / log(1 - 1/d))` も使える。ただし浮動小数の扱いの説明が必要になるので、v1 は上の素直な方式でいく

---

## 6. 認証・トークン

- 登録時に 32 バイトの乱数トークンを生成し、base64url でクライアントへ返す
- サーバは `SHA-256(token)` だけを保存する
- クライアントは `localStorage` に `{ userId, token, name, tag }` を保存する
- ブラウザのデータを消すと復元できない（v1 は仕様として割り切り、画面に明記する）

---

## 7. 不正対策

| 脅威 | 対策 |
|------|------|
| 抽選結果の改ざん | 抽選はサーバのみ。クライアントは結果を演出するだけ |
| ボットによる高速連打 | Rate Limiting binding: user_id ごとに 10 秒 10 回、IP ごとに 10 秒 20 回（仮） |
| アカウント量産（試行回数を水増し） | 登録時に Turnstile、同一 IP で 1 時間 5 件まで |
| トークン漏えい | ハッシュ保存。HTTPS のみ |
| 不適切な名前 | NG ワードリスト + 長さ制限 + NFKC 正規化。運営用の手動非表示フラグは v2 |

注意: 匿名方式なので、複数端末や複数 IP を使った試行の水増しは完全には防げない。「人の手のペースに寄せる」までが目標。

---

## 8. 画面

### 8.1 プレイ画面（`/`）
- 分母のタブ: 2 / 99 / 199 / 319 / 349 / 399
- 巨大な回転数カウンター（サーバの結果をカウントアップ演出で表示）
- 「回す」ボタン（レスポンスと演出が終わるまで無効化）
- 結果: 回転数、珍しさ（%）、自己ベスト更新表示、現在の順位
- 初回は名前入力のモーダル（Turnstile ウィジェット付き）

### 8.2 ランキング画面（`/ranking`）
- 分母のタブ
- 上位 100 件: 順位 / 名前#タグ / ハマり / 珍しさ / 達成日
- 自分の行を強調。100 位圏外なら最下部に固定表示
- 分母ごとの累計回転数

---

## 9. 無料枠の見積もり（要確認）

1 回転あたりの D1 操作
- 書き込み: stats 1 行 + 自己ベスト更新時のみ 1 行
- 読み取り: 認証 1 行 + 更新時の順位計算（インデックス走査）

| 想定 | 回転/日 | D1 書き込み/日 |
|------|---------|----------------|
| 小規模 | 5,000 | 約 5,000〜6,000 |
| バズった日 | 100,000 | 約 100,000（Free 上限付近） |

- 書き込みの大半は `stats` なので、あふれそうなら stats をやめるか、Durable Object や KV に逃がす
- D1 Free は上限を超えるとエラーで止まる。課金はされない。必要になったら Workers Paid（月 $5〜）へ

---

## 10. 未決事項

- [ ] サービス名・ドメイン（`*.pages.dev` で始めるか、独自ドメインにするか）
- [ ] NG ワードリストの出どころ
- [ ] 抽選のたびに Turnstile を挟むか（v1 は登録時のみの予定）
- [ ] 演出（カウントアップ時間、効果音の有無）
- [ ] 利用規約・注意書き（「実在の遊技機とは無関係」「記録は端末に保存」）
- [ ] 引き継ぎコード（v2）

---

## 11. ディレクトリ構成（案）

```
hamari-challenge/
├─ docs/
│  └─ design.md        ← この文書
├─ web/                ← Pages（静的ファイル）
│  ├─ index.html
│  ├─ ranking.html
│  └─ app.js
├─ api/                ← Workers
│  ├─ src/
│  │  ├─ index.ts      ← Hono ルーティング
│  │  ├─ lottery.ts    ← 抽選
│  │  ├─ auth.ts
│  │  └─ db.ts
│  ├─ migrations/
│  │  └─ 0001_init.sql
│  └─ wrangler.toml
└─ README.md
```
