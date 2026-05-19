# 開発環境セットアップ設計書

> **ステータス**: 作成中

---

## 1. 概要

このドキュメントは、Platform Engineering チームおよびアプリ開発チームが開発環境を構築する際のツール管理方針と初期化手順の設計を定める。

対象とする関心事は以下の通り。

- CLI ツールのバージョン管理（aqua）
- シェル環境の初期化（`make init`）
- アプリリポジトリへのツール設定の払い出し（Scaffolder）
- DR 復旧時の環境再構築手順

---

## 2. ツール管理（aqua）

### 2.1 ロール別の管理モデル

ツール管理に aqua（aquaproj）を採用している。  
PE チームとアプリチームではリポジトリ構成が異なるため、設定の共有方式を分けている。

**PE チーム（platform-xxx）**

`platform-infra/aqua.yaml` を唯一の source of truth とし、aqua の `AQUA_GLOBAL_CONFIG` 環境変数でグローバル参照する。

```
AQUA_GLOBAL_CONFIG=$HOME/platform-infra/aqua.yaml
```

この方式により、`platform-gitops` / `platform-charts` にはツール定義ファイルを一切置かなくてよい。  
ツールの追加・バージョン変更は `platform-infra/aqua.yaml` の 1 箇所で完結する。

`platform-gitops` / `platform-charts` にファイルを置かなくてよい理由は ArgoCD の制約にある。  
ArgoCD はセキュリティ上の理由でリポジトリ外を指すシンボリックリンクを `out-of-bounds symlinks` としてエラーにする。  
`AQUA_GLOBAL_CONFIG` はシェルレベルの設定であるため、この制約を回避できる。

| リポジトリ | ファイル | 備考 |
|---|---|---|
| `platform-infra` | `aqua.yaml` | source of truth |
| `platform-gitops` | なし | `AQUA_GLOBAL_CONFIG` から参照 |
| `platform-charts` | なし | 同上 |

**アプリチーム（sample-xxx / backstage）**

アプリチームのメンバーは `platform-infra` をクローンしないため `AQUA_GLOBAL_CONFIG` を設定できない。  
各リポジトリに `aqua.yaml` を自己完結で配置し、クローンしてすぐ使える状態にする。

基本セット（`kubectl` / `argocd` / `gh` / `oha`）を全アプリリポジトリに共通で定義し、アプリ固有のツールは各自が追加する。  
Backstage Scaffolder でアプリ払い出し時に基本セットの `aqua.yaml` を自動生成することで、新規アプリでも即座に使える状態を維持する（詳細は 4 章参照）。

| リポジトリ | ファイル | 内容 |
|---|---|---|
| `sample-backend` | `aqua.yaml` | 基本セット |
| `sample-frontend` | `aqua.yaml` | 基本セット |
| `backstage` | `aqua.yaml` | 基本セット + node |

---

## 3. シェル環境の初期化

### 3.1 make init による自動化方針（作成中）

---

## 4. アプリリポジトリへの aqua.yaml 払い出し

（作成中）

---

## 5. DR 復旧時の位置づけ

（作成中）
