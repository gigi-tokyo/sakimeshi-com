# AWS コスト削減 停止手順書

> ステータス: レビュー済み（実行可）
> 作成日: 2026-03-28
> アカウント: 682138580642（プロファイル: `sakimeshi`）
> リージョン: ap-northeast-1 / us-east-1
> 月額費用: 約 4,000 円
> 方針: **削除ではなく停止**（ALB のみ停止手段がないため削除）

## コスト影響分析

| リソース | 月額費用（推定） | 対応 | 備考 |
|---------|---------------|------|------|
| ALB | ~2,500円 | **削除**（停止手段なし） | 固定費。リダイレクト専用・TG なし |
| Elastic IP × 2 | ~1,080円 | **解放** | 未関連付けで課金中（~$3.6/月 × 2） |
| CloudFront × 3 | ~500円 | **無効化のみ** | Enabled=false で配信由来の課金を大幅削減 |
| Route 53 Hosted Zone | ~60円 | 放置 | $0.50/month（NS 移行済みで参照なし） |
| S3 (5 buckets) | ~100円 | 放置 | ストレージ料金のみ（ログ 54万件含む） |
| Lambda / API GW / DynamoDB | ~0円 | 放置 | リクエストなしなら無料枠内 |
| ACM | 0円 | 放置 | 無料 |

> **注意:** CloudFront 無効化後も S3 ログ保存費用や WAF 等の付帯コストが残る可能性あり。

## 安全ルール

1. **実行順序を守る**: バックアップ → Pre-check → 現状記録 → 停止/削除 → 確認
2. **出力は全て保存**: `aws-decommission-logs/` に tee で記録
3. **1リソースずつ実行**: バッチ実行しない
4. **各ステップで確認してから次に進む**

---

## Phase 0: 事前準備 + バックアップ

```bash
mkdir -p aws-decommission-logs

# 認証確認
aws --profile sakimeshi sts get-caller-identity \
  | tee aws-decommission-logs/00-identity.json
```

### 0-1. 構成バックアップ（Terraform import 用）

将来の復元や他アカウントへの移行に備え、全リソースの構成情報をエクスポートする。

```bash
# ALB
aws --profile sakimeshi --region ap-northeast-1 \
  elbv2 describe-load-balancers \
  | tee aws-decommission-logs/backup-alb.json

aws --profile sakimeshi --region ap-northeast-1 \
  elbv2 describe-listeners \
  --load-balancer-arn "arn:aws:elasticloadbalancing:ap-northeast-1:682138580642:loadbalancer/app/sakimeshi-alb-redirect/eea6b205510c8004" \
  | tee aws-decommission-logs/backup-alb-listeners.json

aws --profile sakimeshi --region ap-northeast-1 \
  elbv2 describe-target-groups \
  --load-balancer-arn "arn:aws:elasticloadbalancing:ap-northeast-1:682138580642:loadbalancer/app/sakimeshi-alb-redirect/eea6b205510c8004" \
  | tee aws-decommission-logs/backup-alb-target-groups.json

# CloudFront
for dist_id in E2E7WQ9K5FXKGX E3E9OXW587FLR E3R8I3203AG4ZI; do
  aws --profile sakimeshi --region us-east-1 \
    cloudfront get-distribution-config --id $dist_id \
    | tee aws-decommission-logs/backup-cf-${dist_id}.json
done

# Route 53
aws --profile sakimeshi route53 list-resource-record-sets \
  --hosted-zone-id Z06068751OUWFN606WMHZ \
  | tee aws-decommission-logs/backup-route53-records.json

# S3 バケット一覧
aws --profile sakimeshi s3api list-buckets \
  | tee aws-decommission-logs/backup-s3-buckets.json

# Lambda
aws --profile sakimeshi --region ap-northeast-1 lambda list-functions \
  | tee aws-decommission-logs/backup-lambda-apne1.json
aws --profile sakimeshi --region us-east-1 lambda list-functions \
  | tee aws-decommission-logs/backup-lambda-use1.json

# API Gateway
aws --profile sakimeshi --region ap-northeast-1 apigateway get-rest-apis \
  | tee aws-decommission-logs/backup-apigw.json

# DynamoDB
aws --profile sakimeshi --region ap-northeast-1 dynamodb list-tables \
  | tee aws-decommission-logs/backup-dynamodb.json

# ACM
aws --profile sakimeshi --region us-east-1 acm list-certificates \
  | tee aws-decommission-logs/backup-acm-use1.json
aws --profile sakimeshi --region ap-northeast-1 acm list-certificates \
  | tee aws-decommission-logs/backup-acm-apne1.json

# Security Groups
aws --profile sakimeshi --region ap-northeast-1 ec2 describe-security-groups \
  | tee aws-decommission-logs/backup-security-groups.json

# IAM
aws --profile sakimeshi iam list-users \
  | tee aws-decommission-logs/backup-iam-users.json
```

**確認:**
- [ ] Account が `682138580642` であること
- [ ] `aws-decommission-logs/backup-*.json` が全て保存されていること

---

## Phase 1: ALB 削除（最優先 — 月額 ~2,500円）

ALB はリダイレクト専用（ターゲットグループなし）。
- Port 80: HTTP → HTTPS リダイレクト
- Port 443: → www.sakimeshi.com へ 301 リダイレクト

DNS は既にお名前.com に移行済みで、この ALB にはトラフィックが来ない。
ALB には「停止」機能がないため削除が必要。

### 1-1. 現状記録

```bash
aws --profile sakimeshi --region ap-northeast-1 \
  elbv2 describe-load-balancers \
  --names sakimeshi-alb-redirect \
  | tee aws-decommission-logs/10-alb-describe.json

aws --profile sakimeshi --region ap-northeast-1 \
  elbv2 describe-listeners \
  --load-balancer-arn "arn:aws:elasticloadbalancing:ap-northeast-1:682138580642:loadbalancer/app/sakimeshi-alb-redirect/eea6b205510c8004" \
  | tee aws-decommission-logs/11-alb-listeners.json

aws --profile sakimeshi --region ap-northeast-1 \
  elbv2 describe-target-groups \
  --load-balancer-arn "arn:aws:elasticloadbalancing:ap-northeast-1:682138580642:loadbalancer/app/sakimeshi-alb-redirect/eea6b205510c8004" \
  | tee aws-decommission-logs/12-alb-target-groups.json
```

**確認:**
- [ ] ALB 情報が記録されていること
- [ ] リスナーが 2 つ（80, 443）であること
- [ ] ターゲットグループ情報が記録されていること（空でも記録）

### 1-2. ALB 削除

```bash
aws --profile sakimeshi --region ap-northeast-1 \
  elbv2 delete-load-balancer \
  --load-balancer-arn "arn:aws:elasticloadbalancing:ap-northeast-1:682138580642:loadbalancer/app/sakimeshi-alb-redirect/eea6b205510c8004" \
  2>&1 | tee aws-decommission-logs/13-alb-delete.json
```

### 1-3. 確認

```bash
# ALB が存在しないことを確認（LoadBalancerNotFound エラーが正常）
aws --profile sakimeshi --region ap-northeast-1 \
  elbv2 describe-load-balancers \
  --names sakimeshi-alb-redirect 2>&1 \
  | tee aws-decommission-logs/14-alb-verify.json

# sakimeshi.com が引き続き GitHub Pages で表示されることを確認
curl -sI http://sakimeshi.com | head -3
```

**確認:**
- [ ] `LoadBalancerNotFound` エラーが返ること（= 削除成功）
- [ ] `http://sakimeshi.com` が 200 OK を返すこと

---

## Phase 2: Elastic IP 解放（月額 ~1,080円）

未関連付けの Elastic IP が 2 つあり、ALB に次ぐコスト要因。
どちらもインスタンスに関連付けられていないため、即解放可能。

### 2-1. 現状記録

```bash
aws --profile sakimeshi --region ap-northeast-1 \
  ec2 describe-addresses \
  | tee aws-decommission-logs/15-eip-describe.json
```

**確認:**
- [ ] 2 つの EIP（35.79.110.222 / 54.92.9.11）が InstanceId=null であること

### 2-2. EIP 解放

```bash
# Allocation ID を取得して解放
for ip in 35.79.110.222 54.92.9.11; do
  ALLOC_ID=$(aws --profile sakimeshi --region ap-northeast-1 \
    ec2 describe-addresses \
    --public-ips $ip \
    --query 'Addresses[0].AllocationId' --output text)

  aws --profile sakimeshi --region ap-northeast-1 \
    ec2 release-address --allocation-id $ALLOC_ID \
    2>&1 | tee -a aws-decommission-logs/16-eip-release.json

  echo "Released: $ip ($ALLOC_ID)"
done
```

### 2-3. 確認

```bash
aws --profile sakimeshi --region ap-northeast-1 \
  ec2 describe-addresses \
  --query 'Addresses[].PublicIp' --output json
```

**確認:**
- [ ] EIP が空配列であること

---

## Phase 3: CloudFront 無効化（月額 ~500円）

3 つのディストリビューションを **無効化のみ**（削除しない）。
無効化により配信由来の課金を大幅に削減する。

| ID | Alias | Origin |
|----|-------|--------|
| E2E7WQ9K5FXKGX | www.sakimeshi.com | S3: sakimeshi-prd |
| E3E9OXW587FLR | stg.sakimeshi.com | S3: sakimeshi-stg |
| E3R8I3203AG4ZI | sakai.sakimeshi.com | S3: sakai.sakimeshi.com |

### 2-1. 現状記録

```bash
for dist_id in E2E7WQ9K5FXKGX E3E9OXW587FLR E3R8I3203AG4ZI; do
  aws --profile sakimeshi --region us-east-1 \
    cloudfront get-distribution-config --id $dist_id \
    | tee aws-decommission-logs/20-cf-${dist_id}-config.json
done
```

**確認:**
- [ ] 3 つの config ファイルが保存されていること

### 2-2. 無効化（1つずつ実行）

> 注意: 各ディストリビューションごとに一時ファイルを分離して誤操作を防止。

```bash
# ---- www.sakimeshi.com ----
DIST_ID="E2E7WQ9K5FXKGX"

aws --profile sakimeshi --region us-east-1 \
  cloudfront get-distribution-config --id $DIST_ID \
  > /tmp/cf-${DIST_ID}-config.json

ETAG=$(python3 -c "import json; print(json.load(open('/tmp/cf-${DIST_ID}-config.json'))['ETag'])")

python3 -c "
import json
data = json.load(open('/tmp/cf-${DIST_ID}-config.json'))
config = data['DistributionConfig']
config['Enabled'] = False
json.dump(config, open('/tmp/cf-${DIST_ID}-disabled.json', 'w'), indent=2)
"

aws --profile sakimeshi --region us-east-1 \
  cloudfront update-distribution \
  --id $DIST_ID \
  --if-match $ETAG \
  --distribution-config file:///tmp/cf-${DIST_ID}-disabled.json \
  | tee aws-decommission-logs/21-cf-${DIST_ID}-disabled.json
```

```bash
# ---- stg.sakimeshi.com ----
DIST_ID="E3E9OXW587FLR"

aws --profile sakimeshi --region us-east-1 \
  cloudfront get-distribution-config --id $DIST_ID \
  > /tmp/cf-${DIST_ID}-config.json

ETAG=$(python3 -c "import json; print(json.load(open('/tmp/cf-${DIST_ID}-config.json'))['ETag'])")

python3 -c "
import json
data = json.load(open('/tmp/cf-${DIST_ID}-config.json'))
config = data['DistributionConfig']
config['Enabled'] = False
json.dump(config, open('/tmp/cf-${DIST_ID}-disabled.json', 'w'), indent=2)
"

aws --profile sakimeshi --region us-east-1 \
  cloudfront update-distribution \
  --id $DIST_ID \
  --if-match $ETAG \
  --distribution-config file:///tmp/cf-${DIST_ID}-disabled.json \
  | tee aws-decommission-logs/21-cf-${DIST_ID}-disabled.json
```

**停止承認チェック:**
- [ ] sakai.sakimeshi.com が現役でないことを担当者確認済み

```bash
# ---- sakai.sakimeshi.com ----
DIST_ID="E3R8I3203AG4ZI"

aws --profile sakimeshi --region us-east-1 \
  cloudfront get-distribution-config --id $DIST_ID \
  > /tmp/cf-${DIST_ID}-config.json

ETAG=$(python3 -c "import json; print(json.load(open('/tmp/cf-${DIST_ID}-config.json'))['ETag'])")

python3 -c "
import json
data = json.load(open('/tmp/cf-${DIST_ID}-config.json'))
config = data['DistributionConfig']
config['Enabled'] = False
json.dump(config, open('/tmp/cf-${DIST_ID}-disabled.json', 'w'), indent=2)
"

aws --profile sakimeshi --region us-east-1 \
  cloudfront update-distribution \
  --id $DIST_ID \
  --if-match $ETAG \
  --distribution-config file:///tmp/cf-${DIST_ID}-disabled.json \
  | tee aws-decommission-logs/21-cf-${DIST_ID}-disabled.json
```

### 2-3. 確認

```bash
# 全ディストリビューションが Enabled=false になったことを確認
for dist_id in E2E7WQ9K5FXKGX E3E9OXW587FLR E3R8I3203AG4ZI; do
  echo -n "$dist_id: "
  aws --profile sakimeshi --region us-east-1 \
    cloudfront get-distribution --id $dist_id \
    --query 'Distribution.{Status:Status,Enabled:DistributionConfig.Enabled}' \
    --output json
done
```

**確認:**
- [ ] 3 つとも `Enabled: false` であること
- [ ] Status が `InProgress` → `Deployed` になるまで待つ（15〜30分）
- [ ] `http://sakimeshi.com` が引き続き GitHub Pages で表示されること

---

## Phase 4: 最終確認

```bash
echo "=== $(date '+%Y-%m-%d %H:%M:%S') ==="

echo "--- DNS ---"
dig +short sakimeshi.com A @8.8.8.8

echo "--- サイト表示 ---"
curl -sI http://sakimeshi.com | head -3

echo "--- ALB（LoadBalancerNotFound が正常） ---"
aws --profile sakimeshi --region ap-northeast-1 \
  elbv2 describe-load-balancers \
  --names sakimeshi-alb-redirect 2>&1

echo "--- CloudFront（全て Enabled=false が正常） ---"
aws --profile sakimeshi --region us-east-1 \
  cloudfront list-distributions \
  --query 'DistributionList.Items[].{Id:Id,Alias:Aliases.Items[0],Enabled:Enabled}' \
  --output table

echo "--- 残存リソース（放置） ---"
echo "Route53:"
aws --profile sakimeshi route53 list-hosted-zones \
  --query 'HostedZones[].Name' --output json
echo "S3:"
aws --profile sakimeshi s3 ls
echo "Lambda (ap-northeast-1):"
aws --profile sakimeshi --region ap-northeast-1 \
  lambda list-functions --query 'Functions[].FunctionName' --output json
echo "DynamoDB:"
aws --profile sakimeshi --region ap-northeast-1 \
  dynamodb list-tables --output json
```

**確認:**
- [ ] sakimeshi.com が GitHub Pages で表示されること
- [ ] ALB が `LoadBalancerNotFound` であること
- [ ] CloudFront が全て Enabled=false であること
- [ ] 残存リソース（S3, Lambda, DynamoDB 等）は放置で問題ないこと

---

## ロールバック手順

### ALB の再作成が必要な場合

`aws-decommission-logs/backup-alb.json`、`backup-alb-listeners.json`、`backup-alb-target-groups.json` を元に再作成。
ただし DNS はお名前.com に移行済みのため、ALB を再作成しても意味がない。

### CloudFront の再有効化

```bash
# 無効化と同じ手順で Enabled=true に戻す
DIST_ID="E2E7WQ9K5FXKGX"  # 対象のディストリビューション

aws --profile sakimeshi --region us-east-1 \
  cloudfront get-distribution-config --id $DIST_ID \
  > /tmp/cf-${DIST_ID}-config.json

ETAG=$(python3 -c "import json; print(json.load(open('/tmp/cf-${DIST_ID}-config.json'))['ETag'])")

python3 -c "
import json
data = json.load(open('/tmp/cf-${DIST_ID}-config.json'))
config = data['DistributionConfig']
config['Enabled'] = True
json.dump(config, open('/tmp/cf-${DIST_ID}-enabled.json', 'w'), indent=2)
"

aws --profile sakimeshi --region us-east-1 \
  cloudfront update-distribution \
  --id $DIST_ID \
  --if-match $ETAG \
  --distribution-config file:///tmp/cf-${DIST_ID}-enabled.json
```

> **注意:** 再有効化後、Status が `Deployed` になるまで 15〜30 分待つこと。
> `Deployed` 確認前はトラフィックが正常に配信されない場合がある。

---

## 今後の完全削除（別タスク）

Phase 1-2 で月額費用の約 75% を削減。残りのリソースは費用がほぼゼロのため、
完全削除は別タスクとして実施する。対象:

- CloudFront ディストリビューションの削除（無効化済み前提）
- Route 53 Hosted Zone の削除
- S3 バケットの削除（ログバケット 54万オブジェクトあり）
- Lambda / API Gateway / DynamoDB の削除
- ACM 証明書の削除
- セキュリティグループの削除
- IAM ユーザーのクリーンアップ
