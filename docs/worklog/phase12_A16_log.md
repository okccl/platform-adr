# Phase12-A16 作業ログ

| # | 作業内容 |
|---|---|
| 1 | CoreDNS rewrite の事前確定（Envoy Gateway SVC 名固定） |
| 2 | bootstrap 単一ルート集約（wave 番号付け替え・root.yaml 作成・旧3ルート削除・Makefile 更新） |
| 3 | bootstrap テスト・タイミング問題の修正（ESO webhook / ExternalSecret 集中管理） |
| 4 | applications/ フラット化 |
| 5 | ExternalSecret 集中管理とディレクトリ構造整理 |
| 6 | kind 混在 YAML のチェックと分離 |
| 7 | apps-root 整理（bootstrap/ 削除・apps-gitops 分離・repoURL 修正） |
| 8 | bootstrap-apps 統合（user-apps-infra 待機後に apps-root apply） |
| 9 | Trivy Helm values 修正と wave 後退（20→23） |
| 10 | ESO webhook 競合対策（wave 13 移動・retry 線形化） |
| 11 | bootstrap テスト結果 |
| 12 | ArgoCD PreSync deadlock 修正（argocd-redis-secret-init Job TTL 問題） |
| 13 | CNPG Cluster ヘルスチェック追加（wave 14 早期完了問題） |
| 14 | Gateway ヘルスチェック追加（wave 4 早期完了問題） |
| 15 | keycloak 待機の kubectl wait 化（port-forward 切断耐性） |
| 16 | ArgoCD を wave-5 に移動（自己管理再起動の wave 10–16 影響を排除） |
| 17 | bootstrap-sync の port-forward 廃止 |
| 18 | argoproj.io/Application ヘルスチェック追加（wave 進行の根本修正） |
| 19 | prometheus-operator-crds を wave-1 に追加（ServiceMonitor CRD 先行インストール） |
| 20 | root App sync retry ポリシー追加（一時 Degraded による wave 失敗対策） |
| 21 | CRD ヘルスチェックを exists=Healthy に変更（3分 Progressing 問題の解消） |
| 22 | bootstrap-sync 待機メッセージの1回表示化・sh 互換 redirect 修正 |
| 23 | bootstrap 経過時間表示追加（bootstrap-start ターゲット・フル計測） |
| 24 | Keycloak pod 待機ラベルセレクタ修正（keycloak → keycloakx） |
| 25 | bootstrap-apps の argocd app wait を kubectl ベースに置き換え |
| 26 | backstage HTTPRoute 用 ReferenceGrant 追加 |
| 27 | bootstrap 完了マイルストーン再構成（platform / アプリ起動を分離） |
| 28 | k3d/Makefile 整理（重複定義・順序・不要ターゲット整理） |
| 29 | gateway-routes Degraded 修正（HTTPRoute カスタムヘルスチェック追加） |
| 30 | bootstrap 完走確認 |

---

## 1. CoreDNS rewrite の事前確定

### 背景

bootstrap 単一ルート集約の前提として、`*.platform.local` の CoreDNS rewrite ルールを bootstrap 前の時点で確定できるようにする必要があった。

従来の `bootstrap-sync` では、Envoy Gateway が Gateway リソースの apply 後に動的生成する Service 名（`envoy-envoy-gateway-system-eg-<ハッシュ>`）を `kubectl get svc -l owning-gateway-name=eg` で取得してから CoreDNS を更新していた。この動的取得処理は ArgoCD の wave 自動進行に割り込めないため、単一ルート化の障害になっていた。

Envoy Gateway v1.1 以降で利用可能な `EnvoyProxy` リソースの `spec.provider.kubernetes.envoyService.name` を使い、Service 名を `envoy-eg` に固定することで解決した。

### 実施内容

`platform/gateway/config/envoy-proxy-config.yaml` を新規作成し、`gateway.yaml` に `infrastructure.parametersRef` を追加した。

```yaml
# 新規: envoy-proxy-config.yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: eg-proxy-config
  namespace: envoy-gateway-system
spec:
  provider:
    type: Kubernetes
    kubernetes:
      envoyService:
        name: envoy-eg
```

```yaml
# gateway.yaml に追加
spec:
  infrastructure:
    parametersRef:
      group: gateway.envoyproxy.io
      kind: EnvoyProxy
      name: eg-proxy-config
```

`k3d/Makefile` の `fix-coredns` ターゲットに `*.platform.local` rewrite ルールを追加し、`bootstrap-sync` の動的 CoreDNS パッチ処理を削除した。

```makefile
cf=d['data']['Corefile']; \
rule='    rewrite name keycloak.platform.local envoy-eg.envoy-gateway-system.svc.cluster.local\n...'; \
d['data']['Corefile']=cf.replace('ready\n', 'ready\n'+rule) if 'rewrite name keycloak.platform.local' not in cf else cf; \
```

push 後に ArgoCD が gateway-config を自動 sync し、旧 SVC（ハッシュ付き）が削除されて `envoy-eg` が作成された。`make fix-coredns` を手動実行して現クラスタの CoreDNS を更新し、`argocd.platform.local` / `keycloak.platform.local` の疎通を確認した。

---

## 2. bootstrap 単一ルート集約

### wave 番号付け替え

グループ間を 10 刻みで分離し、root-2-auth（旧 1–6 → 新 10–15）・root-3-others（旧 1–3 → 新 20–22）を更新した。`user-apps.yaml` / `user-apps-project.yaml` には wave アノテーション自体がなかったため wave 20 を追加した。

### 単一ルート root.yaml 作成

`platform/group-roots/root.yaml` を新規作成。`platform/applications` を `recurse: true` で参照し、`vcluster-dev.yaml` のみ exclude する構成。

```yaml
directory:
  recurse: true
  exclude: 'root-3-others/vcluster-dev.yaml'
```

### 移行操作

```bash
kubectl apply -f ~/platform-gitops/platform/group-roots/root.yaml
argocd app delete root-1-gateway --cascade=false
argocd app delete root-2-auth    --cascade=false
argocd app delete root-3-others  --cascade=false
```

`--cascade=false` により子 App（cert-manager・keycloak 等）を残したまま旧3ルートを削除した。root App が自動 sync して Synced + Healthy になったことを確認後、旧3ルートの YAML ファイルを削除した。

### Makefile 更新

`bootstrap-sync` を3ルートの順次 apply から単一ルートの apply + wait に置き換えた。

```makefile
kubectl apply -f ~/platform-gitops/platform/group-roots/root.yaml
argocd app sync root --server-side || true
# ... Envoy Gateway 待機・ログイン切替 ...
argocd app wait root --sync --health --timeout 1800
```

---

## 3. bootstrap テスト・タイミング問題の修正

### 問題 1：wave 12 App（external-secrets-config）の初回 sync 失敗

**原因**：ESO の webhook Pod がエンドポイントに登録される前に wave 12 の sync が走り、`ClusterSecretStore` apply 時に `no endpoints available for service "external-secrets-webhook"` が発生。

**対処**：`external-secrets-config` と `kyverno-policies`（ともに wave 12）に retry ポリシーを追加。

```yaml
syncPolicy:
  retry:
    limit: 20
    backoff:
      duration: 30s
      factor: 2
      maxDuration: 10m
```

### 問題 2：keycloak-db の CNPG Cluster が作成されない

**原因**：`ClusterSecretStore` 未作成の状態で wave 13 が走り、ExternalSecret が "could not get secret data from provider" になった。その後 retry ポリシーが 4 回目で成功したが、keycloak App wait の 900 秒 timeout 以内に収まらなかった。

**根本対処**：全 ExternalSecret を `platform/secrets/config/` に集中管理する方針に変更。`external-secrets-config`（wave 12）が ClusterSecretStore と全 ExternalSecret を一括 apply することで、wave 13 以降の App 起動時には必ず Secret が存在する状態を保証する。

同様に `keycloak-db` / `backstage-db` にも retry ポリシーを追加（暫定対処として残存）。

### 問題 3：bootstrap-sync の完了条件が分かりにくい

`argocd app wait root`（全 37 App 待機）から、ArgoCD 起動時・Keycloak 起動時の2段階マイルストーン表示に変更。

```
=== ArgoCD 起動完了 ===
URL: https://argocd.platform.local
Username: admin
Password: xxxxxxxx

=== bootstrap 完了 ===
ArgoCD URL:   https://argocd.platform.local
Keycloak URL: https://keycloak.platform.local
```

---

## 4. applications/ フラット化

`platform/applications/root-1-gateway/`・`root-2-auth/`・`root-3-others/` のサブディレクトリを廃止し、全 38 ファイルを `platform/applications/` 直下にフラット化。

`root.yaml` の exclude を `root-3-others/vcluster-dev.yaml` → `vcluster-dev.yaml` に修正。

---

## 5. ExternalSecret 集中管理とディレクトリ構造整理

### 方針決定

アプリ固有の ExternalSecret をアプリ App に持たせる設計は fresh bootstrap 時のタイミング問題を引き起こすため、全 ExternalSecret を `platform/secrets/config/` に集中管理する方針に変更。

### 移動対象

| 元パス | 移動先 |
|---|---|
| `keycloak/db-config/external-secret.yaml` | `secrets/config/keycloak-db-credentials.yaml` |
| `keycloak/db-config/minio-backup-secret.yaml` | `secrets/config/keycloak-minio-backup.yaml` |
| `keycloak/config-cli/external-secret.yaml` | `secrets/config/keycloak-config-cli-secret.yaml` |
| `backstage/db-config/external-secret.yaml` | `secrets/config/backstage-db-credentials.yaml` |
| `backstage/db-config/minio-backup-secret.yaml` | `secrets/config/backstage-minio-backup.yaml` |
| `backstage/app-config/external-secret.yaml` | `secrets/config/backstage-secret.yaml` |
| `backstage/app-config/ghcr-pull-secret.yaml` | `secrets/config/backstage-ghcr-pull-secret.yaml`・`secrets/generators/backstage-ghcr-token.yaml` に分割 |
| `monitoring/alerts/discord-webhook-secret.yaml` | `secrets/config/discord-webhook-secret.yaml` |

ExternalSecret の wave は 4（ClusterSecretStore wave 3 の後）に統一。Namespace も `secrets/namespaces/` に分離。

### secrets/ ディレクトリ最終構造

```
secrets/
  config/      # ClusterSecretStore + ExternalSecret（wave 3–4）
  generators/  # ESO Generator（GithubAccessToken 等）（wave 5–6）
  namespaces/  # wave 12 に必要な事前 Namespace
  sources/     # SOPS 暗号化 source secrets
```

`external-secrets-config` App を multi-source（config・generators・namespaces の3パス）に変更。

---

## 6. kind 混在 YAML のチェックと分離

全リポジトリを対象に「1ファイル1リソース種別」方針に違反する YAML を検索し、対象3ファイルを分離した。

| ファイル | 混在 kind | 対処 |
|---|---|---|
| `crossplane/config/provider.yaml` | `DeploymentRuntimeConfig` + `Provider` | `provider-helm-runtime.yaml` に分離 |
| `cert-manager/config/cluster-issuer.yaml` | `ClusterIssuer` + `Certificate` | `certificate.yaml` に分離 |
| `crossplane/config/rbac.yaml` | `ServiceAccount` + `ClusterRoleBinding` | `service-account.yaml` + `cluster-role-binding.yaml` に分割・旧ファイル削除 |

---

## 7. apps-root.yaml の整理

`platform-gitops/bootstrap/apps-root.yaml` の問題を発見・修正した。

- **repoURL バグ**：`platform-gitops.git` の `apps/` を参照していたが、該当ディレクトリは存在しない。正しくは `apps-gitops.git`
- **配置場所**：`bootstrap-apps` ターゲットから apply するため `platform-infra/k3d/` に移動
- `platform-gitops/bootstrap/` ディレクトリは削除（`.gitkeep` のみだったため）

---

## 8. bootstrap-apps の統合と user-apps 整理

### bootstrap-apps を make bootstrap に統合

`bootstrap-sync` が Keycloak 起動まで待機するようになったため、`bootstrap-apps` を `bootstrap` target に追加。ESO・CNPG が確実に稼働済みの状態でアプリ層を展開できる。

`user-apps-infra`（`sample-app` Namespace + `sample-apps` AppProject）の稼働確認後に `apps-root` を apply するよう `bootstrap-apps` に `argocd app wait user-apps-infra` を追加。

### user-apps / apps-root の一本化

`user-apps.yaml`（wave 20）が `apps-gitops/apps/*/application.yaml` を管理しており、`apps-root.yaml` と役割が重複していた。また wave 20 でアプリが展開されるため「platform 安定後にアプリを展開する」方針が実現できていなかった。

- `platform/applications/user-apps.yaml` / `user-apps-project.yaml` を削除
- `apps-root.yaml` を `recurse: true` + `include: '*/application.yaml'` に修正
- `user-apps-infra.yaml` は platform 側リソース（Namespace ラベル・AppProject）のため platform-gitops に残存

---

## 9. Trivy 修正

### Helm values キー名バグ修正

`excludeNamespaces` と `concurrentScanJobsLimit` のキー名が誤っており、設定が一切反映されていなかった。

| 修正前 | 修正後 |
|---|---|
| `operator.excludeNamespaces` | `excludeNamespaces`（トップレベル） |
| `operator.concurrentScanJobsLimit: 1` | `operator.scanJobsConcurrentLimit: 1` |
| `operator.concurrentNodeScanners: 1` | 削除（存在しないキー） |

### wave 20 → 23 に変更

他コンポーネント起動後にスキャンが始まるよう wave を後退させた。

---

## 10. ESO webhook 競合対策

### 問題

`external-secrets-config`（wave 12）の retry policy が `duration: 30s, factor: 2` の指数バックオフであり、ESO webhook の endpoint 登録が間に合わない場合に最大数分待機が発生していた。

### 対処

wave を1つずらして `kyverno-policies`（wave 12）を自然なバッファとして活用し、retry を線形短縮した。

| App | 変更前 | 変更後 |
|---|---|---|
| `external-secrets-config` | wave 12、30s/factor:2/max:10m/limit:20 | wave 13、10s/factor:1/max:10s/limit:60 |
| `keycloak-db` | wave 13 | wave 14 |
| `keycloak` | wave 14 | wave 15 |
| `keycloak-config-cli` / `keycloak-routes` | wave 15 | wave 16 |

retry limit を 20（200秒）から 60（600秒）に増加。初回 bootstrap テストで limit 20 到達による sync error が発生したため修正。

---

## 11. bootstrap テスト結果

2回のbootstrap実行で以下を確認・修正した。

- `external-secrets-config` sync error → retry limit 不足（200秒）が原因。limit を 60（600秒）に修正
- `backstage` Pod が起動時に `backstage-db-rw` を DNS 解決できず 503 → 手動 restart で解消。根本対応（Dockerfile に `pg_isready` wait を追加）は次セッションへ持ち越し
- `argocd` App の OutOfSync（ServiceMonitor 未作成）は automated sync で自己解決

---

## 12. ArgoCD PreSync deadlock 修正

### 問題

bootstrap 時に `argocd` Application（wave 20）の sync が "waiting for completion of hook batch/Job/argocd-redis-secret-init" のまま永遠に止まる。手動で TERMINATE しないと解消しなかった。

### 原因

ArgoCD Helm chart 9.5.2 に含まれる `argocd-redis-secret-init` PreSync hook Job に `ttlSecondsAfterFinished: 60` が設定されている。Job が完了してから 60 秒後に Kubernetes が自動削除するが、ArgoCD のポーリング間隔が 60 秒前後であるため Job 削除後も "Running" 状態のまま完了を検知できない。`retry` は sync が **Failed** で終了した場合にしか機能せず、"Running" のまま stuck した場合は無効。

### 対処

`redisSecretInit.enabled: false` を設定して PreSync Job を無効化し、`bootstrap-argocd` で `argocd-redis` secret を事前作成するよう Makefile を修正した。

```yaml
# platform/argocd/values.yaml
redisSecretInit:
  enabled: false
```

```makefile
# k3d/Makefile（bootstrap-argocd）
kubectl create secret generic argocd-redis \
    --from-literal=auth=$$(openssl rand -hex 16) \
    -n $(ARGOCD_NS) \
    --dry-run=client -o yaml \
    | kubectl label --local -f - app.kubernetes.io/part-of=argocd -o yaml \
    | kubectl apply -f -
```

---

## 13. CNPG Cluster ヘルスチェック追加

### 問題

bootstrap 時に keycloak が CrashLoopBackOff になる。ログで "Connection to keycloak-db-rw:5432 refused" が出ており、keycloak-db CNPG Cluster が "Setting up primary" のまま wave 15（keycloak）が起動していた。

### 原因

ArgoCD はヘルスチェックが未定義の CRD リソースを**作成直後に Healthy と判定する**。CNPG `Cluster` リソースのカスタムヘルスチェックが未定義だったため、wave 14（keycloak-db）が CNPG Cluster 作成直後に完了と判定され wave 15 が早期起動した。

### 対処

`postgresql.cnpg.io_Cluster` のカスタムヘルスチェックを追加した。初回実装時に `string.find(string.lower(...))` を使用したが、ArgoCD の Lua サンドボックスでは `string` グローバルが利用不可のため `attempt to index a non-table object(nil) with key 'find'` エラーが発生した。`string.find` を削除してシンプルな等値比較に修正した。

```yaml
resource.customizations.health.postgresql.cnpg.io_Cluster: |
  hs = {}
  if obj.status ~= nil and obj.status.phase ~= nil then
    if obj.status.phase == "Cluster in healthy state" then
      hs.status = "Healthy"
      hs.message = obj.status.phase
      return hs
    end
  end
  hs.status = "Progressing"
  hs.message = obj.status ~= nil and obj.status.phase or "Waiting for cluster"
  return hs
```

---

## 14. Gateway ヘルスチェック追加

### 問題

wave 10 以降の App が wave 4 完了前に起動している。

### 原因

`gateway-config`（wave 4）が管理する Gateway リソースにヘルスチェックが未定義で、リソース作成直後に wave 4 が Healthy と判定されていた。

### 対処

`gateway.networking.k8s.io_Gateway` のカスタムヘルスチェックを追加した。Gateway の `status.conditions` に `Programmed: True` が設定された時点を Healthy とすることで、Envoy proxy Pod が実際に起動してトラフィックを処理できる状態になるまで wave 10 への遷移をブロックする。

```yaml
resource.customizations.health.gateway.networking.k8s.io_Gateway: |
  hs = {}
  if obj.status ~= nil and obj.status.conditions ~= nil then
    for i, condition in ipairs(obj.status.conditions) do
      if condition.type == "Programmed" then
        if condition.status == "True" then
          hs.status = "Healthy"
          hs.message = condition.message
          return hs
        else
          hs.status = "Progressing"
          hs.message = condition.message
          return hs
        end
      end
    end
  end
  hs.status = "Progressing"
  hs.message = "Waiting for Gateway to be programmed"
  return hs
```

---

## 15. keycloak 待機の kubectl wait 化

### 問題

`bootstrap-sync` 内で `argocd app wait keycloak --health --timeout 900` を実行中に、wave 20 の `argocd` Application self-managed sync により argocd-server Pod が再起動し、スクリプトが停止する。

### 対処

argocd CLI に依存しない `kubectl wait` に変更した。Pod が存在しない段階で `kubectl wait` を実行すると "no matching resources found" で即エラーになるため、`until` ループで Pod が Ready になるまでリトライする構成にした。

```makefile
@echo "Keycloak pod が Ready になるまで待機中..."; \
until kubectl wait --for=condition=ready pod \
    -l app.kubernetes.io/name=keycloak \
    -n keycloak --timeout=10s 2>/dev/null; do \
    sleep 10; \
done
```

---

## 16. ArgoCD を wave-5 に移動

### 問題

wave 20 の `argocd` Application が自己管理 sync を実行し、argocd-server が再起動する。bootstrap-sync スクリプトが argocd CLI 経由で Keycloak を待機していたため、再起動で接続が切れてスクリプトが停止していた。

### 原因

wave 20 は Keycloak 起動完了（wave 16）直後に始まるため、"bootstrap 完了" 表示の直後に ArgoCD が一時停止する。

### 対処

`argocd` Application を wave 20 → wave 5 に移動した。wave 4（Gateway 起動）と wave 10（auth 基盤）の間に自己管理再起動を完結させ、wave 10–16 中に ArgoCD が再起動しないよう構成した。

bootstrap-sync に Application オブジェクトの Healthy 待機（kubectl 経由）を追加し、"ArgoCD 起動完了" メッセージを再起動完了後に表示するよう修正した。

```makefile
@until kubectl get application argocd -n $(ARGOCD_NS) \
    -o jsonpath='{.status.health.status}' 2>/dev/null | grep -q Healthy; do \
    echo "ArgoCD 自己管理 sync 待機中..."; sleep 10; \
done
```

wave 5 完了判定は ArgoCD 組み込みの Deployment/StatefulSet ヘルスチェックで行われるため、カスタムヘルスチェックの追加は不要。

---

## 17. bootstrap-sync の port-forward 廃止

### 問題

bootstrap-sync が localhost:8080 への port-forward を使って `argocd app sync root --server-side` を実行していた。wave 5 の ArgoCD 自己管理 sync による Pod 再起動で port-forward が切断され、スクリプトが停止する。

### 原因

kubectl port-forward は対象 Pod と TCP 接続を維持するため Pod 再起動で接続が切れる。`argocd app sync root` はストリーミング接続を維持するためスクリプトが止まる。

### 対処

port-forward と `argocd app sync root` を廃止した。root App は `syncPolicy.automated` があるため `kubectl annotate argocd.argoproj.io/refresh=hard` で即時リフレッシュをトリガーするだけで auto-sync が動作する。

```makefile
kubectl apply -f ~/platform-gitops/platform/group-roots/root.yaml
@until kubectl get application root -n $(ARGOCD_NS) &>/dev/null; do sleep 5; done
kubectl annotate application root -n $(ARGOCD_NS) argocd.argoproj.io/refresh=hard --overwrite
```

---

## 18. argoproj.io/Application ヘルスチェック追加

### 問題

wave 5 の argocd Application が "Synced" になった直後（2秒以内）に wave 10 が開始し、wave 10 以降の全 Application が ContainerCreating 状態で一斉起動する。

### 原因

ArgoCD の sync-wave は wave N のリソースが Healthy になるまで wave N+1 を作成しないが、新規作成された Application オブジェクトは `status.health.status` が nil のため ArgoCD が "Unknown" と判定し wave をブロックしない。結果として wave N の Application 作成直後に wave N+1 が始まっていた。以前は auto-retry で各依存コンポーネントが遅延起動しても結果的に通過していたが、厳密な wave 制御が機能しないまま運用されていた。

### 対処

`argoproj.io/Application` のカスタムヘルスチェックを追加し、nil/空の `status.health.status` を Progressing として扱うよう修正した。

```yaml
resource.customizations.health.argoproj.io_Application: |
  hs = {}
  if obj.status ~= nil and obj.status.health ~= nil then
    if obj.status.health.status == "Healthy" then
      hs.status = "Healthy"
      hs.message = obj.status.health.message or "Application is healthy"
      return hs
    end
    if obj.status.health.status == "Degraded" then
      hs.status = "Degraded"
      hs.message = obj.status.health.message or "Application is degraded"
      return hs
    end
  end
  hs.status = "Progressing"
  hs.message = "Waiting for Application to become healthy"
  return hs
```

---

## 19. prometheus-operator-crds を wave-1 に追加

### 問題

Application ヘルスチェック追加により wave が正しく機能するようになった結果、wave 1 の cert-manager Application が ServiceMonitor CRD 不在で sync 失敗し、wave 2 以降が永続的にブロックされた。

### 原因

cert-manager Helm chart は ServiceMonitor リソースを作成するが、ServiceMonitor CRD は wave 10 の kube-prometheus-stack が提供する。以前は wave 機構が機能していなかったため auto-retry で通過していたが、wave が正しく機能するようになり再 sync の機会がなくなった。

### 対処

`prometheus-community/prometheus-operator-crds`（CRD 専用チャート）を wave 1 に追加した（`envoy-gateway-crds` と同パターン）。kube-prometheus-stack の `crds.enabled: false` を設定して二重インストールを防止した。

```yaml
# platform/applications/prometheus-operator-crds.yaml（新規）
source:
  repoURL: https://prometheus-community.github.io/helm-charts
  chart: prometheus-operator-crds
  targetRevision: 29.0.0
```

```yaml
# platform/monitoring/values.yaml に追加
crds:
  enabled: false
```

---

## 20. root App sync retry ポリシー追加

### 問題

bootstrap 実行中に `cnpg`・`envoy-gateway-crds` が wave の health チェック時点で一時的に Degraded になり、root App の sync が失敗する。

```
Failed last sync attempt: one or more synchronization tasks completed unsuccessfully (retried 5 times).
```

root の syncResult で対象 Application が `hookPhase: Failed`・`message: "Application is degraded"` になっていた。これは `argoproj.io/Application` カスタムヘルスチェックが `status.health.status == "Degraded"` を検出したことを示す。

### 原因

`argoproj.io/Application` ヘルスチェック追加（セクション 18）により wave が正確に機能するようになった結果、wave N の Application が deploy 直後に一時的な Degraded 状態を踏んだとき、root sync がその wave を Failed と判定するようになった。一時的な Degraded（数秒〜数十秒で自己回復）だが、retry なしでは5回の selfHeal 試行後に root が諦めてしまう。

### 対処

root App の `syncPolicy` に retry ポリシーを追加した。初期値（30s/factor:2/上限5分）は実測の回復時間（数十秒）に対して長すぎたため、10s/factor:2/上限60s に短縮した。

```yaml
syncPolicy:
  retry:
    limit: 5
    backoff:
      duration: 10s
      factor: 2
      maxDuration: 60s
```

---

## 21. CRD カスタムヘルスチェック（exists=Healthy）

### 問題

`envoy-gateway-crds`・`prometheus-operator-crds`（wave 1）が Synced になった後も約3分間 Progressing のまま待機し、wave 2 への遷移が遅延する。

### 原因

ArgoCD 組み込みの CRD ヘルスチェックが `NamesAccepted=False` を Degraded と判定するため、apply 直後の一時的な状態で Degraded になることがある。対策として `NamesAccepted=False` → Progressing に変更するカスタムヘルスチェックを追加したが、`Established=True` になるまで Progressing を返す実装にしたため、ArgoCD の reconciliation cycle（デフォルト 180s）まで health が再評価されず約3分待ちが発生した。

実測で cert-manager・cilium が Healthy になってから CRD Application が Healthy になるまで 3〜4 分の差があることを確認。CRD 自体は apply 後数秒で Established になっているため、遅延の全てが ArgoCD の reconciliation 待ちであった。

### 対処

CRD ヘルスチェックを「存在すれば Healthy」に変更した。wave 2 が始まる時点では cert-manager・cilium の起動待ちに 2〜3 分かかるため、その間に CRD は必ず Established 済みになることを確認済み。

```yaml
resource.customizations.health.apiextensions.k8s.io_CustomResourceDefinition: |
  hs = {}
  hs.status = "Healthy"
  hs.message = "CRD exists"
  return hs
```

---

## 22. bootstrap-sync 待機メッセージの1回表示化・sh 互換 redirect 修正

### 背景

`until` ループ内でメッセージを出力していたため、待機中に同じ文字列が繰り返し表示されていた。

### 対処

ループ前に1回だけ `echo` し、ループ本体からメッセージを削除した。

```makefile
# 変更前
@until kubectl get deployment envoy-gateway ... 2>/dev/null; do echo "envoy-gateway 待機中..."; sleep 5; done

# 変更後
@echo "envoy-gateway 待機中..."; until kubectl get deployment envoy-gateway ... >/dev/null 2>&1; do sleep 5; done
```

変更時に `&>/dev/null`（bash 専用構文）を使用したところ、Makefile のデフォルトシェル（`/bin/sh`）では `&>` が無効でループ条件が常に成功扱いになり、`until` ループがスキップされる不具合が発生した。`>/dev/null 2>&1` に修正して解消した。

---

## 23. bootstrap 経過時間表示追加

### 背景

bootstrap の各マイルストーン到達時刻と総経過時間を把握したかった。`bootstrap-sync` 単体での実行時も計測できるよう設計する必要があった。

### 実施内容

`bootstrap` ターゲットに `bootstrap-start` を追加し、`date +%s > /tmp/bootstrap_start` でフル bootstrap の開始時刻を記録する。`bootstrap-sync` 単体実行時はファイルが存在しない場合のみ書き込む。

```makefile
bootstrap-start:
    @date +%s > /tmp/bootstrap_start
    @echo "Bootstrap 開始: $$(date '+%H:%M:%S')"

bootstrap-sync:
    @[ -f /tmp/bootstrap_start ] || (date +%s > /tmp/bootstrap_start && echo "Bootstrap 開始: $$(date '+%H:%M:%S')")
```

各マイルストーンで `$$(date +%s) - $$(cat /tmp/bootstrap_start)` で差分を算出し表示する。

---

## 24. Keycloak pod 待機ラベルセレクタ修正

### 問題

keycloak Application が Healthy なのに bootstrap-sync の Keycloak 待機が終わらない。

### 原因

待機条件に `app.kubernetes.io/name=keycloak` を使用していたが、Helm chart が生成する Pod の実際のラベルは `app.kubernetes.io/name=keycloakx` だった。

### 対処

```makefile
# 変更前
-l app.kubernetes.io/name=keycloak

# 変更後
-l app.kubernetes.io/name=keycloakx
```

---

## 25. bootstrap-apps の argocd app wait を kubectl ベースに置き換え

### 問題

```
{"level":"fatal","msg":"rpc error: code = PermissionDenied desc = permission denied"}
make: *** [Makefile:170: bootstrap-apps] Error 20
```

### 原因

`bootstrap-sync` でのログインセッションが `bootstrap-apps` 実行時に失効していた（ArgoCD 自己管理再起動でセッションが無効化された可能性）。

### 対処

`argocd app wait` を `kubectl get application` ベースのループに置き換えた。argocd CLI 認証に依存しなくなる。

```makefile
@echo "user-apps-infra 待機中..."; \
until kubectl get application user-apps-infra -n $(ARGOCD_NS) \
    -o jsonpath='{.status.health.status}' 2>/dev/null | grep -q Healthy; do \
    sleep 10; \
done
```

---

## 26. backstage HTTPRoute 用 ReferenceGrant 追加

### 問題

bootstrap 完了後、`gateway-routes` Application が Degraded になっていた。

### 原因

```
BAD: backstage ResolvedRefs=False
Backend ref to Service backstage/backstage not permitted by any ReferenceGrant.
```

`platform/gateway/routes/platform/` に backstage 用の ReferenceGrant が存在しなかった。他コンポーネント（monitoring・goldilocks・argo-rollouts）には作成済みだったが backstage だけ漏れていた。

### 対処

`reference-grant-backstage.yaml` を新規作成した。

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-gateway-to-backstage
  namespace: backstage
spec:
  from:
    - group: gateway.networking.k8s.io
      kind: HTTPRoute
      namespace: envoy-gateway-system
  to:
    - group: ""
      kind: Service
```

---

## 27. bootstrap 完了マイルストーン再構成

### 背景

`bootstrap-apps` の完了条件として root（全 platform wave）と apps-root（ユーザーアプリ）を両方 Healthy 待機していたが、開発環境ではユーザーアプリが常に Healthy になる保証がないため分離が必要だった。

### 変更後の構成

| タイミング | 表示 | 内容 |
|---|---|---|
| wave-5 完了 | `=== ArgoCD 起動完了 ===` | URL・パスワード・経過時間 |
| wave-16 完了 | `=== Keycloak 起動完了 ===` | URL・経過時間 |
| root Healthy | `=== Platform Bootstrap 完了 ===` | 全 URL・総経過時間 |
| apps-root Healthy | `=== アプリ起動完了 ===` | 総経過時間のみ |

platform bootstrap 完了（root Healthy = wave 0–23 全完了）を主要マイルストーンとし、ユーザーアプリはその後に別待機する。

---

## 28. k3d/Makefile 整理（重複定義・順序・不要ターゲット整理）

bootstrap 安定化に伴い `k3d/Makefile` に問題が蓄積していたため整理した。

- `bootstrap:` ターゲットが 2 行定義されており、GNU Make が依存関係をマージするため `bootstrap-start` の実行順序が不定になっていた → 古い定義を削除
- `bootstrap-start` が `.PHONY` に未登録だった → 追加
- ファイル内のターゲット定義順が `bootstrap:` の実行順と一致していなかった（`fix-coredns` が `install-cilium` より前にあった）→ 実行順に並び替え、`bootstrap:` を `.PHONY` 直後に配置
- `cluster-delete` / `cluster-status` / `generate-dr-manifests` は bootstrap フローに含まれない → `platform-infra/Makefile`（トップレベル）に移動し、`k3d/Makefile` からは削除。`generate-dr-manifests` は `$(MAKE) -C k3d` 委譲から `python3 k3d/scripts/generate-dr-manifests.py` の直接実装に変更

---

## 29. gateway-routes Degraded 修正（HTTPRoute カスタムヘルスチェック追加）

### 問題

bootstrap 実行後、`gateway-routes` Application が `Synced Degraded` のままだった。

### 原因

```
backstage: ResolvedRefs=False
Failed to process route rule 0 backendRef 0: service backstage/backstage not found.
```

`gateway-routes`（wave 20）が backstage HTTPRoute を作成した時点では backstage Service（wave 22）がまだ存在しない。ArgoCD の組み込みヘルスチェックが `ResolvedRefs=False` を Degraded と判定していた。

なお #26 で追加した ReferenceGrant は機能しており、permission エラーは解消済み。今回は Service 未存在による別エラー。

### 対処

`platform/argocd/values.yaml` に HTTPRoute のカスタムヘルスチェックを追加した。`Accepted=True`（ゲートウェイがルートを受理済み）であれば Healthy と判定し、`ResolvedRefs` は確認しない。

バックエンド Service の起動確認は backstage Application のヘルスチェック（Deployment → Pod Ready）が担保するため、ルート側で重複確認する必要はない。

```lua
resource.customizations.health.gateway.networking.k8s.io_HTTPRoute: |
  hs = {}
  if obj.status ~= nil and obj.status.parents ~= nil then
    for i, parent in ipairs(obj.status.parents) do
      for j, condition in ipairs(parent.conditions or {}) do
        if condition.type == "Accepted" then
          if condition.status == "True" then
            hs.status = "Healthy"
            hs.message = condition.message
            return hs
          else
            hs.status = "Progressing"
            hs.message = condition.message
            return hs
          end
        end
      end
    end
  end
  hs.status = "Progressing"
  hs.message = "Waiting for HTTPRoute to be accepted"
  return hs
```

---

## 30. bootstrap 完走確認

手動操作なしで `make bootstrap` が完走することを確認した。

| マイルストーン | 経過時間 |
|---|---|
| ArgoCD 起動完了 | 6分20秒 |
| Keycloak 起動完了 | 12分17秒 |
| Platform Bootstrap 完了（root Healthy） | 15分31秒 |
| アプリ起動完了（apps-root Healthy） | 16分39秒 |

---

## 変更ファイル一覧

| ファイル | 変更内容 |
|---|---|
| `platform-gitops/platform/gateway/config/envoy-proxy-config.yaml` | 新規作成：EnvoyProxy リソース（SVC 名 `envoy-eg` に固定） |
| `platform-gitops/platform/gateway/config/gateway.yaml` | `infrastructure.parametersRef` 追加 |
| `platform-gitops/platform/applications/root-2-auth/*.yaml` | sync-wave: 1–6 → 10–15 |
| `platform-gitops/platform/applications/root-3-others/*.yaml` | sync-wave: 1–3 → 20–22、user-apps 2ファイルに wave 20 追加 |
| `platform-gitops/platform/group-roots/root.yaml` | 新規作成：単一ルート App-of-Apps |
| `platform-gitops/platform/group-roots/root-{1,2,3}-*.yaml` | 削除（単一ルート移行完了） |
| `platform-infra/k3d/Makefile` | fix-coredns に rewrite 追加・bootstrap-sync を単一ルート対応・port-forward 廃止・ArgoCD Healthy 待機追加 |
| `platform-gitops/platform/applications/external-secrets-config.yaml` | retry ポリシー追加・multi-source 化 |
| `platform-gitops/platform/applications/kyverno-policies.yaml` | retry ポリシー追加 |
| `platform-gitops/platform/applications/keycloak-db.yaml` | retry ポリシー追加・ExternalSecret ignoreDifferences 削除 |
| `platform-gitops/platform/applications/backstage-db.yaml` | retry ポリシー追加・ExternalSecret ignoreDifferences 削除 |
| `platform-gitops/platform/applications/*.yaml`（38 ファイル） | サブディレクトリから直下にフラット化 |
| `platform-gitops/platform/secrets/config/`（新規 10 ファイル） | ExternalSecret 集中管理 |
| `platform-gitops/platform/secrets/generators/` | GithubAccessToken を分離 |
| `platform-gitops/platform/secrets/namespaces/` | Namespace を分離 |
| `platform-gitops/platform/argocd/values.yaml` | redisSecretInit 無効化・CNPG/Gateway/Application ヘルスチェック追加 |
| `platform-gitops/platform/applications/argocd.yaml` | sync-wave: 20 → 5 |
| `platform-gitops/platform/applications/prometheus-operator-crds.yaml` | 新規作成：wave-1 CRD 専用 Application |
| `platform-gitops/platform/monitoring/values.yaml` | `crds.enabled: false` 追加 |
| `platform-infra/k3d/Makefile` | bootstrap: 重複定義削除・ターゲット実行順に並び替え・不要ターゲット削除 |
| `platform-infra/Makefile` | cluster-delete / cluster-status / generate-dr-manifests を追加 |
| `platform-gitops/platform/argocd/values.yaml` | HTTPRoute カスタムヘルスチェック追加 |
| `platform-gitops/platform/gateway/routes/platform/reference-grant-backstage.yaml` | 新規作成：backstage 用 ReferenceGrant |
