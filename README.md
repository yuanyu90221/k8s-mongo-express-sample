# k8s-mongo-express-sample

本篇是採用一個 MongoDB 結合 MongoExpress  應用當作範例

來實際操作佈署 app 到 k8s 的案例

MongoDB 是一個以文件為基底儲存類型的資料庫

MongoExpress 是一個網頁版本用來管理 MongoDB 的後台管理系統

## 預計佈署結構

MongoDB 在佈署上, 不希望被外部存取

因此, 會使用 Internal Service 類型

並且需要被其他再同一個 Cluster 內的 Service 透過 DB Url 的方式存取

所以會使用 ConfigMap 來設定 DB Url

並且會用 Secrets 來放置驗證用的資料庫使用者帳密

MongoExpress 在佈署上, 會透過網頁讓權限的使用者操作

屬於 External Service 類型

而在 Deployment 會去存取 建制 MongoDB 所設定的 ConfigMap 與 Secrets

預計佈署結構如下：

![](https://i.imgur.com/rAfD1Ks.png)


預計 Request flow 如下圖：

![](https://i.imgur.com/loszMDL.png)

## mongodb 部份

基礎 mongodb-deployment.yaml 撰寫如下:

```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
```

注意的是 這邊是用 template 的方式來設定 Pod 內部 Container 的運行藍圖

```yaml=
template:
  metadata:
  labels:
    app: mongodb
  spec:
    containers:
    - name: mongodb
      image: mongo
```
然而, 這樣寫沒有寫入用來驗證的 root user 跟 root password

從 dockerhub [mongo image repository](https://hub.docker.com/_/mongo) 內容

說明在 environment 代入 MONGO_INITDB_ROOT_USERNAME, MONGO_INITDB_ROOT_PASSWORD 就會設定好上面兩個選項 


在這個範例, 筆者先建立 Secret 然後在 Deployment 使用 Reference 方式去引用

建立 mongo-secret.yaml 如下：


```yaml=
apiVersion: v1
kind: Secret
metadata:
  name: mongodb-secret
type: Opaque
data:
  mongo-root-username: dXNlcm5hbWU=
  mongo-root-password: cGFzc3dvcmQ=
```

注意的是, 這邊的 Opaque 是用來儲存 key value 對應的資料形式

mongo-root-username, mongo-root-password 是分別用 username, password 做 base64 編碼做出來的字串 

可以用以下方式產生
```bash=
echo -n 'username' | base64
```

重要的是, 由於需要在 mongodb-deployment.yaml 中使用 mongo-secret.yaml 的值

mongo-secret.yaml 必須先發佈才能發佈 mongodb-deployment.yaml

發佈 mongo-secret.yaml
```bash=
kubectl apply -f mongo-secret.yaml
```
![](https://i.imgur.com/i3Xqzi0.png)

而新的 mongodb-deployment.yaml 更新如下：

```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongodb-deployment
  labels:
    app: mongodb
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
      - name: mongodb
        image: mongo
        ports:
        - containerPort: 27017
        env:
        - name: MONGO_INITDB_ROOT_USERNAME
          valueFrom:
             secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
       - name: MONGO_INITDB_ROOT_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
```
發佈 mongodb-deployment.yaml

```shell=
kubectl apply -f mongodb-deployment.yaml
```
![](https://i.imgur.com/7ZUutDR.png)

設定 Service 給 mongodb pod

在這個範例中, 筆者把 Service 跟 Deployment 利用以下的語法寫在同一個檔案

```yaml=
--- 是yaml中用來分隔多個 doucment 的語法
```

所以更新後的結構會如下：
```yaml=
... Deployment 設定 
---
... Service 設定
```
只看 mongodb-deployment.yaml Service的部份如下

```yaml=
apiVersion: v1
kind: Service
metadata:
  name: mongodb-service
spec:
  selector:
    app: mongodb
  ports:
    - protocol: TCP
      port: 27017
      targetPort: 27017
```

關鍵的部份：
1 selector 要跟 Deployment 設定 Pod 的部份一致, 才有辦法關聯 Service 到對應的 Pod

2 targetPort: 是指 Container 的 Port 也就是前面 template 設定的 containerPort 

3 port： 這邊是開發給其他 Service 溝通的 Port

4 預設的 Type 是 ClusterIP 也就是 Internal Service 外部無法直接透過 IP 存取

發佈 mongodb-deployment.yaml 
```shell=
kubectl apply -f mongodb-deployment.yaml
```
![](https://i.imgur.com/qxRmJK5.png)

### 驗證 Service 有應用到 Pod 上面

![](https://i.imgur.com/20FKOLU.png)

驗證 kubectl describe 部份的 EndPoint IP 與 kubectl get pod -o wide 部份的 IP 是相同

## mongo-express 部份

基礎 mongo-express-deployment.yaml 如下

```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
```

從 dockerhub [mongo-express 官方 image](https://hub.docker.com/_/mongo-express) 內容

代表需要設定3個環境變數給 mongo-express 運行

ME_CONFIG_MONGODB_ADMINUSERNAME, ME_CONFIG_MONGODB_ADMINPASSWORD, ME_CONFIG_MONGODB_SERVER

前面兩個用來做 credential, 第3個變數用來指定連結的資料庫位址

而在之前的 mongo-secret.yaml 中, 設定的mongo-root-username, mongo-root-password 剛好就是這兩個

所以只需要在設定好一個 ConfigMap 來儲存 ME_CONFIG_MONGODB_SERVER 即可

設定 mongodb-configmap.yaml 如下：

```yaml=
apiVersion: v1
kind: ConfigMap
metadata:
  name: mongodb-configmap
data:
  database_url: mongodb-service
```
關鍵的部份, 這裡的 database_url 是使用之前發佈的 mongodb-service 這個 Service 名稱來當 url
發佈 mongodb-configmap.yaml 
```shell=
kubectl apply -f mongodb-configmap.yaml
```

更新 mongo-express-deployment.yaml 如下：

```yaml=
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mongo-express
  labels:
    app: mongo-express
spec:
  replicas: 1
  selector:
    matchLabels:
      app: mongo-express
  template:
    metadata:
      labels:
        app: mongo-express
    spec:
      containers:
      - name: mongo-express
        image: mongo-express
        ports:
        - containerPort: 8081
        env:
        - name: ME_CONFIG_MONGODB_ADMINUSERNAME
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-username
        - name: ME_CONFIG_MONGODB_ADMINPASSWORD
          valueFrom:
            secretKeyRef:
              name: mongodb-secret
              key: mongo-root-password
        - name: ME_CONFIG_MONGODB_SERVER
          valueFrom:
            configMapKeyRef:
              name: mongodb-configmap
              key: database_url
```

發佈 mongo-express-deployment.yaml
```shell=
kubectl apply -f mongo-express-deployment.yaml
```
![](https://i.imgur.com/uYBTsV3.png)

### 設定 External Service 
同樣的這樣也會設定 Service 在 Deployment 同一個檔案使用分隔線的方式來撰寫


所以更新後的結構會如下：
```yaml=
... Deployment 設定 
---
... Service 設定
```

只看 Service 的部份如下：
```yaml=
apiVersion: v1
kind: Service
metadata:
  name: mongo-express-service
spec:
  selector:
    app: mongo-express
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 8081
      targetPort: 8081
      nodePort: 30000
```

關鍵的部份：

1 因為是 External Service 要設定 type 是 LoadBalancer

2 因為是 External Service 要額外設定 nodePort 範圍是 30000~32767

```shell=
kubectl apply -f mongo-express-deployment.yaml
```

![](https://i.imgur.com/ohXfyJK.png)

注意的是 在 type 是 LoadBalancer 的部份 EXTERNAL_IP 顯示 pending 代表還沒拿到實體外部 IP

而 minikube 這邊比較特別, 需要使用以下指令才能拿到 EXTERNAL_IP

```shell=
minikube service $service_name
```

以本篇範例來說就是以下指令：

```shell=
minikube service mongo-express-service
```

![](https://i.imgur.com/IGoZ9Tn.png)

然後就可以用瀏覽器打開對應的 URL 做操作, 如下:

![](https://i.imgur.com/x20oVcf.png)