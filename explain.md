# シス管勉強会#4
## Kubernetes上にコンテナをデプロイする

```
# nginx-pod.yaml
$ vim nginx-pod.yaml

###
apiVersion: v1
kind: Pod
metadata:
  name: my-first-pod ①
  labels:
    component: nginx ②
spec:
  containers:
  - name: nginx ③
    image: nginx:latest ④
###
```

1. Podの名前。他のPodと被ってはいけない
1. ラベル。
1. コンテナの名前。一つのPodに複数のコンテナを含める場合はこの名前で区別する。
1. 実行するdocker image。`docker run` に渡すイメージ名と同じものを指定する


```
# applyする
$ kubectl apply -f nginx-pod.yaml

# Podの一覧を表示
$ kubectl get pod 
NAME           READY   STATUS    RESTARTS   AGE
my-first-pod   1/1     Running   0          67s

# Podの詳細情報の表示
$ kubectl describe pod my-first-pod

# コンテナにアクセスし，curlを試す
$ kubectl exec -it my-first-pod bash
root@my-first-pod:/# apt update && apt install -y curl
root@my-first-pod:/# curl -i localhost:80
```

## 他のPodにアクセスする
PodのIPアドレスを用いてKubernetesクラスの中からPodのIPアドレスに対してアクセスする

```
# bastion.yaml
$ vim bastion.yaml

###
apiVersion: v1
kind: Pod
metadata:
  name: bastion
spec:
  containers:
  - name: bastion
    image: debian:stretch
    command: ["sleep", "infinity"]
###

$ kubectl apply -f bastion.yaml

# nginxのIPアドレスの確認
$ kubectl get pod -o wide

# コンテナにアクセス
$ kubectl exec -it bastion bash
root@bastion:/# apt update && apt install -y curl
root@bastion:/# curl -i http://10.1.0.48/
```

## Serviceを使って他のPodにアクセスする
Serviceを使うと特定のラベルをもつPodのどれかにアクセスすることができる。

```
# nginx-service.yaml
$ vim nginx-service.yaml

###
apiVersion: v1
kind: Service
metadata:
  name: my-first-service
spec:
  selector: ①
    component: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
###

$ kubectl apply -f nginx-service.yaml
$ kubectl get service

$ kubectl exec -it bastion bash
root@bastion:/# curl -i http://my-first-service/
```

1. セレクタに指定したラベルにマッチするPodにServiceへのアクセスが割り振られます

今回は`component: nginx`というラベルを持つPodは一つしかなかったが，Podが10個あった時にmy-first-serviceに対してTCPコネクションを張るたびに10個のPodのうちにいずれかに繋がります。リバースプロキシみたいなもの。

Serviceのラベルセレクタによって，リバプロされるPodの集合が決まります。ラベルセレクタには複数のラベルを指定するとそれら全てのラベルをもつPodがセレクトされる。
ラベルは，キーと値で好きな文字列を指定でき，ひとつのPodに複数個のラベルを不要すrことも可能です。

## 同じPodをいくつもたてる
実際の運用環境では冗長性やスケーラビリティのために1つのサービスを複数台のPodで構成することが一般的です。
ReplicaSetというオブジェクトを利用してnginxを指定した台数だけ立ち上げる。

```
# nginx-replicaset.yaml
$ vim nginx-replicaset.yaml

###
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx-replicaset
  labels:
    component: nginx
spec:
  replicas: 3 ①
  selector: ②
    matchLabels:
      component: nginx
  template: ③
    metadata:
      labels:
        component: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
###

# 現在のPodの状態を確認
$ kubectl get pod

# ReplicaSetを作成
$ kubectl apply -f nginx-replicaset.yaml

# 確認
$ kubectl get replicaset
$ kubectl get pod
```

1. レプリカ数。この数だけでPodを立てる
1. ここに指定した条件に一致するPodをこのReplicaSetが管理する
1. このReplicaSetが作成するPodの定義

ReplicaSetは`selector`にマッチするPodの数がreplicasに指定した数になるようにPodを自動的にデプロイしたり削除したりするオブジェクトです。

ReplicaSetがapplyされた時，`component: nginx`というラベルを持っていたPodは`my-first-pod`1つだけで，`replicas`が3に指定されていたのでnginxが2つ追加でデプロイされます。


デプロイされるPodは`template`で指定された定義で作られる。

`my-first-pod`を削除してみよう。

```
$ kubectl delete pod my-first-pod
$ kubectl get pod
```

ReplicaSetは定義されたレプリカ数と実際のレプリカ数が一致するようにPodをデプロイしたり削除してくれたりします。


## ローリングアップデートする
無停止でアプリケーションを更新する手段としてよく利用されるのがローリングアップデートです。

ローリングアップデートを行うにはDploymentというオブジェクトを利用します。
DeploymentはReplicaSetといているが，ReplicaSetと違ってアプデートがサポートされています。

```
# 紛らわしくなるのでReplicaSetを削除する
$ kubectl delete replicaset nginx-replicaset
$ kubectl get pod

# nginx-deployment.yaml
$ vim nginx-deployment.yaml

###
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    component: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      component: nginx
  template:
    metadata:
      labels:
        component: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.15
###

$ kubectl apply -f nginx-deployment.yaml

# 確認する(バージョンを確認するためにIMAGEを表示します)
$ kubectl get pod -o 'custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[*].image,PHASE:.status.phase'

# nginx: 1.15をnginx: 1.16に書き換える
$ vim nginx-deployment.yaml

$ kubectl apply -f nginx-deployment.yaml

# 確認
$ kubectl get pod -o 'custom-columns=NAME:.metadata.name,IMAGE:.spec.containers[*].image,PHASE:.status.phase'
```

大抵の場合は，Deploymentを使ってPodを作成します。Podだけの場合ノードが死ぬとPodも道連れになってしまうためです。

## サービスをクラスタの外部に公開する
Kubernetes上にデプロイされたサービスをKubernetesクラスタの外部に公開する方法はいくつかありますが，ここでは最も簡単なNodePortを使います。

```
# nginx-service.yamlを書き換える
$ vim nginx-service.yaml

###
apiVersion: v1
kind: Service
metadata:
  name: my-first-service
spec:
  selector:
    component: nginx
  ports:
  - protocol: TCP
    port: 80
    targetPort: 80
    nodePort: 30000
  type: NodePort
###

$ kubectl apply -f nginx-service.yaml

$ kubectl get service my-first-service

# minikubeの場合
$ minikube ip

# 確認
$ open http://localhost:30000/
```

NodePort型のServiceを使うと，ノードの指定したポート経由でサービスにアクセスできるようになります。

全てのノードで，指定したポートが指定したサービスにルーティングされるようになる。

## Podをデバッグする
`kubectl exec`や`kubectl descrive`がPodのデバッグに役立ちます。
これに加えて`kubectl logs`と`kubectl port-forward`を利用します。

### コンテナのログをみる
コンテナが標準出力・標準エラー出力に書き出したログを`kubectl logs`コマンドで確認します。`-f`オプションをつけると`tail -f`のようにログを流してくれます。

```
$ kubectl get pod

$ kubectl logs nginx-deployment-6fb8fbbfc8-rcvfw 

$ kubectl logs -f nginx-deployment-6fb8fbbfc8-rcvfw 
```

しかし，`kubectl logs`は一つのコンテナのログしか表示してくれません。複数のコンテナのログを一度にみたい場合は[stern](https://github.com/wercker/stern)が便利です。

```
# install stern
$ brew install stern

$ stern "nginx-deployment-\w"
```

詳しい使い方はQiitaとかgithub見てみてください！

### ポートフォワードする
`kubectl port-forward`コマンドを使うと，Podの特定のポートをローカルホストにポートフォワードすることができます。

```
$ kubectl port-forward deployment/nginx-deployment 8080:80
$ curl http://localhost:8080
```

