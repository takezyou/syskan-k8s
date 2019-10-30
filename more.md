## ヘルスチェック
Podは障害で応答不能な状態になったり，起動中でリクエストを処理できない状態であったりすることがあります。Kubernetesは常にPodの状態を監視しており，必要に応じてPodを再起動したりServiceのリバプロ先から除外したりします。Kubernetesのヘルスチェックは以下の2つがあり，用途に応じて使いわけることができる。

- Liveness probe  
  Podが生きているか死んでいるかをチェックします。一定回数Liveness probeが失敗した場合，Podは再起動されます。
- Rwadiness probe  
  Podがリクエストに応答できるかどうかをチェックします。Readiness probeが成功するまでの間，そのPodは起動が完了していないとみなされ，Serviceのリパプロ先に追加されません。Readiness probeを使うことでPodが起動完了する前にリクエストが飛んでくる現象を防ぐことができます。

例:

```
apiVersion: v1
kind: Pod
metadata:
  name: liveness-http
spec:
  containers:
  - name: my-server
    image: my-server:1.2.3.4
    livenessProbe:
      httpGet:
        path: /v1/health
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 3
    readinessProbe:
      httpGet:
        path: /v1/ready
        port: 8080
      initialDelaySeconds: 3
      periodSeconds: 3
```

