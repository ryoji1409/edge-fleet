# edge-fleet — Rancher Fleet サンプル（複数アプリ・複数edgeデバイス）

各edgeデバイス（自己完結クラスタ）を Rancher Fleet で GitOps 配布するためのサンプルリポジトリ。

前提:
- 各edgeデバイスは独立した k0s/k3s 単一ノードクラスタ
- Rancher に import 済み（例: edge01 / edge02）
- 全マニフェストは imagePullPolicy: IfNotPresent（オフライン耐性）
- 変化の可視化は ConfigMap の index.html を NodePort 30080 で確認

------------------------------------------------------------
## 事前準備: クラスタにラベルを付ける
------------------------------------------------------------
Fleet はラベルでターゲットを選ぶ。Rancher UI の各クラスタ → Edit Config → Labels、
または fleet の cluster リソースに付与する。例:

  edge01:  edge=true  node=edge01  tier=stable  role=inference
  edge02:  edge=true  node=edge02  tier=canary

(pattern によって使うラベルが異なる。下記参照)

------------------------------------------------------------
## Pattern 1: 全edgeデバイスに同一配布（最小構成）
------------------------------------------------------------
path: pattern1-all/apps/hello

GitRepo 設定:
  Paths:   pattern1-all/apps/hello
  Target:  clusterSelector matchLabels { edge: "true" }

確認:
  各edgeデバイスで: sudo k0s kubectl get deploy,cm hello-content
  curl http://<node-ip>:30080   → 全edgeデバイス同じ "deploy version: v1"

更新テスト:
  configmap.yaml の index.html を v2 に書き換えて push
  → 全edgeデバイスが自動で v2 に更新される

------------------------------------------------------------
## Pattern 2: edgeデバイス別の出し分け（targetCustomizations + overlays）
------------------------------------------------------------
path: pattern2-percluster/apps/hello
必要ラベル: node=edge01 / node=edge02

base(apps/hello直下) を全ターゲットに適用しつつ、
node ラベルに応じて overlays/edge01, overlays/edge02 を上書きする。

GitRepo 設定:
  Paths:   pattern2-percluster/apps/hello
  Target:  clusterSelector matchLabels { edge: "true" }

確認:
  edge01: curl http://<edge01-ip>:30080 → "I am edge01"（青背景）
  edge02: curl http://<edge02-ip>:30080 → "I am edge02"（桃背景）
  → 同一リポジトリからedgeデバイスごとに違う内容が配られる

------------------------------------------------------------
## Pattern 3: アプリ別Bundle × 役割グループ（実運用形）
------------------------------------------------------------
path: pattern3-grouped/apps
必要ラベル: edge=true / tier=stable|canary / role=inference

apps 配下の各アプリが独立Bundle（各々に fleet.yaml）。
各アプリの fleet.yaml 内 targets で「どのedgeデバイス群に配るか」を制御する。

  recorder   → edge=true     全edgeデバイス（録画/蓄積の常駐, 仕組み(3)相当）
  monitor    → tier=stable      stable群のみ（canaryedgeデバイスには配らない）
  inference  → role=inference   GPUedgeデバイス(例 Jetson)のみ

GitRepo 設定:
  Paths:   pattern3-grouped/apps
  Target:  （各アプリの fleet.yaml 側で制御するので、GitRepo側は広めに
           clusterSelector matchLabels { edge: "true" } でよい）

確認:
  edge01(stable,inference): recorder + monitor + inference が来る
  edge02(canary):           recorder のみ来る（monitor/inferenceは来ない）
  → 役割ラベルでアプリの配り分けができる

------------------------------------------------------------
## オフライン → 復帰の自動同期テスト（全パターン共通の主眼）
------------------------------------------------------------
1. 両edgeデバイスが同期済みなのを確認
2. edge02 をオフラインに（外部到達を断つ。default GW は残す）
3. オフライン中に Git の内容を変更して push（例: v2→v3、recorderのlog間隔変更）
   → edge01(オンライン) は更新される / edge02 は stale のまま（Rancher UIで確認）
4. edge02 をオンラインに戻す
   → fleet-agent が自動 pull、変更が適用される
5. edge02 で確認:
   sudo k0s kubectl get cm hello-content -o jsonpath='{.data.version}'

  ※ オフライン同期の検証は ConfigMap 書き換えが最適。
    （イメージ変更はオフラインで pull できず、Fleet同期とイメージ取得が混ざる）

------------------------------------------------------------
## トラブル時の確認
------------------------------------------------------------
  kubectl get gitrepo -n fleet-default
  kubectl get bundle -n fleet-default
  kubectl get bundledeployment -A
  kubectl describe gitrepo <name> -n fleet-default

よくある原因: paths のタイポ / clusterSelector がどのedgeデバイスにもマッチしない
（ラベル付け忘れ）/ YAML 不正
