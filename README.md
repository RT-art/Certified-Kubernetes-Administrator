<div align="center">

# 🎓 Certified Kubernetes Administrator (CKA)

CKA合格のための勉強メモ

</div>

## 📦 Pod

### ノードの確認
```bash
kubectl get pods -o wide
```

### 出力オプション（-o）
- **yaml** → YAML形式で全情報を出力
- **json** → JSON形式で出力  
- **wide** → 表形式のまま、情報を広げて表示

> **wideの表示内容**
> - Podがどのノードに配置されているか
> - PodのIPアドレス
> - コンテナのイメージ

### YAML出力
```bash
kubectl run redis --image=redis123 --dry-run -o yaml
```
> ⚠️ このコマンドは非推奨です

### dry-runの種類
- **client** → ローカルで雛形生成・形だけ確認
- **server** → クラスタで本番と同じ検証（ただし保存しない）

### ファイル作成の流れ
```bash
# ターミナルにYAMLを出力
kubectl run redis --image=redis123 --dry-run=client -o yaml

# 実際にファイルを作成
kubectl run redis --image=redis123 --dry-run=client -o yaml > redis.yaml
cat redis.yaml
kubectl create -f redis.yaml
```

### Podの修正方法
#### マニフェストファイルで編集
```bash
vi redis.yaml
kubectl apply -f redis.yaml
```

#### コマンドで直接修正
```bash
kubectl edit
```

## 🔄 ReplicaSet

### 基本的なコマンド
```bash
# ReplicaSetの数を確認
kubectl get replicaset

# 詳細内容を確認
kubectl describe replicaset new-replicaset

# ファイルから作成
kubectl create -f /root/replicaset.yaml

# 省略形
kubectl get rs
```

### kubectl explain
Kubernetesリソースの仕様・フィールドの意味を教えてくれるコマンド
```bash
kubectl explain replicaset
```
> 💡 **覚え方**: YAML関連は `explain`、コマンド忘れたら `help`

### APIバージョンの違い

#### `apiVersion: v1`
KubernetesのコアAPIグループ
- Pod
- Service
- ConfigMap
- Secret
- Namespace
- など

#### `apiVersion: apps/v1`
より「高度なオブジェクト（コントローラ系）」
- Deployment
- StatefulSet
- DaemonSet
- ReplicaSet
- など

### ReplicaSetの仕組み
ReplicaSetは「**特定のラベルを持つPodを監視して、指定した数（replicas）だけ存在するようにする**」コントローラです。

#### 重要な2つの要素
1. **spec.selector** → どんなPodを管理するか条件を定義
2. **spec.template** → その条件に合うPodを自分で作るときのテンプレート

> **ポイント**:
> - セレクターは、今から作られるものだけでなく、既存のPodも管理対象に入れられる
> - テンプレートは、作った瞬間にセレクターの管理配下になる
> - `spec` = そのリソースの望ましい状態

### スケーリング
```bash
kubectl scale rs new-replica-set --replicas=5
```

## 🚀 Deployment

### 階層構造
```
Deployment
   └── ReplicaSet
         └── Pod
```

### 各コンポーネントの役割
- **Deployment** → 「アプリケーションをこういう形で動かしたい（ローリングアップデートで管理してね）」って仕様を宣言するコントローラ
- **ReplicaSet** → 「Podを常にn個保つ」役割。Deploymentの配下で、実際にPodを維持する
- **Pod** → 実際にコンテナが動く最小単位

### 基本的なコマンド
```bash
kubectl get deploy
kubectl get all
```

### 参考リンク
> 📚 **ブックマーク**: [kubectl conventions](https://kubernetes.io/docs/reference/kubectl/conventions/)

### Deployment作成
```bash
kubectl create deployment --image=nginx nginx --replicas=4 --dry-run=client -o yaml > nginx-deployment.yaml
```

> 💡 **コツ**: YAMLは基本runコマンドで生成したものを修正して使う

## 🌐 Service

### NodePort
**ノードポート** = 外部接続に使われるノード自体のポート

#### 接続の流れ
```
ノードポート → サービス → Pod
```

#### ポートの関係
- **targetPort** → Podのポート
- **port** → サービスのポート

#### NodePortのYAML例
```yaml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: nginx   # 管理対象Podのラベル
  ports:
    - targetPort: 80    # Podのポート
      port: 80          # サービスのポート
      nodePort: 30080   # ノードの外部からアクセスするポート (30000-32767)
```

### ClusterIP
PodのIPは頻繁に変わるため、グループで抽象化する仕組みがClusterIPです。

### LoadBalancer
外部からの入り口として使用します。

#### NodePortの問題点
`type: NodePort`にすると、クラスタの全Nodeで同じポート番号（例:30080）が開きます。

**アクセス方法**:
```
http://<Node1のIP>:30080
http://<Node2のIP>:30080
http://<Node3のIP>:30080
```

> ⚠️ **問題**: 「ノードの数だけ外部からの入口URL」ができてしまい、利用者からすると「どのノードにアクセスすればいいの？」という問題が発生

#### LoadBalancerの解決策
`type: LoadBalancer`を使うと、クラウド（AWS/GCP/Azureなど）が**外部ロードバランサ**を自動で立ち上げてくれます。

- 外部ユーザーは**1つの固定のエンドポイント（VIPやDNS名）**にアクセスするだけでOK
- そのLBがクラスタ内のNodePortに分散して、さらにPodに流してくれる

### Service確認
```bash
kubectl get svc
```

## 🏷️ Namespace

### 基本的なコマンド
```bash
# Namespace一覧
kubectl get ns

# 特定のNamespaceのPodを確認
kubectl get pods --namespace=<namespace>
kubectl get pods -n=<namespace>

# 全NamespaceのPodを確認
kubectl get pods --all-namespaces
kubectl get pods -A
```

## ⚡ 命令形コマンド

### ラベル付きPod作成
```bash
kubectl run redis --image=redis:alpine --labels="tier=db"
```

### サービスを命令形で作成（expose）
```bash
kubectl run httpd --image=httpd:alpine --port=80 --expose=true
```

### 参考リンク
> 📚 **ブックマーク**: [kubectl commands](https://kubernetes.io/docs/reference/generated/kubectl/kubectl-commands)

## 📋 スケジューラ

### スケジューラとは
どのPodをどのNodeに配置するかを決定するコンポーネントです。

### 動作の流れ
1. ユーザーがPodを作成
2. 最初は「どのNodeにも割り当てられていない状態（Pending）」
3. スケジューラが働いて、クラスタ内の適切なNodeを選択
4. そのNodeにPodを割り当て

### スケジューラの確認
```bash
kubectl get pods -n kube-system
```

### 手動スケジューリング
YAMLで`nodeName`を指定することで、特定のノードにPodを配置できます：

```yaml
spec:
  nodeName: worker-node-1
```

### Podの再作成
YAMLを編集したら、Podは削除して再作成しなければいけませんが、以下のコマンドで再作成できます：

```bash
kubectl replace --force nginx.yaml
```

### Podの状態監視
```bash
kubectl get pods --watch
```
> 💡 **用途**: PodがRunningになるまで監視

## 🏷️ ラベル（Labels）

### ラベルとは
ラベルは**キー=バリュー**の形式で、リソースにメタデータを付与する仕組みです。

### ラベルでの検索
```bash
# セレクターを使用
kubectl get pods --selector=env=dev

# 短縮形
kubectl get pods -l=env=dev
```

### 複数ラベルの指定
```bash
kubectl get pod -l=env=prod,bu=finance,tier=frontend
```

### ラベルでカウント
```bash
kubectl get pods --selector=env=dev --no-headers | wc -l
```
> ⚠️ **重要**: `--no-headers`を必ず入れる。入れないと一行目のヘッダーもカウントされてしまう

### ReplicaSetのラベル設定
ReplicaSetのYAML内では、**2つの場所でラベルを定義**する必要があります：

```yaml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: replicaset-1
spec: # <----ここが、replicasetの定義
  replicas: 2
  selector:
    matchLabels:
      tier: nginx # レプリカセットのラベル（管理対象の条件）
  template: # <----ここから、podの定義
    metadata: 
      labels:
        tier: nginx # podのラベル（実際に作成されるPodに付与）
    spec:
      containers: 
      - name: nginx
        image: nginx
```

> 💡 **ポイント**: 
> - `spec.selector.matchLabels` → 管理対象のPodを特定する条件
> - `spec.template.metadata.labels` → 実際に作成されるPodに付与されるラベル
> - この2つのラベルは**一致している必要がある**

## 🚫 TaintとToleration

### 基本概念
- **Taint** = ノード側のルール「このノードは普通のPodはダメ」
- **Toleration** = Pod側の許可証「私はこのルールを無視してもOK」

### Taintの設定
```bash
kubectl taint nodes node1 dedicated=experimental:NoSchedule
kubectl taint node node01 spray=mortein:NoSchedule
```

### Taintの削除
```bash
kubectl taint node node01 spray=mortein:NoSchedule-
```

### Tolerationの設定
```yaml
tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "experimental"
  effect: "NoSchedule"
```

> 💡 **注意**: TolerationはYAMLでしか設定できません

### Operatorの種類
- **Equal** → valueまで一致する必要あり（厳格）
- **Exists** → keyが一致すればOK（ゆるめ）

> ⚠️ **重要**: Existsのときにvalueを書くとエラーになります

### 便利なコマンド
```bash
# taint設定のみ抜き出す
kubectl describe node node1 | grep taints
```

> 💡 **vi操作**: `Shift + V` → `>` で一気にインデントできます

## 🎯 Node Selector＆Node Affinity

### 基本概念
Pod側が「ここに配置したい」と希望する仕組みです。Taintだけでは禁止のみで、他のノードに配置されてしまう可能性があるため、積極的に配置先を指定する必要があります。

### Node Affinity（必須条件）
```yaml
# 必ず「zone=us-east-1a」のノードに置きたい
affinity:
  nodeAffinity:
    requiredDuringSchedulingIgnoredDuringExecution: # 必須条件を表す
      nodeSelectorTerms:
      - matchExpressions:
        - key: zone
          operator: In
          values:
          - us-east-1a
```

### Node Affinity（優先条件）
```yaml
# 「disk=ssd」があればそっちに置いてほしいけど、なければ他でもOK
affinity:
  nodeAffinity:
    preferredDuringSchedulingIgnoredDuringExecution: # 優先条件（あれば嬉しい）
    - weight: 1
      preference:
        matchExpressions:
        - key: disk
          operator: In
          values:
          - ssd
```

### Operator一覧
- **In** → 指定した値のいずれかに一致する
- **NotIn** → 指定した値には含まれない
- **Exists** → keyがあればマッチ
- **DoesNotExist** → keyがなければマッチ

### NodeSelector vs NodeAffinity

#### NodeSelector（古い方式）
```yaml
nodeSelector:
  disktype: ssd
```
> シンプルだけど柔軟性がない

#### NodeAffinity（上位互換）
- NodeSelectorの上位互換
- 複数条件やIn/NotIn/Existsが使える

### 組み合わせによる制御
- **Node Affinity** → Podが「ここに置きたい！」と希望する
- **Taint/Toleration** → ノードが「俺は特殊だから勝手に来るなよ」と制限する

> 💡 **効果**: 両方組み合わせると「特定のPodだけ特定ノードに入る」という厳密な制御ができる

## 📊 Resource Limit

### メモリ不足の症状
- **OOMKilled** → メモリ不足でPodが強制終了

### 実行中のPodのリソース変更
```bash
kubectl edit pod elephant
# エラーが発生する場合
error: pods "elephant" is invalid
A copy of your changes has been stored to "/tmp/kubectl-edit-2855789753.yaml"
error: Edit cancelled, no valid changes were saved.

# 保存されたYAMLを使って強制置換
kubectl replace --force -f /tmp/kubectl-edit-2855789753.yaml
```

> 💡 **ポイント**: editでメモリの値を増やすのは実行中のため不可。しかし、edit失敗してもYAMLが吐き出されるため、それを使ってreplace可能

## 🔄 DaemonSet

### DaemonSetとは
「クラスタ内のすべてのノード、または特定の条件を満たすノードに、必ず1つずつPodを配置する仕組み」

> **用途**: ノードごとに必ず必要になるエージェント系のワークロード

### DaemonSetのYAML例
```yaml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: log-agent
  namespace: kube-system
spec:
  selector:
    matchLabels:
      app: log-agent
  template:
    metadata:
      labels:
        app: log-agent
    spec:
      containers:
      - name: log-agent
        image: fluentd:latest
```

### DaemonSetの特徴
- Deploymentに似ているけど、**replicasは指定しない**
- **Pod数はノード数によって自動的に決まる**
- ノードが増えると自動でそのノードにもPodが作られる
- ノードが削除されれば、そのPodも消える

### 特定ノードのみに配置
- **NodeSelector / NodeAffinity** を組み合わせる
- **Tolerations** を組み合わせてtaintされたノード専用にする

### 用途例
- クラスタ内全てのノードに、一つのPodをつける
- 監視エージェントなど

### 確認コマンド
```bash
# 全ての名前空間を確認
kubectl get daemonsets -A
```
> 💡 **注意**: `kubectl get daemonsets`だけだと、デフォルト名前空間のみ

## ⚙️ Static Pod

### Static Podとは
**kubeletが直接管理するPod**。API Server / kube-schedulerを経由せず、そのノード上のkubeletによって起動・監視される特殊なPodです。

> **kubelet** = Kubernetesの各ノード上で動くエージェントプロセス。APIサーバから渡された指示を受けて、実際にコンテナを起動させる役割

### マニフェスト配置場所
```
/etc/kubernetes/manifests/
```

### Static Podの特徴
- **削除方法が特殊**: `kubectl delete pod`しても消えない（kubeletが再作成する）
- **YAMLファイルを消すと停止する**
- **マスターコンポーネントで利用**: Kubernetes自体のコア（kube-apiserver, etcd, controller-manager, scheduler）はStatic Podとして起動されることが多い
- **マニフェスト修正即反映**: ディレクトリ内のファイルを修正すると、自動的にPodが再作成される

### トラブル対応
> 💡 **重要**: トラブル対応（etcd落ちた、APIサーバが起動しない）のときに知っておくと役立つ

### 確認方法
```bash
# 末尾にノード名のPodがStatic
kubectl get pods -A

# Static Podのパス確認
cat /var/lib/kubelet/config.yaml
# staticPodPath: /etc/just-to-mess-with-you
```

### 強制削除
```bash
# Terminatingのまま消えない時
kubectl delete pod POD名 --grace-period=0 --force
```

### コマンド実行
```bash
--command --sleep 1000
```

## 🏆 Pod Priority

### Pod Priorityとは
Podに「優先度」を与える仕組み。クラスタ内のリソースが足りなくなったとき、どのPodを優先するかを決める仕組みです。

### PriorityClassの定義
```yaml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "This priority class should be used for critical pods only."
```

### 優先度の仕組み
- **value**: 優先度（大きいほど優先される）
- **ケジューリング時**: 高優先度Podは、低優先度Podよりも先にスケジューリングされる

### プリエンプション（追い出し）
ノードに空きがなくても、高優先度Podが来ると、低優先度Podが強制削除されて場所を譲ることがあります。

> **Preemption** = プリエンプション（追い出し）

### 特徴
- **名前空間を持たない**

### 確認コマンド
```bash
kubectl get priorityclass
kubectl get pods -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priorityClassName"
```

## 🔀 複数のスケジューラ

### 基本概念
普段は、デフォルトのスケジューラ（kube-scheduler）がいい感じにやってくれていますが、クラスタに複数のスケジューラをデプロイできます。

> **用途**: 独自のスケジューラアルゴリズムで、ノードにPodを配置したいとき

### カスタムスケジューラの指定
Podのspecに、スケジューラ名を書けば、複数のスケジューラを同時に動かせます：

```yaml
schedulerName: スケジューラ名
```

## 🛡️ Admission Controller

### 基本概念
KubernetesでPodやDeploymentを作るとき、APIサーバーにリクエストが届きます。APIサーバーは以下の流れで処理します：

1. リクエストを受け取る
2. 認証/認可チェック（この人操作していい？）
3. etcdに保存

この途中に「チェックマン」や「修正マン」を挟めるのが**Admission Controller**です。

### Admission Controllerとは
「リクエストを受けたときに、クラスタに保存される前にチェックしたり書き換えたりする仕組み」

### 2つのタイプ

#### Validating（検証系）
「このPod、セキュリティポリシー違反してない？」

**例**: root権限で動かそうとしていたら「ダメ！」と拒否する

#### Mutating（書き換え系）
「設定が足りないから自動で補っておくね」

**例**: Podにデフォルトのsidecar（ログ収集コンテナ）を勝手に追加する

### 役割
- **APIサーバーにリソースが保存される前のゲートキーパー**
- 役割は「チェック（Validating）」と「自動修正（Mutating）」

> 💡 **注意**: Admission Controllerは認証は何もしません

### 確認コマンド
```bash
# 有効化されているAdmission Controllerを確認
kube-apiserver -h | grep enable-admission-plugins

# 有効化されていないAdmission Controllerを調べる
kubectl get pods -n kube-system
kubectl exec -it kubeapiserver-controlplane -n kube-system -- kube-apiserver -h | grep 'enable-admission-plugins'
```
exec: 実行中のPod内でコマンドを実行する
### 各部分の説明

#### 1. `kubectl exec -it`
- **exec**: 実行中のPod内でコマンドを実行する
- **-it**: インタラクティブ（-i）でターミナル（-t）を割り当てる
  - 標準入出力を接続して、コマンドの結果を表示

#### 2. `kubeapiserver-controlplane`
- **対象のPod名**: kube-apiserverが動いているPod
- このPod内でコマンドを実行する

#### 3. `-n kube-system`
- **namespace指定**: kube-system名前空間内のPodを対象とする
- kube-apiserverは通常kube-system名前空間で動いている

#### 4. `--`
- **区切り文字**: kubectlのオプションと、Pod内で実行するコマンドを分ける
- `--`以降はPod内で実行されるコマンド

#### 5. `kube-apiserver -h`
- **kube-apiserver**: Kubernetes APIサーバーのバイナリ
- **-h**: ヘルプオプション（--helpと同じ）
- 利用可能なオプション一覧を表示

#### 6. `| grep 'enable-admission-plugins'`
- **パイプ**: 前のコマンドの出力を次のコマンドに渡す
- **grep**: 指定した文字列を含む行のみを抽出
- **'enable-admission-plugins'**: Admission Controllerの有効化オプションを検索

## 🎯 このコマンドの目的

### 何を調べているか
- **有効化されているAdmission Controller**を確認
- kube-apiserverがどのAdmission Controllerを使用しているかを調べる

### なぜこの方法を使うか
- kube-apiserverは**Static Pod**として動いている
- 設定ファイルを直接見るよりも、実際に動いているプロセスのオプションを確認する方が確実

## 📋 実行結果の例

```bash
--enable-admission-plugins strings
                          admission plugins that should be enabled in addition to default enabled ones (NamespaceLifecycle, LimitRanger, ServiceAccount, TaintNodesByCondition, Priority, DefaultTolerationSeconds, DefaultStorageClass, StorageObjectInUseProtection, PersistentVolumeClaimResize, MutatingAdmissionWebhook, ValidatingAdmissionWebhook, ResourceQuota). Comma-delimited list of admission plugins: AlwaysAdmit, AlwaysDeny, AlwaysPullImages, DefaultStorageClass, DefaultTolerationSeconds, DenyEscalatingExec, DenyExecOnPrivileged, EventRateLimit, ExtendedResourceToleration, ImagePolicyWebhook, Initializers, LimitPodHardAntiAffinityTopology, LimitRanger, MutatingAdmissionWebhook, NamespaceAutoProvision, NamespaceExists, NamespaceLifecycle, NodeRestriction, OwnerReferencesPermissionEnforcement, PersistentVolumeClaimResize, PersistentVolumeLabel, PodNodeSelector, PodPreset, PodSecurityPolicy, PodTolerationRestriction, Priority, ResourceQuota, SecurityContextDeny, ServiceAccount, StorageObjectInUseProtection, TaintNodesByCondition, ValidatingAdmissionWebhook. The order of plugins in this flag does not matter.
```

## 💡 実用的な使い方

### トラブルシューティング
```bash
# 特定のAdmission Controllerが有効かどうか確認
kubectl exec -it kubeapiserver-controlplane -n kube-system -- kube-apiserver -h | grep -A 5 -B 5 'PodSecurityPolicy'
```

### 設定確認
```bash
# 現在のAdmission Controller設定を確認
kubectl get pod kubeapiserver-controlplane -n kube-system -o yaml | grep -A 10 -B 10 'enable-admission-plugins'
```

## 🔧 関連コマンド

```bash
# より詳細な情報を取得
kubectl exec -it kubeapiserver-controlplane -n kube-system -- kube-apiserver -h | grep -A 20 'enable-admission-plugins'

# 無効化されているAdmission Controllerも確認
kubectl exec -it kubeapiserver-controlplane -n kube-system -- kube-apiserver -h | grep 'disable-admission-plugins'
```

このコマンドは、Kubernetesクラスタの**セキュリティポリシー**や**リソース制限**がどのように設定されているかを調べる際に非常に有用

## 🔍 kube-apiserverとAdmission Controllerの関係

### kube-apiserver内にAdmission Controllerがある理由

#### 1. **アーキテクチャ上の理由**
```
API Request → kube-apiserver → Admission Controller → etcd
```

- **kube-apiserver**は全てのAPIリクエストの**入り口**
- リクエストがetcdに保存される**直前**でチェック・修正が必要
- そのため、kube-apiserver**内に**Admission Controllerが組み込まれている

#### 2. **処理の流れ**
1. クライアントがAPIリクエストを送信
2. kube-apiserverが受信
3. **認証・認可**をチェック
4. **Admission Controller**で検証・修正 ← ここ！
5. etcdに保存

### なぜkube-apiserverコマンドを使うのか

#### 1. **設定確認のため**
```bash
# 実際に動いているkube-apiserverの設定を確認
kubectl exec -it kubeapiserver-controlplane -n kube-system -- kube-apiserver -h
```

- kube-apiserverは**Static Pod**として動いている
- 設定ファイルを見るよりも、**実際のプロセス**のオプションを確認する方が確実

#### 2. **利用可能なAdmission Controller一覧**
```bash
kubectl exec -it kubeapiserver-controlplane -n kube-system -- kube-apiserver -h | grep 'enable-admission-plugins'
```

このコマンドで分かること：
- どのAdmission Controllerが**利用可能**か
- どのAdmission Controllerが**デフォルトで有効**か
- どのAdmission Controllerを**追加で有効化**できるか

## 📋 デフォルトで有効なAdmission Controller

```bash
# デフォルトで有効なもの
NamespaceLifecycle
LimitRanger
ServiceAccount
TaintNodesByCondition
Priority
DefaultTolerationSeconds
DefaultStorageClass
StorageObjectInUseProtection
PersistentVolumeClaimResize
MutatingAdmissionWebhook
ValidatingAdmissionWebhook
ResourceQuota
```

## 🔧 他の確認方法との比較

### 1. **設定ファイルを直接見る方法**
```bash
# Static Podの設定ファイル
cat /etc/kubernetes/manifests/kube-apiserver.yaml
```
> **問題**: 設定ファイルと実際の動作が異なる場合がある

### 2. **Podの設定を確認**
```bash
kubectl get pod kubeapiserver-controlplane -n kube-system -o yaml
```
> **問題**: 長すぎて見づらい

### 3. **kube-apiserverコマンドを使う方法** ← **推奨**
```bash
kubectl exec -it kubeapiserver-controlplane -n kube-system -- kube-apiserver -h | grep 'enable-admission-plugins'
```
> **利点**: 実際に動いているプロセスの設定を確認できる

## 💡 実用的な使い方

### トラブルシューティング
```bash
# 特定のAdmission Controllerが有効かどうか確認
kubectl exec -it kubeapiserver-controlplane -n kube-system -- kube-apiserver -h | grep -A 5 -B 5 'PodSecurityPolicy'
```

### セキュリティポリシーの確認
```bash
# セキュリティ関連のAdmission Controllerを確認
kubectl exec -it kubeapiserver-controlplane -n kube-system -- kube-apiserver -h | grep -i security
```

## 🎯 まとめ

- **kube-apiserver内にAdmission Controllerがある**のは、アーキテクチャ上の必然
- **kube-apiserverコマンドを使う**のは、実際の設定を確認するため
- 設定ファイルを見るよりも、**実際に動いているプロセス**のオプションを確認する方が確実

つまり、kube-apiserverコマンドを使うのは「設定確認のため」であり、Admission Controllerがkube-apiserver内にあるのは「処理の流れ上、そこにしか置けない」からです！

## 📊 Logging

### kubectl top
```bash
# Podのリソース使用量を確認
kubectl top pods

# ノードのリソース使用量を確認
kubectl top nodes

# 特定の名前空間のPodを確認
kubectl top pods -n kube-system
```

> ⚠️ **前提条件**: **メトリクスサーバ**が動いている必要がある

#### メトリクスサーバの確認
```bash
# メトリクスサーバが動いているか確認
kubectl get pods -n kube-system | grep metrics-server

# メトリクスサーバの詳細確認
kubectl describe pod metrics-server-xxx -n kube-system
```

### kubectl logs
```bash
# Podのログを確認
kubectl logs <pod-name>

# 特定のコンテナのログを確認
kubectl logs <pod-name> -c <container-name>

# リアルタイムでログを追跡
kubectl logs -f <pod-name>

# 過去のログも含めて確認
kubectl logs --previous <pod-name>

# 特定の行数だけ表示
kubectl logs --tail=100 <pod-name>

# 特定の時間以降のログを表示
kubectl logs --since=1h <pod-name>
```

### ログの実用的な使い方

#### トラブルシューティング
```bash
# エラーログを検索
kubectl logs <pod-name> | grep -i error

# 特定のパターンを検索
kubectl logs <pod-name> | grep "connection refused"

# 複数のPodのログを同時に確認
kubectl logs -l app=nginx
```

#### ログの保存
```bash
# ログをファイルに保存
kubectl logs <pod-name> > pod-logs.txt

# 複数のPodのログを保存
kubectl logs -l app=nginx > nginx-logs.txt
```

### ログレベルの設定
```yaml
# Podの環境変数でログレベルを設定
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: my-container
    image: nginx
    env:
    - name: LOG_LEVEL
      value: "debug"
```

## deploy update

ailias設定
alias k="kubectl"

pod内コマンド
kubectl run webapp-green --image=kodekloud/webapp-color --color=green

multi container
apiVersion: v1
kind: Pod
metadata:
  name: pods
  labels:
    run: pods
spec:
  containers:
    - name: lemon
      image: busybox
      command: ['sleep', '1000']
      resources: {}
    - name: gold
      image: redis
      resources: {}
  dnsPolicy: ClusterFirst
  restartPolicy: Always
status: {}

initコンテナ
https://kubernetes.io/docs/concepts/workloads/pods/init-containers/

サイドカーコンテナ
https://kubernetes.io/docs/concepts/workloads/pods/sidecar-containers/

k get hpa

## maintenance

そのノード上にあるPodを**他のノードに退避（削除→再スケジュール）**させる
つまり、「メンテナンスしたいから、一旦このノードを空っぽにしたい」って時に使う
k drain node01

k uncordon node01
つまり、cordonの逆コマンド。
これを実行すると、再びスケジューラがPodをそのノードに割り当てるようになる。

replicasetでない、純粋なpodがあると、drainできない

k cordon node01
ノードを“これ以上スケジュール禁止”にする。

ETCDCTL_API=3 etcdctl snapshot save /opt/snapshot-pre-boot.db \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/kubernetes/pki/etcd/ca.crt \
  --cert=/etc/kubernetes/pki/etcd/server.crt \
  --key=/etc/kubernetes/pki/etcd/server.key


## security

証明書ファイルのパス
etc/kubernetes/manifests/から見る

ca.crt：証明書を発行する側（信頼の親）
server.crt：証明書を使う側（本人）

このapiserver.crtは、俺が信頼しているca.crtによって署名されてるから安全だな👌
ca.crtを信用している限り、それが署名した他の証明書も信用できるという連鎖的な信頼構造

/etc/kubernetes/pki/ca.crt → クラスタ全体で使う信頼の“根”
/etc/kubernetes/pki/apiserver.crt → そのCAが署名したAPIサーバー証明書

CN（Common Name）
証明書の「この証明書は誰のものか」を示す識別名
Kubernetesでは、apiserver.crt は kubeadm によって自動生成され、
通常 CN=kube-apiserver として作られます。

証明書の「発行者（Issuer）」と「所有者（Subject）がある
Issuer: CN = kubernetes 
Validity Not Before: Oct 7 03:51:30 2025 GMT Not After : Oct 7 03:56:30 2026 GMT 
Subject: CN = kube-apiserver

SAN（Subject Alternative Name）
SSL証明書には「この証明書はどんな名前（ドメイン）でアクセスされたときに有効なのか
SAN は “Subject Alternative Name”（別名）という意味で、
「この証明書はこの名前たちにも有効だよ」
というリストを持てる仕組みです。
CN = kube-apiserver
SAN = kubernetes, kubernetes.default, kubernetes.default.svc, ...

Kubernetes でのSANの重要性

KubernetesのAPIサーバは、いろんな経路・名前でアクセスされます。

Pod 内部から → kubernetes.default.svc

管理者から → controlplane（ノード名）

kubeletから → 10.96.0.1（ClusterIP）

そのどれもで「証明書が一致」しないと通信が拒否されてしまう。

だからAPIサーバの証明書には、複数のDNS名とIPアドレスがSANとして登録されている

/etc/kubernetes/pki/
Public Key Infrastructure の略
/etc/kubernetes/pki/ は、Kubernetes クラスターの心臓部の証明書置き場

覚えておくと強いポイント

kubeadm で作られる証明書は基本1年有効
kubeadm certs check-expiration で期限確認できる
ca.crt だけ10年有効（他の証明書を署名し続けるため）

全部削除すると、クラスタが完全に動かなくなる⚠️
APIサーバーが etcd に繋げなくなる
kubelet が API サーバーに認証できなくなる

復旧する時は kubeadm certs renew コマンドが便利
kubeadm certs renew all

kubectl使えないときは、dockerコマンドか、crictlで、ログ見る

## kubeconfig
クラスターへの「接続情報」と「認証情報」をひとまとめにしたもの
~/.kube/config にだいたい保存されている
kubectl get pods
って打つと、kubectl は kubeconfig を見て「どの API サーバーに」「どの権限のユーザーで」アクセスするかを決めてる。
### 現在のコンテキストを確認
kubectl config current-context

### 利用可能なコンテキストを一覧表示
kubectl config get-contexts

### コンテキストを切り替える
kubectl config use-context my-context

🔹 cluster → 行き先（どの Kubernetes 環境か）
🔹 user → 身分証明（誰として入るか）
🔹 namespace → 対象の作業エリア（どこの部署を見るか）
🔹 context → その３つのセット
🔹 current-context → 今使ってるセット

echo 'export KUBECONFIG=$HOME/my-kube-config:$HOME/.kube/config' >> ~/.bashrc
source ~/.bashrc

## RBAC
RBACは4つの主要なオブジェクトで構成されます：

種類	役割	スコープ
Role	名前空間単位の権限定義	Namespace
ClusterRole	クラスタ全体の権限定義	Cluster全体
RoleBinding	RoleをユーザやServiceAccountに紐づける	Namespace
ClusterRoleBinding	ClusterRoleをユーザやServiceAccountに紐づける	Cluster全体

k create role

kubectl auth

kubectl auth reconcile

## コンテナ、pod単位のセキュリティ

securityContext:
      runAsUser: 0  # Rootユーザーで実行
      capabilities:
        add: ["SYS_TIME"] 

コンテナごとに権限を細かく制御する仕組みとして securityContext

Linuxの権限（Capabilities）をコンテナに追加する方法


## volumes

基礎となるマニフェスト書き方（コマンドではなく、volumeは全てマニフェストに直）
```
- mountPath: /log
      name: log
~
 volumes:
  - name: log
    hostPath:
      path: /var/log/webapp
```

ブックマーク
https://kubernetes.io/docs/concepts/storage/persistent-volumes/#persistent-volumes

永続ボリューム定義
```
apiVersion: v1
kind: PersistentVolume
metadata:
  name: pv-log
spec:
  capacity:
    storage: 100Mi
  volumeMode: Filesystem
  accessModes:
    - ReadWriteMany
  persistentVolumeReclaimPolicy: Retain
  storageClassName: slow
  mountOptions:
    - hard
    - nfsvers=4.1
  hostPath:
    path: /pv/log
```

永続ボリュームクレーム
```
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: claim-log-1
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Mi
  volumeMode: Filesystem
```
pvとpvcの違い
基本的に PersistentVolume（PV）と PersistentVolumeClaim（PVC）はセット
PVを手で作っておき、PODがPVCで要求する。両方セットで定義。

PVで棚を作る
PVCで、k8sが、条件に合った棚を見つける
PVとPVCは、条件一致でないといけない

strageclass
https://kubernetes.io/docs/concepts/storage/storage-classes/#local

Pod は自分でストレージを直接作りません。
→ 代わりに「PersistentVolumeClaim（PVC）」を使って「ストレージちょうだい！」とお願いする。

PVC は「このStorageClassでお願い」と言う。
→ StorageClass が「OK、じゃあEBSを自動で作るね！」と動く。

PersistentVolume（PV） が自動的に作られてPodに渡される。
→ これが実際のディスク。

```
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: kubernetes.io/aws-ebs
parameters:
  type: gp3
reclaimPolicy: Delete
```
provisioner:
どの仕組みでストレージを作るか

parameters:
 type: gp3
EBSを作るときの細かい設定。

reclaimPolicy: Delete
「ストレージを削除するときどうするか」
delete,retain

pvcのpendingとは
まだ使えるストレージ（PV）が見つかってない状態

## Networking

ネットワーク基礎のlinuxコマンド

SSHで各ノードに入ったときに、「そのノードがどんなIP設定になっているか」を調べたい時
sshで各nodeに入る
```
ip address
ip address show eth0
```

CNI（Container Network Interface）
コンテナにネットワーク（IPアドレス・通信経路）を与えるための仕組み・ルール

Podが作られるとき、KubernetesはCNIにこう頼みます：
「このPodにネットワークインターフェース（NIC）を作って、IPアドレスを割り当てて、他のPodとも通信できるようにして！

コンテナ（Pod）内に仮想NICを作る（例: eth0）
そのNICにIPを割り当てる（例: 10.244.0.5）
ノードのネットワーク（例: caliXYZ, flannel.1）と接続
Pod間・Service間通信ができるようルーティングを設定

Podが作られるとき、KubernetesはCNIにこう頼みます：
「このPodにネットワークインターフェース（NIC）を作って、IPアドレスを割り当てて、他のPodとも通信できるようにして！」

ip route
ルーティングテーブル（経路表）」 を見るコマンド
```
default via 10.0.1.1 dev eth0
10.0.1.0/24 dev eth0 proto kernel scope link src 10.0.1.23
10.244.0.0/16 via 10.0.1.100 dev eth0
```
一行目デフォゲ

nodeで今どんな通信が発生しているか」「どんなポートが開いているか」 を調べるためのコマンド
netstat

netstat -npl | 
オプション	意味
-n	ホスト名を名前解決せずに、数値(IPアドレス) で表示（速い＆見やすい）
-p	どの プロセス（PID/プログラム名） がそのポートを使っているかを表示
-l	「LISTEN」状態、つまり 待ち受け中のポートだけ を表示

netstat -npa | grep -i etcd | grep -i 2379

ssでもいい

CNI
ノードでCNI関連の設定を確認するには：
ls /etc/cni/net.d/
→ CNIの設定ファイル（例：10-calico.conflist など）がある

ip a
→ Podネットワーク用の仮想インターフェース（cni0など）が見える

kubectl get pods -A -o wide
→ PodがどのIPを持っているか確認できる（CNIで割り当てられている）

動いているプロセス（=実行中のプログラム）」を一覧表示するコマンド
ps -aux

kubelet の設定内容を調べて、コンテナランタイムが何かを確認する

kubelet
Node（ノード）でPodを動かすためのエージェント
kubelet は「Nodeの管理人」。
Kubernetes本部（APIサーバ）からの命令を受けて、
コンテナを実際に作業員（container runtime）に作らせる。

container runtime
containerdが主流
kubelet はコンテナを直接作りません。
→ 「container runtimeにお願いして」 コンテナを作ってもらいます。

endpoint（エンドポイント）とは？
「kubeletがcontainer runtimeと通信するための入り口」 です。

systemd は Linux の「サービス管理ツール」です。
Windowsでいう「サービスマネージャー」にあたります。
kubelet は OSレベルの「サービス」として常駐

/var/lib/kubelet/config.yaml
にkubelet の設定ファイルある

ls /opt/cni/bin/

controlplane ~ ➜  ls /etc/cni/net.d/
10-flannel.conflist
→これでCNIはFlannel

問題の挙動	frontend → backend の curl が成功してしまった
期待する挙動	deny-backend ポリシーで通信拒否される
実際の原因	CNI プラグイン（Flannel）が NetworkPolicy 非対応
解決策	Calico や Cilium のような policy-aware CNI に切り替える

daemonset削除
cofigmap削除
rm -rf /etc/cni

kube-proxy が必要なの？
Kubernetes では、Pod は常に消えたり増えたりするため、
Service という仮想的なIP（ClusterIP）でPodのグループを代表させています。

kube-proxy の役割
各ノード上で動くデーモン（DaemonSetとして常駐）

この中の「NodePort → ClusterIP → PodIP」への転送を
iptables または ipvs ルールで設定してるのが kube-proxy です。

外部クライアント
     │
     ▼
(Node外)
 ┌──────────────┐
 │ Node PublicIP:NodePort  ← NodePort Service
 └──────────────┘
     │
     ▼
[ kube-proxy ]
  │ （iptables/ipvs）
  ▼
ClusterIP (仮想IP)
  │
  ▼
PodIP（実際のPod、可変）

ingress



