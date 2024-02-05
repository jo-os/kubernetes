Установка kubespray - https://habr.com/ru/companies/domclick/articles/682364/

1
```
kubectl config view
kubectl cluster-info
kubectl get no
sudo kubectl get no -o wide
```
2
```yml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
  - name: nginx-conteiner
    image: nginx
```
```
kubectl get namespaces
kubectl get po -n kube-system
sudo kubectl get po -A
kubectl create -f pod.yml
```
Статусв
- Pending - объект создан
- Running - все контейнеры Poda запущены
- Succeeded - все контейнеры корректно завершили работу
- Failed - все контейнеры остановлены (как минимум один завершил работу не корректно)
- Unknow - kube-apiserver не имеет информации о поде
```yml
apiVersion: v1
kind: Pod
metadata:
  name: app
spec:
  containers:

    - name: server
      image: jooos/test-server:1.0

    - name: client
      image: curlimages/curl:7.76.0
      command: ["/bin/sh"]
      args: [ "-c", "while true; do curl -s http://localhost:8080/date;sleep 2;done" ]
```
```
kubectl logs my-pod
kubectl logs -f app -c server
kubectl logs -f app -c client
```
3
```yml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
  labels:
    app: nginx
    cluster: prod
spec:
  containers:
  - name: nginx-conteiner
    image: nginx
```
```
kubectl get pods --show-labels
kubectl get po -l app=nginx
kubectl get po -l cluster=prod
kubectl get po -l app!=nginx,cluster=prod
kubectl delete pods -l cluster=prod
```
```yml
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: nginx
  labels:
    app: nginx
    cluster: prod
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx
```
```
kubectl create -f rs.yml
kubectl get replicaset
kubectl edit pod {pod-name}
sudo kubectl get pods -o=custom-columns=NAME:.metadata.name,DATA:metadata.ownerReferences - кем создан под
kubectl scale replicaset {replicaset name} --replicas=5
kubectl delete replicaset {replicaset name}
```
operator (matchExpressions)
- In - label должен совпадать с одним из указанных значений в values
- NotIn - label должен не совпадать с одним из указанных значений в values
- Exists - под должен содержать метку с указанным ключем
- DoesNotExist - под не должен содержать метку с указанным ключем
```
spec:
  replicas: 3
  selector:
    matchExpressions:
    - key: app
      operator: In
      values:
      - nginx
```
```
kubectl delete replicaset nginx --cascade=false - удаляет replicaset но оставляет поды
```
4
```yml
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 25%
      maxUnavailable: 25%
  selector:
    matchLabels:
      app: app
  template:
    metadata:
      labels:
        app: app
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
```
```
kubectl get deployment
kubectl rollout status deployment app-deployment
kubectl set image deployment app-deployment app=app:v2.0 --record
kubectl rollout undo deployment app-deployment
kubectl rollout history deployment app-deployment
kubectl rollout undo deployment app-deployment --to-revision=1
```
```yml
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: fluentd-elasticsearch
spec:
  selector:
    matchLabels:
      name: fluentd-elasticsearch
  template:
    metadata:
      labels:
        name: fluentd-elasticsearch
    spec:
      nodeSelector: # только поды с этой меткой
        env: prod
      containers:
      - name: fluentd-elasticsearch
        image: quay.io/fluentd_elasticsearch/fluentd:v2.5.2
```
```
kubectl get daemonset

kubectl get nodes --show-labels
kubectl label node node9 env=dev # задать метку поду
kubectl label node node9 env=prod --overwrite # перезапишем метку
```
