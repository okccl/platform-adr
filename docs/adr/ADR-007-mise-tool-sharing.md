# ADR-007: ツール定義の共有戦略（platform-infra を source of truth とする管理方針）

## Context

kubectl / helm / k3d / argocd / sops など、開発・運用に必要なツール群のバージョン管理に
mise を採用していた。これらは platform-infra・platform-gitops・platform-charts など
複数のリポジトリで共通して使用するため、ツール定義をどこで管理し、
他のリポジトリからどう参照するかという設計方針が必要になった。

IaC において「設定の一貫性はコードで担保する」ことを根本的な原則として捉えている。
以前の現場では Push 型パイプラインでの構成管理が原因で、担当者による直接編集が常態化し、
リポジトリと実環境の乖離が慢性化していた。ArgoCD の導入でその問題を解消した経験が
この方針の背景にある（詳細は ADR-001 参照）。

mise の設定についても同じ考え方を適用したい。
複数リポジトリが独立した `.mise.toml` を持つ構成では、
ツール追加時に全リポジトリへの手動反映が必要になり、sync 漏れが起きうる。
今回の文脈ではそれ自体が致命的な問題になるわけではないが、
IaC の根本的な考え方に反するため看過できないと判断した。

---

## Options

### Option 1: 各リポジトリに同内容の `.mise.toml` を置く

platform-infra の `.mise.toml` を source of truth とし、
他リポジトリには同一内容をコピーしてコメントで明示する。
ツール追加時は全リポジトリを手動更新する運用とする。

**問題点:** ツールの一貫性がコードではなく運用規律に依存する。
IaC の原則から見て筋が悪く、恒常的な解として採用したくない。

### Option 2: `~/.mise.toml` を `platform-infra/.mise.toml` へのシンボリックリンクにする

platform-infra の `.mise.toml` のみを git 管理し、
`~/.mise.toml → ~/platform-infra/.mise.toml` のシンボリックリンクを通じて
全ディレクトリからツールを参照する。他リポジトリには `.mise.toml` を置かない。

mise はディレクトリツリーを上位に向かってトラバースするため、
`~/.mise.toml` があれば全リポジトリで有効になる。

**問題点:** `~/.mise.toml` 自体は git 管理されないため、
DR 時にシンボリックリンク作成 `ln -sf ~/platform-infra/.mise.toml ~/.mise.toml` という
手動手順が 1 つ必要になる。

### Option 3: 各リポジトリに `../platform-infra/.mise.toml` へのシンボリックリンクを置く

各リポジトリの `.mise.toml` を `../platform-infra/.mise.toml` への
シンボリックリンクとする。シンボリックリンク自体は git 管理される。

**問題点:** ArgoCD が管理するリポジトリ（platform-gitops / platform-charts）では、
リポジトリ外を指すシンボリックリンクに対して
`repository contains out-of-bounds symlinks` エラーが発生し使用できない。
また mise には `extends` のような機能もなく、設定ファイルレベルでの継承も不可。

### Option 4: GitHub Actions でツール定義を他リポジトリに配布する

platform-infra の `.mise.toml` 更新時に CI が他リポジトリへ PR を自動作成して配布し、
他リポジトリでの直接編集 PR を CI でブロックする。

**問題点:** オーバーエンジニアリング。
ツールバージョン変更という低頻度の作業に対して、
クロスリポジトリ CI の仕組みを構築・維持するコストは割に合わない。

### Option 5: aqua の `AQUA_GLOBAL_CONFIG` を使う

mise から aqua（aquaproj）に移行し、`AQUA_GLOBAL_CONFIG` 環境変数で
`platform-infra/aqua.yaml` を全ディレクトリからグローバル参照する。

aqua は `AQUA_GLOBAL_CONFIG` に指定したファイルをシェルレベルで読み込むため、
各リポジトリにツール定義ファイルを置く必要がない。
ツール追加は `platform-infra/aqua.yaml` の 1 箇所で完結し、
`platform-gitops` / `platform-charts` には一切ファイルを置かなくてよい。

また aqua は SLSA / Cosign によるバイナリの署名検証を標準サポートしており、
mise に比べてセキュリティ面でも優位性がある。

**アプリリポジトリの扱い:**
アプリチーム（sample-backend / sample-frontend / backstage）は platform-infra をクローンしないため
`AQUA_GLOBAL_CONFIG` を設定できない。これらは各リポジトリに `aqua.yaml` を自己完結で置く。
Backstage Scaffolder でアプリ払い出し時に基本ツールセットの `aqua.yaml` を同梱することで
「クローンしてすぐ使える」状態を維持する。

---

## Decision

**Option 5（aqua の `AQUA_GLOBAL_CONFIG`）を採用する。**

---

## Reasons

1. **Option 3 は ArgoCD の制約により選択不可だった**

   管理コストが最も低いのはリポジトリにシンボリックリンクを置く方法だったが、
   ArgoCD がセキュリティ上の理由でリポジトリ外を指すシンボリックリンクをエラーにするため断念した。
   `extends` による設定継承も現バージョンでは未対応。
   これらの制約により、git 管理されたシンボリックリンクで問題を解決する方法は現時点で存在しない。

2. **Option 5 は Option 2 で残っていた手動手順すら不要にする**

   Option 1 の暫定採用後、Option 2（`~/.mise.toml` シンボリックリンク）への移行を検討していたが、
   aqua の `AQUA_GLOBAL_CONFIG` はシェルプロファイルへの 1 行追加で同等の効果を得られる。
   シンボリックリンク作成という手動手順も不要になるため、Option 2 より優れた解となる。

3. **ツール管理を 1 つに統一できる**

   mise と aqua を併用する状態を避けるため、言語ランタイム（node）も含めて aqua に統一した。
   ツールの追加・バージョン変更の操作先が 1 ツールに絞られる。

4. **Option 4 はコストに見合わない**

   ツールバージョン変更の頻度と影響範囲に対して、
   クロスリポジトリの CI/PR 制御の仕組みを構築・維持するコストは明らかに過剰。

---

## Consequences

**PE チーム（platform-xxx）:**

- `platform-infra/aqua.yaml` が唯一の source of truth
- `platform-gitops` / `platform-charts` にはツール定義ファイルを置かない
- ツール追加・バージョン変更は `platform-infra/aqua.yaml` のみ更新すればよい
- シェルプロファイルに `AQUA_GLOBAL_CONFIG=$HOME/platform-infra/aqua.yaml` を設定する（DR 手順に記載）

**アプリチーム（sample-xxx / backstage）:**

- 各リポジトリに `aqua.yaml` を自己完結で配置
- `kubectl` / `argocd` / `gh` / `oha` を基本セットとし、アプリ固有ツールを各自追加できる
- Backstage Scaffolder でアプリ払い出し時に `aqua.yaml` を自動生成する（別途設計書参照）

**mise との関係:**

- mise は完全廃止。全リポジトリから `.mise.toml` を削除済み
- `~/.bashrc` の `mise activate` を削除し、aqua の PATH 設定に置き換え済み
