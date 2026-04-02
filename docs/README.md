# GIGI-6946 sakimeshi.com サイト閉鎖

## ステータス: 進行中（STUDIO で閉鎖案内作成中 / GitHub Pages 案を並行検討中）

## 概要

さきめし.com はコロナ禍において Gigi 株式会社が飲食店を応援するために始めたサービスの**特設ページ**。実際のチケット購入は ごちめし（gochimeshi.com）側で行われていた。現在はサービスとしての役割を終えており、サイトを閉鎖して跡地ページに切り替える。

- サイト: https://www.sakimeshi.com/
- 現在のホスティング: AWS（費用負担の実態は確認中）
- ドメイン: お名前.com（GMO）、更新期限 2026/4/16
- 各自治体版（hamamatsu.sakimeshi.com 等）は全て閉鎖済み
- sakimeshi.com 本体の閉鎖が残タスク

### 現在のサイト技術構成

| 項目 | 技術 |
|------|------|
| フレームワーク | **Nuxt.js**（Vue.js + SSR） |
| UI ライブラリ | **Vuetify** |
| ランタイム | Node.js |
| トラッキング | Google Tag Manager (GTM-TX8CMW, GTM-KMMN8P6) / Yahoo! Tag Manager |
| 構成 | SPA（Single Page Application）、スクラッチ開発 |

### ソースコードの状況

- **株式会社シロクマ**（東京都中央区、shirokuma-team.com）に外注して開発
- 現在、検索窓が正常に動作しない
- 当時の外注先に問い合わせたが「わからない」との回答
- 正常動作していた頃のソースコードは現在の開発パートナー **AI&T** が保持（別サービスを作る際に有用な可能性あり、保持しておく）
- ただし AI&T も修正方法はわからないとのこと
- → **修正不可能な状態のため、閉鎖して別サイトに切り替える判断**

---

## 子課題

| チケット | タイトル | ステータス | 担当 | 期限 |
|---------|---------|-----------|------|------|
| GIGI-6947 | サイト閉鎖案内 | 処理中 | 藤田 | 3/6 |
| GIGI-6948 | AWS アカウント解約 | 未対応 | 滝島 | — |

---

## GIGI-6947: サイト閉鎖案内

### 決定事項

- **ホスティング: STUDIO** に決定（藤田 3/3）
  - gochi-platform.com の有料プラン契約プロジェクトを流用（2026/12 まで有効）
  - 追加費用なし
  - 2026/12 以降は閉鎖案内ページ自体も不要との判断

### STUDIO 下書き

https://preview.studio.site/live/bXqzvvp4qD

### 検討経緯

| 日付 | 誰 | 内容 |
|------|-----|------|
| 2/18 | — | チケット作成 |
| 3/2 | 藤田 | 詳細検討: ペライチはサイト公開数制限で不可。STUDIO に決定。GCP 案はボツ |
| 3/2 | 谷脇 | ロリポップに HTML 置けば追加費用なしと提案 |
| 3/3 | 藤田 | STUDIO でもロリポップでも追加費用なし。STUDIO に決定 |
| 3/3 | 毛利 | GitHub Pages 案を提案（dev@gigi.tokyo で GitHub アカウント取得済み） |
| 3/3 | 藤田 | 「詳細を確認します」← ここで停止 |

### 不採用案

| 案 | 理由 |
|----|------|
| ペライチ | サイト公開数の上限に達している |
| GCP | コスト面で不採用 |
| ロリポップ | 追加費用なしだが STUDIO が選択された |

---

## GIGI-6948: AWS アカウント解約

- GIGI-6947（サイト閉鎖案内）完了後に実施
- AWS コスト削減が目的

### AWS インフラ構成（3/7 調査）

| コンポーネント | 用途 |
|-------------|------|
| EC2 x 2 台 | apex ドメイン（54.64.61.197, 54.150.235.165） |
| CloudFront | www 配信（CDN） |
| S3 | 静的ファイルホスティング（最終更新: 2022/12/25） |
| Route 53 | DNS（ネームサーバー: awsdns） |
| ACM | SSL 証明書（Amazon RSA 2048 M01） |

- AWS アカウント: `info+sakimeshi@shirokuma-team.com`（株式会社シロクマ名義）
- **MFA 設定済み** — ログインできず費用確認不可
- Gigi 本体の AWS アカウント（349850153971）とは**別アカウント**
- シロクマ側の契約に相乗りしている可能性あり（費用負担の実態が不明）

### Gigi 本体 AWS アカウントの sakimeshi 関連リソース

Gigi 本体アカウント上にも sakimeshi 関連サービスが GOCHI 共有インフラ上で稼働:

| リソース | Origin |
|---------|--------|
| `web-sakimeshi.app-gochimeshi.com` | gcms-v2-PROD-common-alb-pub |
| `sakimeshi-app-api.app-gochimeshi.com` | gcms-v2-PROD-common-alb-pub |

これらは GOCHI プラットフォームの共有 ALB 上で動作しており、sakimeshi.com 閉鎖とは別に対応が必要。

### 費用確認の進捗

- 3/7: 北さんへ Slack で MFA デバイス所在 + シロクマへの確認を依頼
  - https://gigitokyo.slack.com/archives/C06SK1HJ7UN/p1772879395033609?thread_ts=1772879239.408379&cid=C06SK1HJ7UN
- 3/9: **AI&T から AWS 通知に関する報告あり**（Slack #gochi-dev-team）
  - AI&T が AWS の支払い期限超過通知を直接受け取っている
  - **月額約 4,000 円**のコストが発生中
  - 現在さきめしサイトは利用していないため影響なしとの AI&T 見解
  - 関連チケット: [GIGI-4416](https://gigi-tokyo.backlog.com/view/GIGI-4416)（さきめしサイト閉鎖タイミングの判断）
  - https://gigitokyo.slack.com/archives/C03VBKSGYTG/p1765851534998779
- 3/9: **AI&T 回答あり — 3点とも解決**
  1. **ログイン: 可能** — `info+sakimeshi@shirokuma-team.com` ではなく、AI&T 開発者自身のメールアドレスでログイン。MFA も適用済み
  2. **リソース停止・解約: 権限あり** — DNS 切替後に EC2 / CloudFront / S3 / Route 53 等を停止・削除できる
  3. **費用: Gigi が支払い** — 添付ファイルで内訳共有あり（要確認）
  - https://gigitokyo.slack.com/archives/C03VBKSGYTG/p1773042204802419

---

## 毛利の構想: GitHub Pages で Gigi の実績アピールサイト（個人案、チーム未共有）

STUDIO の閉鎖案内ではなく、**GitHub Pages** で「さきめし」の跡地として Gigi の社会貢献を記録するサイトを構築したい。

### サイト構成（mockup.html 作成済み）

| セクション | 内容 |
|-----------|------|
| ヒーロー | メインメッセージ + KPI 4つ（カウントアップアニメーション） |
| ストーリー | さきめしの歩み（タイムライン形式） |
| 成果ダッシュボード | 月別支援金額推移 + 都道府県別店舗数（Chart.js） |
| 受賞歴 | 7件の受賞をカード表示 |
| メディア掲載 | カード形式 + カテゴリフィルタ（TV/新聞/Web/ラジオ） |
| 連携パートナー | 自治体・企業9社グリッド |
| 動画 | YouTube 埋め込み |
| フッター | サービス終了案内 + Gigi/ごちめし リンク |

### 技術選定

- HTML/CSS/JS のみ（フレームワーク・ビルドツール不要）
- Tailwind CSS + Chart.js + CountUp.js（全て CDN）
- さきめしブランドカラー: 濃紺 `#1f3657` / オレンジ `#f5a74c` / シアン `#37a89f`

### ダッシュボードに必要な数値（DB から取得予定）

**メイン KPI**: 総支援金額 / 支援された店舗数 / 支援者数 / 支援回数
**サービス規模**: 登録店舗数 / 展開自治体数 / 運営期間 / 登録ユーザー数
**ピーク**: 月間最大支援金額 / 1日最大支援件数 / 緊急事態宣言期間中の支援額
**地域別**: 都道府県別 TOP10 / ジャンル別比率 / 平均支援金額

### GitHub Pages のメリット

| 項目 | STUDIO 案 | GitHub Pages 案 |
|------|----------|-----------------|
| 費用 | 無料（既存プラン流用） | 無料 |
| 期限 | **2026/12 まで** | **無期限** |
| 内容 | 閉鎖案内テキストのみ | ダッシュボード + メディア + 動画 |
| 価値 | 「閉鎖しました」の告知 | **Gigi の社会貢献の恒久的な記録** |

---

## さきめしの受賞歴・メディア掲載（3/7 リサーチ）

### 受賞歴

| 受賞日 | 賞名 | 主催 | ソース |
|--------|------|------|--------|
| 2020/07 | 日本ギフト大賞 2020 緊急特別賞「飲食店応援賞」 | 日本ギフト大賞 | [PR TIMES](https://prtimes.jp/main/html/rd/p/000000059.000045433.html) |
| 2020/10 | グッドデザイン賞 2020「グッドデザイン・ベスト100」 | 日本デザイン振興会 | [Good Design Award](https://www.g-mark.org/award/describe/51113) / [note](https://note.com/gochimeshi/n/n1981c340686e) |
| 2020/10/30 | グッドフォーカス賞［新ビジネスデザイン］| 経済産業省 大臣官房 商務・サービス審議官賞 | [PR TIMES](https://prtimes.jp/main/html/rd/p/000000085.000045433.html) / [西日本新聞](https://www.nishinippon.co.jp/item/o/659687/) |
| 2020 | ACC 銅賞 | ACC | [Wikipedia 今井了介](https://ja.wikipedia.org/wiki/%E4%BB%8A%E4%BA%95%E4%BA%86%E4%BB%8B) |
| 2021/04 | Forbes 30 Under 30 Asia 2021（今井了介） | Forbes | [Gigi プレスリリース](https://www.gigi.tokyo/press-release/) |
| 2023/03 | ウェルビーイングアワード 2023 | — | [Gigi プレスリリース](https://www.gigi.tokyo/press-release/) |

### テレビ放送: 50番組以上（公式サイト記載）

確認済み: グッとラック！(TBS) / ミント！(MBS) / 人生最高のレストラン(TBS) / Live News it α(フジ) / あさイチ(NHK)

### 新聞: 日経・朝日・東京・西日本新聞 等

### Web: ITmedia / 新R25 / Foodist Media / MdN 等

### 自治体・企業連携: 神戸市 / 浜松市 / 生駒市 / 茨城県境町 / 大阪ガス / 西日本新聞 / 西部ガス / トヨタファイナンス / サントリー

→ 詳細は `PLAN.md` を参照

---

## ファイル構成

```
GIGI-6946-sakimeshiサイト閉鎖/          # ← 本ディレクトリ（計画・調査ドキュメント）
├── README.md          # 本ファイル（現況と全体整理）
├── PLAN.md            # GitHub Pages サイト構築計画（メディアリサーチ結果含む）
├── requirements.md    # デザイン要件（ユーザーフィードバック記録）
├── デザインメモ_ノスタルジー演出.md  # CSS/JS 演出効果の候補
└── mockup.html        # デザインモックアップ（ブラウザで開いて確認可能）
```

### GitHub Pages リポジトリ（実装）

```
~/dev/github.com/gigi-tokyo/sakimeshi-com/   # ← 実装はこちら
```

- GitHub（gigi-tokyo org）に push → GitHub Pages で公開
- mockup.html をベースに実装を進める

---

## 次のアクション

### ブロッカー
- [x] ~~北さんからの回答待ち（AWS MFA / シロクマへの確認）~~ → AI&T が直接ログイン可能
- [x] ~~AWS 費用の実態確認~~ → Gigi が支払い、月額約 4,000 円

### GitHub Pages 案（毛利）
- [ ] DB からダッシュボード数値を取得
- [ ] 社内広報にテレビ放送全50番組リストを確認
- [ ] YouTube の代表的な動画を特定
- [ ] チームに GitHub Pages 案を正式提案するか判断
- [ ] **AWS 解約前に DB データを必ず抽出すること**
- [ ] `~/dev/github.com/gigi-tokyo/sakimeshi-com/` にサイト実装

### 閉鎖作業（AI&T に依頼可能）
- [ ] DNS 切り替え（sakimeshi.com → GitHub Pages）
- [ ] AWS リソース停止・解約（AI&T が権限あり）
- [ ] Gigi 本体 AWS の sakimeshi 関連リソース（web-sakimeshi / sakimeshi-app-api）の扱いを確認
