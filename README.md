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


