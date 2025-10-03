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

### 確認コマンド
```bash
# 有効化されているAdmission Controllerを確認
kube-apiserver -h | grep enable-admission-plugins

# 有効化されていないAdmission Controllerを調べる
kubectl get pods -n kube-system
kubectl exec -it kubeapiserver-controlplane -n kube-system -- kube-apiserver -h | grep 'enable-admission-plugins'
```

> 💡 **注意**: Admission Controllerは認証は何もしません





