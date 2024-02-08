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
<details>
  <summary>Replicaset</summary>
  
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
</details>

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
<details>
  <summary>Deployment</summary>
  
```yml
apiVersion: apps/v1
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
</details>

```
kubectl get deployment
kubectl rollout status deployment app-deployment
kubectl set image deployment app-deployment app=app:v2.0 --record
kubectl rollout undo deployment app-deployment
kubectl rollout history deployment app-deployment
kubectl rollout undo deployment app-deployment --to-revision=1
```
<details>
  <summary>DaemonSet</summary>
  
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
</details>

```
kubectl get daemonset

kubectl get nodes --show-labels
kubectl label node node9 env=dev # задать метку поду
kubectl label node node9 env=prod --overwrite # перезапишем метку
```
5
<details>
  <summary>Service</summary>
  
```yml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: MyApp
  ports:
    - protocol: TCP
      port: 80
      targetport: 9376
```
</details>
  
<details>
  <summary>Deployment</summary>
  
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
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
      - name: server
        image: nginx:1.20
        ports: 
        - containerPort: 80
```
</details>

```
kubectl create -f nginx-deployment.yml
kubectl get po -o wide
kubectl expose deployment nginx-deployment --port 80 --target-port 80
kubectl get services
kubectl describe service nginx-deployment
kubectl get endpoints
kubectl describe endpoints nginx-deployment
kubectl run tmp-pod --rm -i --tty --image nicolaka/netshoot -- /bin/bash # одиночный под для проверок
curl http://nginx-deployment
kubectl delete service nginx-deployment
```
dnsPolicy:
- Default - наследует настройкм с ноды на которой запушен
- ClusterFirst - по умолчанию - все днс запросы которые не совпадают с кластер днс суффикс перенаправляются на upstreamDNS сервер, адрес которого наследуется с ноды
- ClusterFirstWithHostNet - для подов который запушены с параметром HostNetwork
- None - можно прописать настройки днс в спецификации пода
```
cat /etc/resolv.conf
search default.svc.cluster.local svc.cluster.local cluster.local
в одном namespace default:
curl http://nginx-deployment
curl http://nginx-deployment.default
curl http://nginx-deployment.default.svc.cluster.local
```
6
<details>
  <summary>Job</summary>
  
```yml
apiVersion: batch/v1
kind: Job
metadata:
  name: pi
spec:
  completions: 5
  parallelism: 5
  template:
    spec:
      containers:
      - name: pi
        image: perl
        command: ["perl","-Mbignum=bpi","-wle","print bpi(2000)"]
      restartPolicy: Never
      activeDeadlineSeconds: 100
  backoffLimit: 4
```
</details>

```
kubectl get jobs
kubectl get pods - смотрим STATUS
```
<details>
  <summary>CronJob</summary>
  
```yml
apiVersion: batch/v1
kind: CronJob
metadata:
  name: hello
spec:
  schedule: "*/1 * * * *"
  jobTemplate:
    spec:
      template:
        spec:
          restartPolicy: OnFailure
          containers:
          - name: hello
            image: busybox
            command:
            - /bin/sh
            - -c
            - date; echi Hello from Kube cluster
```
</details>

```
kubectl get cronjob
kubectl get pods
kubectl logs {pod-name}
```
<details>
  <summary>Init</summary>

```
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
spec:
  initContainers:
  - name: init-db
    image: busybox:1.28
    command: ['sh','-c',"until nslookup mydb.default.svc.cluster.local; do echo waiting for mydb; sleep 2; done"]
  containers:
  - name: myapp-container
    image: busybox:1.28
    command: ['sh','-c','echo The app is running! && sleep 3600']
```
</details>

```
kubectl logs myapp-pod -c init-db
kubectl get event --field-selector involvedObject.name=myapp-pod --watch
kubectl create service clusterip mydb --tcp=5432:5432 - сервис для бд
```
7

**Admission Controller**
- InitialResources - устанавливает лимиты по умолчанию для ресурсов
- LimitRanger - устанавливает лимиты по умолчанию для запросов и лимитов контейнера
- ResourceQuota - считает количество объектов и общие потребляемые ресурсы и предотвращает их превышение

**Webhooks**
- MutatingAdmissionWebhook - модифицирует запрос
- ValidatingAdmissionWebhook - валидирует запрос

```

```
8
<details>
  <summary>Env in Pod</summary>
  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: envar-demo
  labels:
    purpose: demonstrate-envars
spec:
  containers:
  - name: envar-demo-container
    image: gcr.io/google-samples/node-hello:1.0
    env:
    - name: DEMO_GREETING
      value: "Hi all!"
```
</details>

```
kubectl exec envar-demo -- printenv
```
<details>
  <summary>ConfigMap</summary>
  
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-app-config
data:
  DRIVER_ADDR: "https://payment.prod.env:8080"
  JWT_ISSUER: "team.prod.env"
```
</details>

```
kubectl describe configmaps demo-app-config
```
<details>
  <summary>Pod+ConfigMap</summary>
  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: demo-app
spec:
  containers:
    - name: demo-app
      image: gcr.io/google-samples/node-hello:1.0
      envFrom:
      - configMapRef:
          name: demo-app-config
      env:
        - name: JWT_ISSUER_SECOND
          valueFrom:
            configMapKeyRef:
              name: demo-app-config
              key: JWT_ISSUER
```
</details>

```
kubectl exec demo-app -- printenv
```
<details>
  <summary>ConfigMap 2</summary>
  
```yml
apiVersion: v1
kind: ConfigMap
metadata:
  name: demo-app-config
data:
  DRIVER_ADDR: "https://payment.test.env:8080"
  JWT_ISSUER: "team.test.env"
```
</details>

```
kubectl create namespace test
kubectl create -f cm-test.yml -n test
kubectl describe configmap demo-app-config -n test
```
<details>
  <summary>Secret</summary>
  
```yml
apiVersion: v1
kind: Secret
metadata:
  name: mysecret
type: Opaque
data:
  username: YWRtaW4K
  password: dmVyeS1zZWMtcGFzc3dvcmQK
```
</details>

```
kubectl get secret
kubectl describe secret mysecret
kubectl create secret generic user-creds --from-file=./username.txt --from-file=./password.txt
kubectl describe secret user-creds
```
<details>
  <summary>Pod+Secret</summary>
  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
  - name: sec-app
    image: gcr.io/google-samples/node-hello:1.0
    envFrom:
      - secretRef:
          name: mysecret
      - secretRef:
          name: user-creds
```
</details>

```
kubectl exec -it secret-pod -- printenv
```
9
Контроль доступа
- Аутентификация - кто?
- Авторизация - что можно?
- Admission Control - можно изменить или отклонить запрос

ServiceAccount - для служб и общения внутри кластера, по умолчанию default - но у default мало прав
```
kubectl get pod coredns-69db55dd76-mqbdt -n kube-system -o json
        "serviceAccount": "coredns",
        "serviceAccountName": "coredns",
kubectl get serviceaccount coredns -n kube-system -o json
kubectl config view
```
```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx
  namespace: kube-system
spec:
  containers:
  - image: nginx
    name: nginx
  serviceAccountName: coredns
```
```
kubectl exec -it nginx -n kube-system -- bash
cd /var/run/secrets/kubernetes.io/serviceaccount/
ls -> ca.crt	namespace  token
cat token
jwt.io ->
    "serviceaccount": {
      "name": "coredns",
```
Плагины аутентификации (модули)
- Пользовательские сертификаты - если мало пользователей / нужен центр сертификации для отзыва
- Файлы с токеном - подкладывает api серверам файл с токенами
- Файл с пользователями/паролями - устарел
- Service account - для внутренних пользователей - по ключу который генерит куб
- Authenticating Proxy - аутентификация по заголовкам, которые подставляет прокси
- OpenID Connect Provider - через внешнего провайдера, который дает access токен


Файл с токеном:
token,user,uid,"group1,group2,group3"

Well you have to pass the path where is static token file located on your host machine in directoy so that you can point to that file just like this. Edit the kubeapiserver.yaml file which is located at /etc/kubernetes/manifests and add the below flag. 
```
--token-auth-file=/path/where/yourfile/located/which/contain/tokens
```
10

Модули авторизации
- Node - раздает разрешение kubelet смотреть и писать в api про свою ноду
- ABAC - устарел
- RBAC - role based access control - актуальный
  - subject - пользователи и процессы которыс нужен доступ к api
  - resources - объекты доступные в кластере
  - verbs - совокупность операций которые могут быть выполнены над ресурсами
- Webhook - http вызов к стороннему сервису - для определения прользовательских привелегий

RBAC = subject + resources + verbs

**Role, ClusterRole** - соединяют ресурсы и глаголы, можно использовать для разных пользователей, Role - действуют в пределах одного namespace, ClusterRole - для всего кластера

**RoleBinding и ClusterRoleBinding** - соединяют роли с субъектами (пользователями), RoleBinding в пределах namespaces, ClusterRoleBinding на кластер

Role
```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-reader
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get", "watch", "list"]
```
RoleBinding
```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods
  namespace: default
subjects:
- kind: User
  name: user1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
```
curl -k -H "Authorization: Bearer user-token" https://ip:6443/api/v1/names
```
```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: read-pods-group
  namespace: default
subjects:
- kind: Group
  name: developers
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
```
```
kubectl get clusterrole -A # все роли
kubectl auth can-i get pods -n default --as user1 # проверяем что может пользователь user1
```
```
sudo kubectl config view # ищем под кем выполняем сейчас команды
users:
- name: kubernetes-admin

kubectl get clusterrolebinding cluster-admin -o yaml
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:masters

# добавим токен в kubeconfig
kubectl config get-contexts # какие контексты есть в kubeconfig
kubectl config set-context default-pod-reader --cluster=kubernetes-admin@cluster.local --user=user1
# создаем контекст default-pod-reader связывает кластер с пользователем
kubectl config set-credentials user1 --token=31ada4fd-adec-460c-809a-9e56ceb75269
# настроим пользовтеля
kubectl config use-context default-pod-reader # используем контекст
проверяем что может пользователь
```
