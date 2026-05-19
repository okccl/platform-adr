# phase12_A17 作業ログ

| # | 作業内容 |
|---|---|
| 1 | ArgoCD SharedResourceWarning 解消（ReferenceGrant 整理・keycloak-routes 廃止） |
| 2 | gateway-routes Wave 引き上げ・bootstrap-sync 修正 |
| 3 | Backstage 起動時 DB 接続 retry 対応 |

---

## 1. ArgoCD SharedResourceWarning 解消

### 背景

ArgoCD で `ReferenceGrant/allow-gateway-to-backstage is part of applications argocd/gateway-routes and backstage` という SharedResourceWarning が発生していた。調査の結果、同一リソースが2箇所に定義されていることが判明した。

- `platform/backstage/app-config/reference-grant.yaml`（backstage App 管理）
- `platform/gateway/routes/platform/reference-grant-backstage.yaml`（gateway-routes App 管理）

調査を進める中で、Gateway API リソースの管理方針が混在していることも判明した。

**HTTPRoute と ReferenceGrant の役割の違い：**

- HTTPRoute は Gateway 側の設定（どこに転送するか）→ `gateway-routes` で一元管理が適切
- ReferenceGrant は namespace オーナーによる許可宣言（自分の namespace へのアクセスを許可する）→ 各 App が自分の namespace で管理が適切

keycloak は `keycloak/app-config/reference-grant.yaml` で自己管理していたが、monitoring・goldilocks・argo-rollouts の ReferenceGrant は gateway-routes に混在していた。また `keycloak-routes` が単体 App として存在していたのは、かつて `root-2-auth` / `root-3-others` という App-of-Apps グループ構造があった時代の名残であり、フラット化後は不要な分離となっていた。

### 実施内容

**ReferenceGrant の整理：**

```bash
# backstage の重複を削除
rm platform/gateway/routes/platform/reference-grant-backstage.yaml

# monitoring・goldilocks・argo-rollouts の ReferenceGrant を各 App に移動
mkdir -p platform/monitoring/app-config
mkdir -p platform/goldilocks/app-config
mkdir -p platform/argo-rollouts/app-config
# → 各 app-config/reference-grant.yaml を作成（sync-wave: "-1" 付与）

# gateway-routes から旧 ReferenceGrant ファイルを削除
rm platform/gateway/routes/platform/reference-grant-monitoring.yaml
rm platform/gateway/routes/platform/reference-grant-argo-rollouts.yaml

# kube-prometheus-stack・goldilocks・argo-rollouts の Application 定義に app-config source を追加
```

**keycloak-routes 廃止・gateway-routes 統合：**

```bash
# keycloak HTTPRoute を gateway-routes 配下に移動
mv platform/gateway/routes/keycloak/httproute-keycloak.yaml \
   platform/gateway/routes/platform/httproute-keycloak.yaml
rm -rf platform/gateway/routes/keycloak/
rm platform/applications/keycloak-routes.yaml
```

---

## 2. gateway-routes Wave 引き上げ・bootstrap-sync 修正

### 背景

keycloak-routes（Wave 16）を廃止して keycloak の HTTPRoute を gateway-routes（Wave 20）に統合したことで、bootstrap の Milestone ②「Keycloak 起動完了」表示時点では gateway-routes がまだ未実行のため、表示した `https://keycloak.platform.local` にアクセスできない状態になっていた。

### 実施手順

**gateway-routes を Wave 20 → Wave 16 に引き上げ：**

```yaml
# platform/applications/gateway-routes.yaml
argocd.argoproj.io/sync-wave: "16"  # 変更前: "20"
```

Wave 16 では goldilocks・argo-rollouts（Wave 20）の ReferenceGrant がまだ存在しないが、HTTPRoute の health check は `ResolvedRefs` を確認しない設計のため問題なし。

**bootstrap-sync に gateway-routes 完了待機を追加：**

keycloak pod が Ready（Wave 15 相当）になった後、gateway-routes App が Healthy になるまで待機してから Milestone ② を表示するよう修正した。

```makefile
@echo "gateway-routes（Wave 16）完了待機中..."; \
until kubectl get application gateway-routes -n $(ARGOCD_NS) \
    -o jsonpath='{.status.health.status}' 2>/dev/null | grep -q Healthy; do \
    sleep 10; \
done
```

---

## 3. Backstage 起動時 DB 接続 retry 対応

### 背景

Backstage は起動時に PostgreSQL への接続に失敗すると retry せず 503 のまま止まる。CNPG の "Cluster in healthy state" 判定から実際に接続を受け付けるまでわずかにラグがあり、bootstrap 時に発生していた（手動 restart で解消していた）。

### 実施手順

image 側で `pg_isready` による待機を追加した。initContainer での対処も可能だが、DB 接続待機はこの image を動かすための前提条件であるため image 側に持つほうが適切と判断。

**`packages/backend/entrypoint.sh`（新規作成）：**

```sh
#!/bin/sh
set -e

until pg_isready -h "$POSTGRES_HOST" -p "$POSTGRES_PORT" -U "$POSTGRES_USER" -q; do
  sleep 2
done

exec node packages/backend --config app-config.yaml --config app-config.production.yaml
```

**`packages/backend/Dockerfile` の変更：**

```dockerfile
# postgresql-client を追加（USER node の前）
RUN apt-get install -y --no-install-recommends postgresql-client

# bundle 展開後に entrypoint.sh をコピー
COPY --chown=node:node packages/backend/entrypoint.sh ./
RUN chmod +x entrypoint.sh

# CMD → ENTRYPOINT に変更
ENTRYPOINT ["./entrypoint.sh"]
```

push 後、GitHub Actions によりイメージがビルドされ platform-gitops の tag が自動更新された。bootstrap 実行で正常動作を確認。

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform/applications/gateway-routes.yaml` | sync-wave を 20 → 16 に変更 |
| `platform/applications/keycloak-routes.yaml` | 削除（gateway-routes に統合） |
| `platform/applications/kube-prometheus-stack.yaml` | app-config source を追加 |
| `platform/applications/goldilocks.yaml` | app-config source を追加 |
| `platform/applications/argo-rollouts.yaml` | single-source → multi-source に変更、app-config source を追加 |
| `platform/gateway/routes/platform/httproute-keycloak.yaml` | 新規（keycloak/ から移動） |
| `platform/gateway/routes/platform/reference-grant-backstage.yaml` | 削除（重複解消） |
| `platform/gateway/routes/platform/reference-grant-monitoring.yaml` | 削除（各 App に移動） |
| `platform/gateway/routes/platform/reference-grant-argo-rollouts.yaml` | 削除（各 App に移動） |
| `platform/monitoring/app-config/reference-grant.yaml` | 新規作成 |
| `platform/goldilocks/app-config/reference-grant.yaml` | 新規作成 |
| `platform/argo-rollouts/app-config/reference-grant.yaml` | 新規作成 |
| `k3d/Makefile`（platform-infra） | bootstrap-sync に gateway-routes 完了待機を追加 |
| `packages/backend/Dockerfile`（backstage） | postgresql-client 追加・entrypoint.sh 組み込み |
| `packages/backend/entrypoint.sh`（backstage） | 新規作成（pg_isready 待機） |
