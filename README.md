# holmesgpt-fsc-env

RHDP Field Sourced Content (FSC) 用の HolmesGPT デモ環境（Phase 0: インストール）。

## 目的

OpenShift 上に [HolmesGPT](https://holmesgpt.dev/) を Helm でデプロイし、LiteMaaS 経由の LLM で `/api/chat` が動くところまでを再現可能にする。

## 構成

```
helm/                         # App of Apps（Argo CD）
├── Chart.yaml
├── values.yaml               # 中央設定 + litemaas 受け皿
├── templates/applications.yaml
└── components/
    └── holmes-secrets/       # LiteMaaS → Secret
```

| コンポーネント | 内容 |
|----------------|------|
| `holmes-secrets` | FSC 注入の `litemaas.*` を Secret に載せる |
| `holmes`（外部 chart） | `robusta/holmes` を Argo CD が直接デプロイ |

含まないもの（意図的）:

- Showroom（後続）
- Ansible post-deploy
- OpenShift Route（必要になったら追加。公式 chart は Route を作らない）
- Holmes Operator / HealthCheck（Phase 1 以降）

## 前提

- RHDP Field Content CI（または同等の Argo CD + FSC workload）
- クラスタに OpenShift GitOps（`openshift-gitops`）
- LiteMaaS が FSC から `litemaas.apiUrl` / `apiKey` / `model` を注入できること

## 使い方（概要）

1. このリポジトリを Git に push する
2. `helm/values.yaml` の `gitops.repoUrl` を実リポジトリ URL に合わせる
3. RHDP で Field Content CI を注文し、このリポジトリ URL を指定する
4. Argo CD で Application が Sync されたら、smoke を実行する

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

### Smoke（クラスタ上）

```bash
# Service 名は release 名に依存。デフォルト想定: holmes / namespace holmesgpt
oc -n holmesgpt port-forward svc/holmes-holmes 8080:80

curl -sS -X POST http://localhost:8080/api/chat \
  -H 'Content-Type: application/json' \
  -d '{"ask":"list pods in namespace default","model":"litemaas"}'
```

`model` は `values.yaml` の `holmes.modelList` キー（デフォルト `litemaas`）と一致させる。

## Phase 計画（メモ）

| Phase | 内容 |
|-------|------|
| **0（本リポ）** | Holmes API + LiteMaaS + port-forward smoke |
| 1 | Holmes Operator + `HealthCheck`（`mode: monitor`、Slack なし） |
| 2 | Prometheus / Alertmanager 連携 |
| 3 | Route / Showroom（必要なら） |

## 参考

- [HolmesGPT Helm インストール](https://holmesgpt.dev/latest/installation/kubernetes-installation/)
- [OpenAI-Compatible（LiteMaaS 向け）](https://holmesgpt.dev/latest/ai-providers/openai-compatible/)
- 構成の先例: `rhsc2025-env`（App of Apps + `litemaas`）
