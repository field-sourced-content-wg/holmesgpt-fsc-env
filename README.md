# holmesgpt-fsc-env

RHDP Field Sourced Content (FSC) 用の HolmesGPT デモ環境。

## 目的

OpenShift 上に [HolmesGPT](https://holmesgpt.dev/) を Helm / Argo CD でデプロイし、LiteMaaS 経由の LLM・Operator HealthCheck・Prometheus 経由の firing alerts 検証・Holmes CLI 付き Web Terminal までを再現可能にする。

## 構成

```
helm/                              # App of Apps（Argo CD）
├── Chart.yaml
├── values.yaml                    # 中央設定 + litemaas 受け皿
├── templates/applications.yaml
└── components/
    ├── holmes-secrets/            # LiteMaaS → Secret
    ├── holmes-route/              # OpenShift Route → Holmes Service
    ├── holmes-healthcheck/        # HealthCheck (mode: monitor)
    ├── web-terminal-operator/     # WTO Subscription → openshift-operators
    └── holmes-web-terminal/       # ImageStream + BuildConfig
container/holmes-web-terminal/     # Dockerfile + holmes CLI helpers
```

| コンポーネント | 内容 |
|----------------|------|
| `holmes-secrets` | FSC 注入の `litemaas.*` を Secret に載せる |
| `holmes`（外部 chart） | `robusta/holmes`（API + **Operator** + prometheus toolset） |
| `holmes-route` | Service `holmes-holmes` への Route（edge TLS） |
| `holmes-healthcheck` | `firing-alerts-probe`（`mode: monitor`）— Prometheus で firing が見えるかの検証用 |
| `web-terminal-operator` | Web Terminal Operator Subscription（**別 App** → `openshift-operators`） |
| `holmes-web-terminal` | Holmes CLI 入り Web Terminal 用イメージを BuildConfig でビルド |

含まないもの（意図的）:

- Showroom（後続）
- Ansible post-deploy
- Slack / `mode: alert` destinations
- クラスタ全体の Web Terminal 既定イメージ差し替え（共有 RHDP 向けに無効）

## 前提

- RHDP Field Content CI（または同等の Argo CD + FSC workload）
- クラスタに OpenShift GitOps（`openshift-gitops`）
- LiteMaaS が FSC から `litemaas.apiUrl` / `apiKey` / `model` を注入できること
- プラットフォーム Monitoring（Thanos Querier）が利用可能であること（prometheus toolset 用）
- Web Terminal Operator は GitOps で入れます（`webTerminalOperator.enabled`）。既にクラスタにある場合も同名 Subscription を追従する想定です

## 使い方（概要）

1. このリポジトリを Git に push する
2. `helm/values.yaml` の `gitops.repoUrl` を実リポジトリ URL に合わせる
3. RHDP で Field Content CI を注文し、このリポジトリ URL を指定する
4. Argo CD で Application が Sync されたら、下記の検証を実行する

### ローカル検証（クラスタ不要）

```bash
cd helm
helm lint .
helm template holmesgpt-fsc . \
  --set deployer.domain=apps.example.com \
  --set litemaas.apiKey=dummy \
  --set litemaas.apiUrl=https://litemaas.example.com/v1 \
  --set litemaas.model=gpt-4o-mini
```

### Smoke — HTTP API（Route）

```bash
HOST=$(oc -n holmesgpt get route holmes -o jsonpath='{.spec.host}')

curl -sS -X POST "https://${HOST}/api/chat" \
  -H 'Content-Type: application/json' \
  -d '{"ask":"list pods in namespace default","model":"litemaas"}'
```

`model` は `values.yaml` の `holmes.modelKey`（デフォルト `litemaas`）と一致させる。

### 検証 — HealthCheck（Prometheus firing alerts）

Operator 有効 + `prometheus/metrics`（Thanos Querier）+ サンプル HealthCheck がデプロイされます。

```bash
# Operator / CRD
oc get crd | grep holmesgpt.dev
oc -n holmesgpt get pods -l app.kubernetes.io/name=holmes-operator

# プローブ結果（mode: monitor → Slack なし。status に残る）
oc -n holmesgpt get hc firing-alerts-probe
oc -n holmesgpt describe hc firing-alerts-probe

# 再実行
oc -n holmesgpt annotate hc firing-alerts-probe holmesgpt.dev/rerun=true --overwrite
```

確認したいこと:

- Prometheus クエリが成功すること（firing 件数は 0 でも >0 でも **pass** が正しい）
- AlertManager API 専用 CLI と同等ではないこと（見えるのは Prometheus 側）

再実行: `oc -n holmesgpt annotate hc firing-alerts-probe holmesgpt.dev/rerun=true --overwrite`

※ 以前の query は「アラートがある＝fail」と弱モデルが早合点しやすかったため、
  「ツール成功＝pass」の検証用 query に変更済み。

### Web Terminal + Holmes CLI

1. **Operator**（別 Argo CD App → `openshift-operators`）

```bash
oc -n openshift-operators get sub web-terminal
oc -n openshift-operators get csv | grep -i web-terminal
```

2. **イメージビルド**（`holmesgpt` NS の BuildConfig / ImageStream）

```bash
oc -n holmesgpt get is holmes-cli-terminal   # TAGS に latest
oc -n holmesgpt get rolebinding holmes-cli-terminal-openshift-terminal-puller
```

GitOps が `system:image-puller` を `system:serviceaccounts:openshift-terminal` に付与します  
（無いと `ImagePullBackOff` / `authentication required`）。

3. コンソールで Web Terminal を開く。**Start の前に Image** を押し、次を指定:

   `image-registry.openshift-image-registry.svc:5000/holmesgpt/holmes-cli-terminal:latest`

   以前 Failed した端末が残っている場合:

```bash
oc -n openshift-terminal delete dw --all
```

   Image 欄が無い / 全ユーザー既定にする場合:  
   Administrator → Cluster Settings → Configuration → Console → Customize → Web Terminal

4. ターミナル内:

```bash
source holmes-demo-env
holmes ask "list pods in namespace holmesgpt" --model=openai/${LITEMAAS_MODEL}
# CLI 経路（AlertManager API）。CronJob ではない。
holmes investigate alertmanager --alertmanager-url "${ALERTMANAGER_URL}"
```

OpenShift AlertManager が Bearer 必須で CLI が弾く場合は、upstream どおり port-forward して `http://localhost:9093` を使う（[Investigating Prometheus Alerts](https://holmesgpt.dev/latest/walkthrough/investigating-prometheus-alerts/)）。

クラスタ全体の `web-terminal-tooling` DevWorkspaceTemplate は **変更しない**（`patchClusterTemplate: false`）。共有環境向け。

## Phase 計画（メモ）

| Phase | 内容 |
|-------|------|
| **0** | Holmes API + LiteMaaS + Route |
| **1（本リポ）** | Operator + HealthCheck `monitor` + Prometheus toolset + CLI Web Terminal イメージ |
| 2 | AlertManager 認証の詰め / ScheduledHealthCheck（必要なら） |
| 3 | Showroom（必要なら） |

## 参考

- [HolmesGPT Helm インストール](https://holmesgpt.dev/latest/installation/kubernetes-installation/)
- [Holmes Operator](https://holmesgpt.dev/latest/operator/)
- [Health Checks](https://holmesgpt.dev/latest/operator/health-checks/)
- [Investigating Prometheus Alerts（CLI）](https://holmesgpt.dev/latest/walkthrough/investigating-prometheus-alerts/)
- [OpenAI-Compatible（LiteMaaS 向け）](https://holmesgpt.dev/latest/ai-providers/openai-compatible/)
- [OpenShift Web Terminal](https://docs.redhat.com/en/documentation/openshift_container_platform/4.15/html/web_console/web-terminal) / [KCS 6962984（custom image）](https://access.redhat.com/solutions/6962984)
- 構成の先例: `rhsc2025-env`（App of Apps + `litemaas`）
