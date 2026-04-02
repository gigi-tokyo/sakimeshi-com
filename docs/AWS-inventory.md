# AWS リソースインベントリ — sakimeshi アカウント

> 調査日: 2026-03-27
> アカウントID: 682138580642
> リージョン: ap-northeast-1（東京）/ us-east-1（バージニア）
> CLI プロファイル: `--profile sakimeshi`

## サマリ

| サービス | リソース数 | 費用影響 | 備考 |
|---------|----------|---------|------|
| ALB | 1 | **高** | 月額費用の主要因（推定） |
| Elastic IP | 2 | **高** | 未関連付け（~1,080円/月） |
| CloudFront | 2 | 中 | www.sakimeshi.com / sakai.sakimeshi.com |
| S3 | 5 | 低 | 静的サイトホスティング |
| Route 53 | 1 zone (52 records) | 低 | 現在はお名前.com に NS 移行済み |
| Lambda | 9 | 低 | CloudFront 連携 + API 用 |
| API Gateway | 1 | 低 | Sakimeshi-API |
| DynamoDB | 3 | 低 | マスタ/モックデータ |
| ACM | 1 | 無料 | sakimeshi.com + *.sakimeshi.com |
| EC2 | 0 | - | インスタンスなし |
| RDS | 0 | - | インスタンスなし |

---

## 詳細

### ALB（Application Load Balancer）

| 名前 | DNS | 状態 | AZ |
|------|-----|------|-----|
| sakimeshi-alb-redirect | sakimeshi-alb-redirect-1676421597.ap-northeast-1.elb.amazonaws.com | active | ap-northeast-1a, 1c |

- SG: `sg-058f2a2027f11bc52`（sakimeshi-alb）
- 作成日: 2020-05-08
- **月額費用の主要因と推定**

---

### CloudFront

| ID | ドメイン | Alias | Origin | 状態 |
|----|---------|-------|--------|------|
| E2E7WQ9K5FXKGX | d17dfoily3kssy.cloudfront.net | www.sakimeshi.com | S3: sakimeshi-prd | Deployed |
| *(sakai用)* | - | sakai.sakimeshi.com | S3: sakai.sakimeshi.com | Deployed |

- Lambda@Edge 連携: `redirectToTrailingSlash`, `OriginRequest`, `BasicAuthentication`
- ViewerProtocolPolicy: redirect-to-https

---

### S3 バケット

| バケット名 | 用途（推定） |
|-----------|------------|
| sakimeshi-prd | 本番サイト（CloudFront Origin） |
| sakimeshi-stg | ステージング |
| sakai.sakimeshi.com | 堺市さきめしサイト |
| file.sakimeshi-stg | ステージング用ファイル |
| cf-accesslog-sakimeshi | CloudFront アクセスログ |

---

### Route 53

- Hosted Zone: `sakimeshi.com.`（ID: Z06068751OUWFN606WMHZ）
- レコード数: 52

**注意:** ネームサーバーはお名前.com（01-04.dnsv.jp）に移行済みのため、このゾーンは現在参照されていない。ただし自治体サブドメインのレコードが多数残っている。

#### 主要レコード

| レコード | Type | 値 |
|---------|------|-----|
| sakimeshi.com | A | Alias（CloudFront/ALB） |
| www.sakimeshi.com | A | Alias（CloudFront） |
| stg.sakimeshi.com | A | Alias |
| sakai.sakimeshi.com | A | Alias（CloudFront） |

#### 自治体サブドメイン（本番）

| サブドメイン | 向き先 |
|------------|--------|
| akishima.sakimeshi.com | ALB (lexus-alb) |
| daigas-energy.sakimeshi.com | ALB (lexus-alb) |
| fukuoka.sakimeshi.com | ALB (lexus-alb) |
| gasfes.sakimeshi.com | ALB (lexus-alb) |
| hamamatsu.sakimeshi.com | ALB (lexus-alb) |
| happy-eat.sakimeshi.com | ALB (lexus-alb) |
| kobe.sakimeshi.com | ALB (lexus-alb) |
| nagasaki.sakimeshi.com | ALB (lexus-alb) |
| sakura.sakimeshi.com | ALB (lexus-alb) |
| yotsukaido.sakimeshi.com | ALB (lexus-alb) |
| choisoko-okazaki.sakimeshi.com | Vercel |
| gasten.sakimeshi.com | Vercel |
| hiroshima.sakimeshi.com | Vercel |
| ikoma.sakimeshi.com | Vercel |
| narashino-ticket.sakimeshi.com | Vercel |
| okazaki.sakimeshi.com | Vercel |
| okz-rally-mawarin.sakimeshi.com | Vercel |
| saibufes.sakimeshi.com | Vercel |

#### 自治体サブドメイン（ステージング）

| サブドメイン | 向き先 |
|------------|--------|
| stg-akishima | staging ALB |
| stg-fukuoka | staging ALB |
| stg-gasfes | staging ALB |
| stg-hamamatsu | staging ALB |
| stg-happy-eat | staging ALB |
| stg-hiroshima | staging ALB |
| stg-ikoma | staging ALB |
| stg-kobe | staging ALB |
| stg-nagasaki | staging ALB |
| stg-sakura | staging ALB |
| stg-test | staging ALB |
| stg-yotsukaido | staging ALB |
| s-daigas-energy | staging ALB |

---

### Lambda

#### ap-northeast-1（東京）— API 用

| 関数名 | Runtime | 用途（推定） |
|--------|---------|------------|
| addMasterTable | Python 3.7 | マスタデータ投入 |
| addShopList | Python 3.7 | 店舗データ投入 |
| Sakimeshi_getMaster | Python 3.7 | マスタ取得 API |
| Sakimeshi_getShopList | Python 3.7 | 店舗一覧 API |
| Sakimeshi_getShopDetail | Python 3.7 | 店舗詳細 API |
| Sakimeshi_getTotalDonation | Python 3.7 | 寄付総額 API |

#### us-east-1（バージニア）— Lambda@Edge

| 関数名 | Runtime | 用途 |
|--------|---------|------|
| BasicAuthentication | Node.js 10.x | Basic 認証（stg 用） |
| redirectToTrailingSlash | Node.js 10.x | URL 末尾スラッシュ補完 |
| OriginRequest | Node.js 10.x | Origin リクエスト処理 |

---

### API Gateway

| 名前 | ID | エンドポイント | セキュリティ |
|------|-----|-------------|------------|
| Sakimeshi-API | roj28wpusl | REGIONAL | TLS 1.0 |

---

### DynamoDB

| テーブル名 | 用途（推定） |
|-----------|------------|
| MasterTable_Cities | 市区町村マスタ |
| MasterTable_Prefectures | 都道府県マスタ |
| SakimeshiMock_ShopData | 店舗モックデータ |

---

### ACM（Certificate Manager）

| ドメイン | ステータス | 有効期限 | リージョン |
|---------|----------|---------|----------|
| sakimeshi.com / *.sakimeshi.com | ISSUED | 2026-12-09 | us-east-1 |

---

### IAM ユーザー

| ユーザー名 | 作成日 | 用途（推定） |
|-----------|--------|------------|
| sakimeshi-admin | 2020-05-04 | 管理者 |
| sakimeshi-cli | 2020-09-02 | CLI 操作用 |
| sakimeshi-s3 | 2020-05-06 | S3 アクセス用 |
| yusuke.mori-gigi | 2026-03-27 | 今回作成 |

---

### Elastic IP

| IP | 関連付け | 備考 |
|----|---------|------|
| 35.79.110.222 | なし | 未関連付け（課金中: ~$3.6/月） |
| 54.92.9.11 | なし | 未関連付け（課金中: ~$3.6/月） |

- 未関連付けの EIP は 1つあたり約 $3.6/月（~540円）
- 2つ合計で **月額 ~1,080円**
- ALB に次ぐコスト要因

---

### VPC / Security Group

| リソース | ID | 備考 |
|---------|-----|------|
| VPC | vpc-cff6ffa8 | デフォルト VPC (172.31.0.0/16) |
| SG | sg-85c4b9f2 | default |
| SG | sg-0428cfaa76a9fdcd9 | launch-wizard-1 |
| SG | sg-058f2a2027f11bc52 | sakimeshi-alb |

---

## 停止時の優先順位

1. **ALB** — 費用の主要因。最初に停止すべき
2. **CloudFront** — 配信停止（Disable → Delete）
3. **Route 53 Hosted Zone** — NS はお名前.com に移行済みなので削除可能
4. **S3** — データバックアップ後に削除
5. **Lambda / API Gateway / DynamoDB** — 費用は低いが不要なら削除
6. **ACM** — 無料だが不要なら削除
7. **IAM ユーザー** — 最後にクリーンアップ
