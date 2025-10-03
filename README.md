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

