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

## taintとToleration

Taint = ノード側のルール
「このノードは普通のPodはダメ」

Toleration = Pod側の許可証
「私はこのルールを無視してもOK」

kubectl taint nodes node1 dedicated=experimental:NoSchedule

tolerations:
- key: "dedicated"
  operator: "Equal"
  value: "experimental"
  effect: "NoSchedule"

kubectl taint node node01 spray=mortein:NoSchedule

削除するときは
kubectl taint node node01 spray=mortein:NoSchedule-

トレレーションは、yamlでしか設定できない

operetion
Equal → value まで一致する必要あり（厳格）

Exists → key が一致すれば OK（ゆるめ）

Exists のときに value 書くとエラーになる

## Node Selector＆Node Affinity

Pod側がここに配置したいと希望する仕組み
taintだけでは禁止のみで、他のnodeに配置されてしまう可能性があるため

```
# Node Affinity
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

```
# Node Affinity
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

Operatorが、演算子
In → 指定した値のいずれかに一致する
NotIn → 指定した値には含まれない
Exists → key があればマッチ
DoesNotExist → key がなければマッチ

Node Affinity と NodeSelector の違い

NodeSelector
シンプルに key=value だけで指定できる（古い方式）。

nodeSelector:
  disktype: ssd

→ シンプルだけど柔軟性がない。

Node Affinity
NodeSelector の上位互換。
複数条件や In/NotIn/Exists が使える。

Node Affinity
→ Pod が「ここに置きたい！」と希望する。

Taint/Toleration
→ ノードが「俺は特殊だから勝手に来るなよ」と制限する。
両方組み合わせると「特定のPodだけ特定ノードに入る」という厳密な制御ができる。

taint設定のみ抜き出す
kubectl describe node node1 | grep taints

Shift + V
>
で一気にずらせる

## resource limit

メモリ不足
 OOMKilled

 editでメモリの値を増やすのは不可 実行中のため。
 しかし、edit失敗しても、yamlが吐き出されるため、それをつかってreplace可能
 controlplane ~ ➜  kubectl edit pod elephant 
error: pods "elephant" is invalid
A copy of your changes has been stored to "/tmp/kubectl-edit-2855789753.yaml"
error: Edit cancelled, no valid changes were saved.

kubectl replace --force -f /tmp/kubectl-edit-2855789753.yaml

## Daemon set

🔹 DaemonSetとは？

「クラスタ内のすべてのノード、または特定の条件を満たすノードに、必ず1つずつ Pod を配置する仕組み」

DaemonSet は「ノードごとに必ず必要になるエージェント系のワークロード」に使われます。

```
# daemon set
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
Deployment に似ているけど、replicas は指定しない

Pod 数はノード数によって自動的に決まる

ノードが増えると自動でそのノードにも Pod が作られる

ノードが削除されれば、その Pod も消える

もし「全ノードじゃなくて特定のノードだけ」に入れたい場合は

NodeSelector / NodeAffinity を組み合わせる

Tolerations を組み合わせて taint されたノード専用にする

クラスタ内全てのnodeに、一つのpodをつける
監視エージェントなど

全ての名前空間をみるには -A
kubectl get daemonsets だけだと、デフォルト名前空間のみ

## static pod

kubeletが直接管理するpod
API Server / kube-scheduler を経由せず、そのノード上の kubelet によって起動・監視される特殊な Pod。
マニフェストファイルをノードの特定ディレクトリに置くと、その内容に従って自動的に起動する。

kubelet＝Kubernetes の各ノード上で動くエージェントプロセス。
APIサーバから渡された指示を受けて、実際にコンテナを起動させる役割

/etc/kubernetes/manifests/

削除方法が特殊
kubectl delete pod しても消えない（kubelet が再作成する）
YAML ファイルを消すと停止する
マスターコンポーネントで利用
Kubernetes 自体のコア（kube-apiserver, etcd, controller-manager, scheduler）は Static Pod として起動されることが多い
マニフェスト修正即反映
ディレクトリ内のファイルを修正すると、自動的に Pod が再作成される

トラブル対応（etcd 落ちた、APIサーバが起動しない） のときに知っておくと役立つ

 kubectl get pods -Aで、末尾にノード名のpodが、static

 cat /var/lib/kubelet/config.yaml 
 staticPodPath: /etc/just-to-mess-with-you
 config.yaml

Terminating のまま消えない時は？
 kubectl delete pod POD名 --grace-period=0 --force

コマンド
--command --sleep 1000

## Pod Priority

Pod に「優先度」を与える仕組み
クラスタ内のリソースが足りなくなったとき、どの Pod を優先するかを決める仕組み

```
# PriorityClass の定義
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-priority
value: 100000
globalDefault: false
description: "This priority class should be used for critical pods only."

```
これを作成後、podに割り当てる

value: 優先度（大きいほど優先される）
ケジューリング時

高優先度 Pod は、低優先度 Pod よりも先にスケジューリングされる

プリエンプション（追い出し）

ノードに空きがなくても、高優先度 Pod が来ると、低優先度 Pod が強制削除されて場所を譲ることがある

これを Preemption と呼ぶ

名前空間もたない

kubectl get priorityclass

kubectl get pods -o custom-columns="NAME:.metadata.name,PRIORITY:.spec.priorityClassName"

## 複数のスケジューラ

普段は、デフォルトのスケジューラが、いい感じにやってくれている。(kube-scheduler)
クラスタに、複数のスケジューラをデプロイできる
→独自のスケジューラアルゴリズムで、ノードにpodを配置したいとき

podのspecに、スケジューラ名を書けば、複数のスケジューラを同時に動かせる
schedulerName: スケジューラ

## admission controller

Kubernetes で Pod や Deployment を作るとき、APIサーバー にリクエストが届きます。
APIサーバーはこのリクエストを
受け取る
認証/認可チェック（この人操作していい？）
etcd に保存
の流れで処理します。
この途中に「チェックマン」や「修正マン」を挟めるのが Admission Controller です。
Admission Controller とは？

「リクエストを受けたときに、クラスタに保存される前にチェックしたり書き換えたりする仕組み」
Validating（検証系）

「この Pod、セキュリティポリシー違反してない？」

例：root 権限で動かそうとしていたら「ダメ！」と拒否する。

Mutating（書き換え系）

「設定が足りないから自動で補っておくね」

例：Pod にデフォルトの sidecar（ログ収集コンテナ）を勝手に追加する。

APIサーバーにリソースが保存される前のゲートキーパー
役割は「チェック（Validating）」と「自動修正（Mutating）」

kube-apiserver -h | grep enable-admission-plugins

アドミッションコントローラは認証何もしない

有効化されていないaddmission controlerを調べる
kubectl get pods -n kube-system
kubectl exec -it kubeapiserver-controlplane -n kube-system -- kube-apiserver -h | grep 'enable-admission-plugins'





