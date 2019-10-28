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

