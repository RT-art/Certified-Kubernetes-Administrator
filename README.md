<div align="center">

# 🎓 Certified Kubernetes Administrator (CKA)

試験合格のためのチートシートです

</div>

## pod
ノードを確認
kubectl get pods -o wide
-o アウトプット 

-o yaml → YAML形式で全情報を出す
-o json → JSON形式で出す
-o wide → 表形式のまま、ちょっと情報を広げて表示する

wideの表示内容
Podがどのノードに乗ってるか
PodのIP
コンテナのイメージ

yamlで出力
kubectl run redis --image=redis123 --dry-run -o yaml
→これは非推奨。

クライアントとサーバがある
client ローカルで雛形生成・形だけ確認
server クラスタで本番と同じ検証（ただし保存しない）

ターミナルにyamlを吐くだけ
kubectl run redis --image=redis123 --dry-run=client -o yaml

実際にファイルを作る
kubectl run redis --image=redis123 --dry-run=client -o yaml > redis.yaml
cat redis.yaml
kubectl create -f redis.yaml

既存のpodを修正したいとき
マニフェストファイルで編集
vi redis.yaml
kubectl apply -f redis.yaml

サクッとコマンドで修正
kubectl edit

## replicaset

replicasetの数を見る
kubectl get replicaset

replicasetの詳細内容を確認する
kubectl describe replicaset new-replicaset

replicasetをファイルで作成する
kubectl create -f /root/replicaset.yaml

kubectl explainは、Kubernetesリソースの仕様・フィールドの意味を教えてくれる
kubectl explain replicaset

→YAML関連は explain、コマンド忘れたら help

apiVersion: v1
KubernetesのコアAPIグループ
pod
sevice
ConfigMap
secret
Namespace
など

apiVersion: apps/v1
より「高度なオブジェクト（コントローラ系）」がここに入ってる
Deployment
StatefulSet
DaemonSet
ReplicaSet
など

ReplicaSet は「特定のラベルを持つ Pod を監視して、指定した数（replicas）だけ存在するようにする」コントローラ。
つまり ReplicaSet は
spec.selector（どんなPodを管理するか条件を定義）
spec.template（その条件に合うPodを自分で作るときのテンプレート）
の2つを使って動いてる。

セレクターは、今から作られるものだけでなく、既存のpodも管理対象に入れられるようになっている
テンプレートは、作った瞬間、セレクターの管理配下

spec = そのリソースの望ましい状態

rsで、replicaset省略可能
kubectl get rs

レプリカセットをスケール数を増減
kubectl scale rs new-replica-set --replicas=5

## deployment

Deployment

   └── ReplicaSet

         └── Pod


Deployment
→ 「アプリケーションをこういう形で動かしたい（ローリングアップデートで管理してね）」って仕様を宣言するコントローラ。

ReplicaSet
→ 「Podを常に n 個保つ」役割。Deploymentの配下で、実際にPodを維持する。

Pod
→ 実際にコンテナが動く最小単位。

kubectl get deploy

kubectl get all

ブックマークするページ（run apply dry-runなどのコマンド）
https://kubernetes.io/docs/reference/kubectl/conventions/

kubectl create deployment --image=nginx nginx --replicas=4 --dry-run=client -o yaml > nginx-deployment.yaml

yamlは基本runコマンドで生成したものを修正して使う。

## service

nodeport
ノード ポート
つまり、外部接続に使われるノード自体のポート

ノードポート→サービス→pod

サービス側から視点
targetport→podのport
port→サービスのport

yaml
ラベルはつけてもつけなくてもいい
```
# nodeport

apiVersion: v1
kind: Service
metadata:# ラベルはつけてもつけなくてもいい
  name: my-service
spec:
  type: NodePort
  selector:
    app: nginx   # 管理対象Podのラベル
  ports:
    - targetPort: 80    # Podのポート
      Port: 80    # 自分のport
      nodePort: 30080   # ノードの外部からアクセスするポート (30000-32767)
```

clusterIP
podは、IPがめちゃくちゃ変わる
グループで抽象化する仕組みが、clusterIP

ロードバランサー
外部からの入り口
type: NodePort にすると、クラスタの全Nodeで同じポート番号（例:30080）が開く

だからアクセスの仕方はこうなる👇

http://<Node1のIP>:30080
http://<Node2のIP>:30080
http://<Node3のIP>:30080


つまり「ノードの数だけ外部からの入口URL」ができちゃう

👉 これだと利用者からすると 「どのノードにアクセスすればいいの？」問題 が発生する
type: LoadBalancer を使うと、クラウド（AWS/GCP/Azureなど）が 外部ロードバランサ を自動で立ち上げてくれる

外部ユーザーは 1つの固定のエンドポイント（VIPやDNS名） にアクセスするだけでOK

そのLBがクラスタ内のNodePortに分散して、さらにPodに流してくれる

kubectl get svc

## namespace

kubectl get nc

kubectl get pods --namespace=
kubectl get pods -n=

kubectl get pods --all-namespaces
kubectl get pods -A



