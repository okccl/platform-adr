# リソース管理設計書

---

## 1. 概要

本ドキュメントは、Platform Engineering チームとして platform-gitops リポジトリ上のリソースをどの ArgoCD Application が管理するかの方針を定める。

基本原則は**「リソースはそのオーナーが管理する」**。リソースが属する関心領域（Gateway の設定か、App の namespace か）によって管理 App を決定する。リソースを誤った App に配置すると、ArgoCD の SharedResourceWarning・wave 順序の乱れ・責任境界の不明確化を引き起こすため、新規リソースを追加する際は本設計に照らして配置先を判断すること。

---

## 2. Gateway API リソース

### 2.1 HTTPRoute

**管理 App：`gateway-routes`**

HTTPRoute は「Gateway がどのトラフィックをどの Service に転送するか」を定義する、Gateway 側の設定である。バックエンドの App がどの namespace にあるかに関係なく、ルーティングルールはすべて `gateway-routes` App が一元管理する。

| 配置先 | namespace |
|---|---|
| `platform/gateway/routes/platform/` | envoy-gateway-system |

### 2.2 ReferenceGrant

**管理 App：各バックエンド App**

ReferenceGrant は「自分の namespace へのクロス namespace 参照を許可する」という、namespace オーナーによる許可宣言である。Gateway API の設計上、許可は**参照される側**が発行する。したがって ReferenceGrant は対象 App の namespace に属し、その App の ArgoCD Application が管理する。

各 App の `app-config/reference-grant.yaml` に配置し、`sync-wave: "-1"` を付与して App 内で最初に作成されるようにする。

| App | ReferenceGrant namespace |
|---|---|
| `kube-prometheus-stack` | monitoring |
| `goldilocks` | goldilocks |
| `argo-rollouts` | argo-rollouts |
| `keycloak` | keycloak |
| `backstage` | backstage |

### 2.3 設計上の補足

HTTPRoute の health check は `ResolvedRefs`（バックエンド参照の解決）を確認しない（`platform/argocd/values.yaml` のカスタムヘルスチェックを参照）。このため、ReferenceGrant が HTTPRoute より後の wave で作成される場合でも wave の進行は止まらない。routing は両リソースが揃った時点で正常に動作する。

---

## 3. （以降追記予定）

README 整備フェーズで他のリソース種別（Secret / RBAC / NetworkPolicy 等）の管理方針を順次追加する。
