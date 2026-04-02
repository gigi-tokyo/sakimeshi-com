# GIGI-7246 DNS切替 + HTTPS有効化 手順書 v2

> 更新: 2026-03-27
> ステータス: Step 3 まで完了、Step 4 待機中（GitHub ドメイン検証のクールダウン待ち）

## 前提情報

| 項目 | 値 |
|------|-----|
| ドメインレジストラ | お名前.com（ネームサーバー: 01-04.dnsv.jp） |
| sakimeshi.com 更新期限 | 2026/04/16 |
| リポジトリ | https://github.com/gigi-tokyo/sakimeshi-com |
| プレビュー | http://sakimeshi.com（HTTP で表示中） |
| 方針 | apex 固定（`sakimeshi.com`） |

## 変更前の DNS レコード

| ホスト名 | TYPE | TTL | VALUE | 対応 |
|---------|------|-----|-------|------|
| sakimeshi.com | NS | 86400 | ns-*.awsdns-*.* (4件) | → 01-04.dnsv.jp に変更 |
| sakimeshi.com | A | 3600 | 13.112.187.226 | → GitHub Pages IPs に変更 |
| sakimeshi.com | MX | 300 | mailforward.dnsv.jp. | 変更不要 |

## 変更後の DNS レコード（現在の状態）

| ホスト名 | TYPE | TTL | VALUE | 状態 |
|---------|------|-----|-------|------|
| sakimeshi.com | NS | 86400 | 01-04.dnsv.jp | ✅ 設定済み |
| sakimeshi.com | A | 3600 | 185.199.108.153 | ✅ 設定済み |
| sakimeshi.com | A | 3600 | 185.199.109.153 | ✅ 設定済み |
| sakimeshi.com | A | 3600 | 185.199.110.153 | ✅ 設定済み |
| sakimeshi.com | A | 3600 | 185.199.111.153 | ✅ 設定済み |
| www.sakimeshi.com | CNAME | 3600 | gigi-tokyo.github.io | ✅ 設定済み |
| sakimeshi.com | MX | 300 | mailforward.dnsv.jp. | ✅ 変更なし |
| _github-pages-challenge-gigi-tokyo.sakimeshi.com | TXT | 3600 | 04bf2665c256ea0b947909e346e5b7 | ✅ 設定済み（Pages検証用） |
| _gh-gigi-tokyo-o.sakimeshi.com | TXT | 3600 | 22523614ff | ✅ 設定済み（Org検証用） |

---

## Step 1: 変更前の状態を記録 ✅ 完了

**作業:**
- お名前.com の DNS 設定画面をスクリーンショットで保存
- 現在のサイト表示（sakimeshi.com）をスクリーンショットで保存

**確認:**
- [x] DNS レコード一覧のスクショが手元にあること
- [x] 現在の sakimeshi.com の表示状態のスクショが手元にあること

---

## Step 2: CNAME ファイル作成 + DNS レコード変更 ✅ 完了

### 2-1. CNAME ファイル

リポジトリに `CNAME` ファイル（内容: `sakimeshi.com`）をコミット＆プッシュ済み。

**確認:**
- [x] https://github.com/gigi-tokyo/sakimeshi-com に CNAME ファイルが存在すること
- [x] GitHub Actions が成功していること

### 2-2. ネームサーバー変更（お名前.com）

AWS Route 53 → お名前.com のネームサーバーに変更済み。

**確認:**
- [x] `dig sakimeshi.com NS` → `01-04.dnsv.jp` が返ること

### 2-3. DNS レコード変更（お名前.com）

旧 A レコード削除 + GitHub Pages 用 A レコード 4件 + www CNAME を追加済み。

**確認:**
- [x] NS レコード（01-04.dnsv.jp）が変更されていないこと
- [x] MX レコード（mailforward.dnsv.jp.）が変更されていないこと
- [x] A レコードが 4件（185.199.108-111.153）に変更されていること
- [x] CNAME レコード（www → gigi-tokyo.github.io）が追加されていること
- [x] 全リゾルバ（8.8.8.8 / 1.1.1.1 / 9.9.9.9）で正しい値が返ること
- [x] `curl -sI http://sakimeshi.com` → `200 OK` / `Server: GitHub.com`

---

## Step 3: GitHub Pages Custom domain 設定 ✅ 完了（DNS check 未通過）

**作業:**
GitHub Pages Settings で Custom domain に `sakimeshi.com` を設定済み。

**現在の状態:**
- Custom domain: `sakimeshi.com` が設定されている
- DNS check: **unsuccessful**（NotServedByPagesError）
- `protected_domain_state`: **unverified**
- `http://sakimeshi.com` → サイトは正常に表示されている

**原因:**
GitHub 側のドメイン検証（`protected_domain_state`）が `unverified` のため DNS check が通らない。
Verify を連続実行したため、試行回数制限（"You've reached the maximum number of verification attempts"）に到達。

---

## Step 4: GitHub ドメイン検証 ⏳ 待機中

**状態:**
- Pages 検証 TXT: `_github-pages-challenge-gigi-tokyo.sakimeshi.com` → 全リゾルバで確認済み
- Org 検証 TXT: `_gh-gigi-tokyo-o.sakimeshi.com` → 設定済み
- Org domains: sakimeshi.com → Approved / Pending verification
- Verify 試行回数制限に到達 → **クールダウン待ち**

**作業（クールダウン後）:**

1. https://github.com/organizations/gigi-tokyo/settings/domains で sakimeshi.com の Verify を **1回だけ** 実行
2. 成功を確認

**確認:**
- [ ] Org domains で sakimeshi.com が「Verified」になること
- [ ] `gh api repos/gigi-tokyo/sakimeshi-com/pages` で `protected_domain_state` が `verified` になること

**失敗時:**
- GitHub Support へ証跡（dig結果 / API結果 / エラー画面スクショ）付きで問い合わせ
- https://support.github.com/

---

## Step 5: Pages DNS check + HTTPS 有効化

**前提:** Step 4 のドメイン検証が完了していること

**作業:**
1. https://github.com/gigi-tokyo/sakimeshi-com/settings/pages で「Check again」を押す
2. DNS check が successful になることを確認
3. 「Enforce HTTPS」にチェックを入れる

**確認:**
- [ ] DNS check が「successful」になること
- [ ] 「Enforce HTTPS」にチェックが入ること
- [ ] 数分待った後、`https://sakimeshi.com` にアクセスしてブラウザの鍵アイコンが表示されること
- [ ] 証明書の発行元が「Let's Encrypt」であること

---

## Step 6: サイト動作確認

**確認（すべてブラウザのシークレットモードで実施）:**
- [ ] `http://sakimeshi.com` → HTTPS にリダイレクトされてサイト表示
- [ ] `https://sakimeshi.com` → サイトが正常表示
- [ ] `http://www.sakimeshi.com` → サイトが正常表示
- [ ] `https://www.sakimeshi.com` → サイトが正常表示
- [ ] ヒーロー（KV 画像背景）が表示されている
- [ ] 受賞歴のリンクが正常に開く（グッドデザイン賞、ACC 等）
- [ ] メディア掲載のリンクが正常に開く（新R25、ITmedia、文化放送）
- [ ] ごちめしバナー（手紙後）→ gochi.online が開く
- [ ] フッターのごちめしボタン → gochi.online が開く
- [ ] 404 ページ（/xxxxx 等）→ 404.html が表示され、3秒後に `/` にリダイレクト

---

## Step 7: noindex/nofollow 解除

**前提:** Step 6 の動作確認が完了していること

**作業:**
index.html から以下の行を削除:
```html
<meta name="robots" content="noindex, nofollow">
```
コミット＆プッシュ。

**確認:**
- [ ] GitHub Actions が成功していること
- [ ] `https://sakimeshi.com` のソースに `noindex` が含まれていないこと
- [ ] `curl -s https://sakimeshi.com | grep noindex` → 結果なし

---

## Step 8: 完了報告

**確認:**
- [ ] Step 1〜7 の全チェックが完了していること
- [ ] GIGI-7246 の完了条件を満たしていること
- [ ] GIGI-6946（親課題）にコメントで報告

---

## ロールバック手順（問題発生時）

1. お名前.com で A レコードを `13.112.187.226` に戻す
2. ネームサーバーを AWS Route 53 に戻す（ns-537, ns-1060, ns-276, ns-1828）
3. www CNAME レコードを削除
4. GitHub Pages の Custom domain を空にする
5. CNAME ファイルを削除
6. DNS 浸透を待って旧サイトが表示されることを確認

---

## 注意事項

- NS / MX レコードは絶対に変更しないこと（メール転送が止まる）
- DNS 変更後、ブラウザキャッシュで旧サイトが表示される場合はシークレットモードで確認
- sakimeshi.com のドメイン更新期限は 2026/04/16（移行後も更新が必要）
- HTTPS 証明書は GitHub が Let's Encrypt で自動発行（手動対応不要）
- GitHub ドメイン検証の Verify は連打しないこと（試行回数制限あり）
