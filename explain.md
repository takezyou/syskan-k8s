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

$ kubectl exec -it bastion bash
```
