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
## Безопасность

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
<details>
  <summary>RoleBinding User</summary>
  
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
</details>

```
curl -k -H "Authorization: Bearer user-token" https://ip:6443/api/v1/names
```
<details>
  <summary>RoleBinding Group</summary>
  
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
</details>

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
11

Capabilities

Отнимем у nginx право на выполнение привилегированного действия - слушать до 1024 порты
```
docker run --network=host --cap-drop CAP_NET_BIND_SERVICE nginx
```
Secutiry Context - настройки которые определяют привилегии и доступы которые будут иметь под и его контейнер

<details>
  <summary>securityContext</summary>
  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: ecurity-context-demo
spec:
  securituContext:
    runAsUser: 1000
    runAsGroup: 3000
  containers:
  - name: sec-ctx-demo
    image: busybox
    commsnd: ["sh","-c","sleep 1h"]
    SecutiryContext:
      capabilities:
      add: ["NET_ADMIN", "SYS_TIME"]
```
</details>

<details>
  <summary>Role+RoleBinding</summary>
  
```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: pod-creator
rules:
- apiGroups: [""]
  resources: ["pods", "pods/exec"]
  verbs: ["get", "watch", "list", "create"]
```
```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: create-pods
  namespace: default
subjects:
- kind: User
  name: user1
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-creator
  apiGroup: rbac.authorization.k8s.io
```
</details>

<details>
<summary>Hack Pod</summary>
    
```yml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-hack
spec:
  containers:
  - name: nging
    image: nginx
    volumeMounts:
    - mountPath: /master
      name: master
  volumes:
  - hostPath:
      path: /
      type: Directory
    name: master
  tolerations: # - игнорируем существует ли taint NoSchedule
  - effect: NoSchedule
    operator: Exists
  nodeSelector: # - хотим запускаться только на мастерах
    node-role.kubernetes.io/control-plane: ""
```
</details>

```
kubectl describe nodes - посмотреть описания и метки подов
kubectl label nodes snowflake3 disk=hdd - добавить метку
kubectl run nginx --image=nginx -n default --as=user1
kubectl create -f hack-pod.yml
kubectl exec -it nginx-hack -n default -- bash
копируем содержимое admin.conf из /master/etc/kuberbetes/admin.conf
kubectl --kubeconfig=config.yml auth can-i "*" "*" - проверяем права с этим файлом - можем все в кластере
```
**PodSecutiryPolicy** - проверяет спецификацию пода на соответствие заданным требованиям (устарело)

**Pod Security Policies (PSP)** - в релизе Kubernetes 1.25 политики безопасности подов полностью удалены. Сейчас мы сосредоточимся только на допуске к безопасности пода. PSA включен по умолчанию в Kubernetes 1.23, однако нам нужно будет указать, какие поды должны соответствовать стандартам безопасности. Все, что вам нужно сделать, чтобы включить функцию PSA, — это добавить метку определенного формата в пространство имен. Все поды в этом пространстве имен должны будут следовать заявленным стандартам.
```
metadata:
  labels:
    pod-security.kubernetes.io/enforce: restricted
```
Мод может в свою очередь принимать три значения:
- enforce: Нарушения приведут к отклонению запуска пода,
- audit: Создание пода будет разрешено. Нарушения будут добавлены в лог аудита,
- warn: Создание пода будет разрешено. Нарушения будут отображаться на консоли.

Также у нас есть политики безопасности, определяемые уровнем, установленным для PSA:
- privileged: Полностью неограниченная политика,
- baseline: Минимально ограничительная политика, которая охватывает важные стандарты,
- restricted: Жестко ограниченная политика в соответствии с рекомендациями по усилению защиты подов с точки зрения безопасности.
```yml
apiVersion: v1
kind: Namespace
metadata:
  name: psa-restricted # - в этом namespace ничего не запустится по безопасности
  labels:
      pod-security.kubernetes.io/enforce: restricted
```
```
kubectl get clusterrole
kubectl get rolebindings -n kube-system
```
## Сеть
12

Модель сети Kubernetes:
- контейнеры внутри пода имеют один общий уникальный адрес
- поды могут общаться с другими подами исользуя их ip без nat
- ip который видит контейнер должен быть таким для всех

На поде создается сетевой namespace и контейнера на поде делять его. Перед созданием пода кубер создает сетевой пространсво имен для этого пода и держит его специальным служебным контейнером (является родительским контейнером для всех контейнеров пода) - поэтому все будет жить пока не убить под. Контейнеры используют общий сетевой namespace они можут общаться через localhost напрямую по портам

Для того чтобы связывать сетевые namespace в linux есть virtual ethernet device (veth) - он сосотоит из 2х виртуальных интерфейсов которые можно подключить к разным namespace. Получаем трубу между namecpase пода (eth0) и хоста (veth000) - далее взаимодействие подов идет через сетевой мост - и поиск нужного устройства через ARP.

**Поды на разных хостах**

IP подов должны быть уникальны по всему кластеру (это требование к сети kubernetes). Кубер не пытается настроить сеть между подами сам (много вариантов и систем) - делегирует это плагинам CNI (Container Network Interface). CNI - это стандарт для конфигурирования сети, который используется в kubernetes. Плигин должен иметь методы по спецификации - ADD, DEL, CHECK, VERSION

CNI плагины
- Flannel
- Calico
- Waeve Net
- Cilium

```
kubectl get daemonset -A # - смотрим все демонсеты
kubectl get pods -A -o wide | grep calico # - смотрим все поды по сетям
```
Внутри кластера любой под доступен любому другому по сети - это уязвимость закрывается сетевыми политиками - NetworkPolicy

Запретить все подключения
```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny
  namespace: default
spec:
  podSelector: {}
```
<details>
  <summary>Разрешить подключение к БД</summary>
  
```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-allow
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: app
    ports:
    - port: 5432
```
</details>

<details>
  <summary>Разрешить подключение из namespace billing</summary>

```yml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: ns-default-allow
  namespace: default
spec:
  podSelector:
    matchLabels:
      app: database
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          ns: billing
    ports:
    - port: 5432
```
</details>
  
Kube-proxy - отвечает за перенаправление запросов к соответствующим сервисам в приватной сети кластера.
- конфигурирует правила сети на узлах
- обычно использует iptables на воркер нодах

Режимы работы
- user space - в это режиме kubeproxy пропускает через себя трафик и маршрутизирует на нужные поды (устарел)
- iptables - kubeproxy настраивает правила на ноде и таким образом перенаправляет и балансирует трафик на поды
- ipvs - ip virtual server - kubeproxy использует инструмент ядра линукс ipvs

serv.yml
```yml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```
```
kubectl run nginx1 --image nginx --labels="app=my-app"
kubectl run nginx2 --image nginx --labels="app=my-app"
kubectl create -f serv.yml
kubectl get po -o wide
kubectl describe service my-service
```
Способы опубликовать сервисы наружу

- С помощью kubectl локально
  - kubectl proxy --port=8080
  - kubectl port-forward service/my-service 10000:80
- Сервис NodePort
- Сервис LoadBalancer
- Создать ресурс Ingress
```
kubectl proxy --port=8080
kubectl port-forward service/my-service 10000:80
```
**NodePort** - каждая нода кластера открывает порт на своем внешнем интерфейсе и перенаправляет трафик в требуемый сервис - curl [любая нода]:30007
```yml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  type: NodePort
  selector:
    app: my-app
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30007
```
**LoadBalancer** - расширение типа NodePort и используется с облачными провайдерами - внешняя инфраструктура облачного провайдера узнает из api кубернетеса об этом и создает выделенный балансировщик нагрузки, который перенаправляет трафик на порты воркер нод

Нужен внешний ip - когда появится, то сервис сразу его использует
```yml
apiVersion: v1
kind: Service
metadata:
  name: my-service
spec:
  selector:
    app: my-app
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
  type: LoadBalancer
```
**Ingress** - набор правил внутри кластера, предназначены для того чтобы входящие поключения могли достич сервиса приложений, работает на 7 уровне OSI. Понимает заголовки http запросов. Один Ingress может предоствлять доступ ко множеству служб.

<details>
  <summary>Ingress</summary>
  
```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
  nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
  - host: hello-world.com
    http:
      paths:
      - path: /svc1
        pathType: Prefix #ImplementatioSpecific | Exact | Prefix
        backend:
          service:
            name: my-service
            port:
              number: 80
```
</details>
  
spec - правила по которым ingress контроллер будет маршрутизировать трафик. В данном случае - проксировать все входящие запросы, которые приходят на хост hello-world.com на endpoint /svc1 на сервис my-service с портом 80. 

pathType - тип пути (обязательный параметр)
- ImplementatioSpecific - настраиваемый в зависимости от реализации тип
- Exact - url путь должен точно совпадать
- Prefix - url путь должен подходить под заданный префикс

```
kubectl run nginx1 --image=nginx --labels="app=app"
kubectl expose pod nginx1 --port 80 --target-port 80
sudo kubectl describe service nginx1
sudo kubectl create -f ingress.yml
```
<details>
  <summary>Ingress 2 path</summary>
  
```yml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: demo-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - http:
        paths:
        - path: /
          pathType: Prefix
          backend:
            service:
              name: nginx1
              port:
                number: 80
    - http:
        paths:
        - path: /apache
          pathType: Prefix
          backend:
            service:
              name: apache
              port:
                number: 80
```
</details>

```
kubectl describe ingress demo-ingress
https://coderoad.ru/44110876/%D0%92%D0%BD%D0%B5%D1%88%D0%BD%D0%B8%D0%B9-IP-%D0%B0%D0%B4%D1%80%D0%B5%D1%81-%D1%81%D0%BB%D1%83%D0%B6%D0%B1%D1%8B-Kubernetes-%D0%BE%D0%B6%D0%B8%D0%B4%D0%B0%D0%B5%D1%82%D1%81%D1%8F
```
**Ingress controller** - приложение занимается обработкой трафика, настройка формируется из ingress правил - которые описаны в ingress объектах

**Ingress контроллеры:**
- Ingress Kubernetes
- Ingress nginx
- Traefik
- HAProxy
- Kong

## Хранилище данных в Kubernetes

**Типы volumes**
- in-tree - плагины разрабатываются и поставляются вместе с бинарниками kubernetes, сразу доступны для использования, без дополнительных установок, они уже протестированы и стабильны
- csi - container storage interface - разные вендоры могут сами писать плагины, плагины потом устанавляваются в кластер отдельно

**Плагины in-tree**
- Ephemeral (эфимерные) - жизнь = жизни пода
  - emptyDir - простой пустой каталог, для хранения временных данных, может сохранять данные как на диске так и в памяти
  - configMap, secret - для монтирования в под ресурсов кубернетес
  - downwardApi - можно смонтировать информацию о поде в котором этот контейнер работает
- Persistent (постоянные) - данные сохраняются независимо от состояния пода
  - awsElasticBlockStore, azureDisk, gcePersistentDisk - хранение облачных провайдеров
  - hostPath (монтирование папок из файловой системы ноды) nfs, iscsi, rbd - 
  - PersistenVolumeClaim - способ использовать заранее созданное администратором или динамически резервируемое постоянное хранилище
 
Сторонние CSI плагины - https://kubernetes-csi.github.io/docs/drivers

 <details>
  <summary>hostPath + emptyDir</summary>
   
```yml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: my-pod
      image: debian
      command: ["/bin/sh"]
      args: ["-c", "sleep 3600"]
      volumeMounts:
        - name: slow-persistent-volume
          mountPath: /mnt/slow-persistent
        - name: fast-ephemeral-volume
          mountPath: /mnt/fast-ephemeral
  volumes:
    - name: slow-persistent-volume
      hostPath:
        path: "/mnt/hostpath"
    - name: fast-ephemeral-volume
      emptyDir: {
        medium: Memory
      }
```
</details>

```
for dir in /mnt/*;do echo $dir; dd if=/dev/zero of=$dir/test.delete bs=128k count=10000;done
```
<details>
  <summary>Shared data 2 pods</summary>
  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
  labels:
    app: web-server
spec:
  restartPolicy: Never

  volumes:
    - name: shared-data
      emptyDir: { }
  
  containers:
    - name: first-container
      image: nginx
      volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
    - name: second-container
      image: debian
      volumeMounts:
        - name: shared-data
          mountPath: /pod-data
      command: [ "/bin/sh" ]
      args: [ "-c", "while true; do date > /pod-data/index.html 2>&1 ;sleep 2;done" ]
```
</details>

```
kubectl exec -it two-containers -c second-container -- bash
 - tail -f /pod-data/index.html 
kubectl exec -it two-containers -c first-container -- bash
 - while true; do curl http://localhost; sleep 2; done
```
nginx.conf
```
server {
    listen 8080 default_server;
    server_name _;
    root /usr/share/nginx/html;

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```
```
kubectl create configmap nginx-config --from-file=nginx.conf
```
<details>
  <summary>nginx.conf with ConfigMap</summary>
  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: two-containers
  labels:
    app: web-server
spec:
  restartPolicy: Never

  volumes:
    - name: shared-data
      emptyDir: { }
    - name: nginx-config
      configMap:
        name: nginx-config
  
  containers:
    - name: first-container
      image: nginx
      volumeMounts:
        - name: shared-data
          mountPath: /usr/share/nginx/html
        - name: nginx-config
          mountPath: /etc/nginx/conf.d
          readOnly: true
    - name: second-container
      image: debian
      volumeMounts:
        - name: shared-data
          mountPath: /pod-data
      command: [ "/bin/sh" ]
      args: [ "-c", "while true; do date > /pod-data/index.html 2>&1 ;sleep 2;done" ]
```
</details>

```
kubectl exec -it two-containers -c first-container -- bash
 - nginx -T
 - while true; do curl http://localhost:8080;sleep 2; done
```

Для того чтобы приложениям самим запрашивать хранилище, без взаимодействия со спецификой инфраструктуры ввели:
- **PV** - постоянное хранилище
- **PV claim** - заявка на постоянное хранилище

**accessModes**: - монтируем тома с определенными правилами
- ReadWriteOnce - том может быть смонтирован на чтение и запись только одной нодой
- ReadOnlyMany - только чтение сразу несколькими нодами
- ReadWriteMany - чтение и запись несколькими нодами
- ReadWriteOncePod - чтение и запись одним подом (новый режим)

persistentVolumeReclaimPolicy: - политика определяет что будет с PV после того как он будет использован и использовавший его PVC будет удален
- Retain - оставить с данными
- Recycle - удаление данных
- Delete - будет удален

<details>
  <summary>PersistenVolume</summary>
  
```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: volume-slow
spec:
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/volume1"
```
</details>

<details>
  <summary>PersistenVolumeClaim</summary>
  
```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: pvc-slow
spec:
  storageClassName: ""
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```
</details>

<details>
  <summary>Pod + persistenVolumeClaim</summary>
  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: pod-pvc
spec:
  containers:
  - image: nginx
    name: web-server
    volumeMounts:
    - mountPath: /cache
      name: cache-volume

  volumes:
  - name: cache-volume
    persistentVolumeClaim:
      claimName: pvc-slow
```
</details>

**StorageClass** - необходим для автоматического управления томами

- **provisioner** - задает используемый provisioner - то есть способ которым StorageClass будет динамически создавать тома. Для облаяных провайдеров свои - kubernetes.io/aws-ebs, kubernetes.io/gce-pd...
- **allowVolumeExpansion** - разрешать или нет расширение диска
- **volumeBindingMode**
  - Immediate - Volume создается сразу после создания для него подходящей pvc
  - WaitForFirsCustomer - том не будет создан до того момента, пока не будет создан под его использующий
 
https://kubernetes.io/docs/tasks/configure-pod-container/configure-persistent-volume-storage/

<details>
  <summary>StorageCalss</summary>

```yml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
</details>

<details>
  <summary>PersistenVolume</summary>

```yml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: task-pv-volume
  labels:
    type: local
spec:
  storageClassName: manual
  capacity:
    storage: 10Gi
  accessModes:
    - ReadWriteOnce
  hostPath:
    path: "/mnt/data"
```
</details>

<details>
  <summary>PersistenVolumeClaim</summary>
  
```yml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: task-pv-claim
spec:
  storageClassName: manual
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 3Gi
```
</details>

<details>
  <summary>Pod + pvc</summary>
  
```yml
apiVersion: v1
kind: Pod
metadata:
  name: task-pv-pod
spec:
  volumes:
    - name: task-pv-storage
      persistentVolumeClaim:
        claimName: task-pv-claim
  containers:
    - name: task-pv-container
      image: nginx
      ports:
        - containerPort: 80
          name: "http-server"
      volumeMounts:
        - mountPath: "/usr/share/nginx/html"
          name: task-pv-storage
```
</details>

**Headless service**
- не имеет IP адреса
- не выполняет балансировку запросов
```
kubectl create deployment nginx --image=nginx --replicas 3
kubectl get po -o wide
kubectl expose deployment/nginx --name nginx-normal --port=80 --target-port=80 # обычный
kubectl expose deployment/nginx --name nginx-headless --port=80 --target-port=80 --cluster-ip="None" # headless
kubectl run tmp-pod --rm -i --tty --image nicolaka/netshoot -- /bin/bash
nslookup nginx-normal
nslookup nginx-headless
```
**StatefulSet** - обеспечивает постоянство сетевой идентичности, так как каждой реплике пода присваивается порядковый индекс с отсчетом от 0. Адрес определяется имя пода + headless service.

<details>
  <summary>Service + custerIP none</summary>
  
```yml
apiVersion: v1
kind: Service
metadata:
  labels:
    app: cassandra
  name: cassandra
spec:
  clusterIP: None
  ports:
    - port: 9042
  selector:
    app: cassandra
```
</details>

<details>
  <summary>StatefulSet</summary>
  
```yml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: cassandra
  labels:
    app: cassandra
spec:
  serviceName: cassandra
  replicas: 3
  selector:
    matchLabels:
      app: cassandra
  template:
    metadata:
      labels:
        app: cassandra
    spec:
      containers:
        - name: cassandra
          image: gcr.io/google-samples/cassandra:v13
          imagePullPolicy: Always
          env:
            - name: MAX_HEAP_SIZE
              value: 512M
            - name: HEAP_NEWSIZE
              value: 100M
            - name: CASSANDRA_SEEDS
              value: "cassandra-0.cassandra.default.svc.cluster.local"
            - name: CASSANDRA_CLUSTER_NAME
              value: "K8Demo"
            - name: CASSANDRA_DC
              value: "DC1-K8Demo"
            - name: CASSANDRA_RACK
              value: "Rack1-K8Demo"
          volumeMounts:
            - name: cassandra-data
              mountPath: /cassandra_data
  volumeClaimTemplates:
    - metadata:
        name: cassandra-data
      spec:
        accessModes: [ "ReadWriteOnce" ]
        resources:
          requests:
            storage: 1Gi
```
</details>
  
**Vault**

Что не так с секретами:
- Нет механизма динамического обновления
- Проблемы с отзывом
- Нет аудита
- По умолчанию хранится в открытом виде

Vault:
- Умеет шифровать данные
- Может динамически генерировать секреты
- Имеет удобные политики доступа
- Есть аудит-логи

Методы аутентификации:
- userpass
- token
- сертификаты
- kubernetes
- AWS
- LDAP

```
kubectl run vault --image=vault:1.13.3 --env="VAULT_DEV_ROOT_TOKEN_ID=8fb95528-57c6-422e-9722-d2147bcba8aa"
kubectl expose pod/vault --name vault --port=8200 --target-port=8200
kubectl port-forward vault 8200:8200 --address='0.0.0.0'

kubectl create serviceaccount vault-auth
kubectl describe clusterrole system:auth-delegator
```
```yml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
  - kind: ServiceAccount
    name: vault-auth
    namespace: default
```
```
export VAULT_ADDR=http://localhost:8200
vault login
8fb95528-57c6-422e-9722-d2147bcba8aa

vault kv get secret/myapp/config
```

https://tutorials.akeyless.io/docs/kubernetes-authentication

Kubernetes auth method

<details>
  <summary>Kuber + Vault</summary>
  
```
cat << EOF > akl_gw_token_reviewer.yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: gateway-token-reviewer
  namespace: default
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: role-tokenreview-binding
  namespace: default
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:auth-delegator
subjects:
- kind: ServiceAccount
  name: gateway-token-reviewer
  namespace: default 
EOF

kubectl apply -f akl_gw_token_reviewer.yaml

cat <<EOF > akl_gw_token_reviewer.yaml 
apiVersion: v1
kind: Secret
metadata:
  name: gateway-token-reviewer-token
  namespace: default
  annotations:
    kubernetes.io/service-account.name: gateway-token-reviewer
type: kubernetes.io/service-account-token
EOF

kubectl apply -f akl_gw_token_reviewer.yaml

SA_JWT_TOKEN=$(kubectl get secret gateway-token-reviewer-token \
  --output 'go-template={{.data.token | base64decode}}')

CA_CERT=$(kubectl config view --raw --minify --flatten  \
    --output 'jsonpath={.clusters[].cluster.certificate-authority-data}')


vault auth enable kubernetes

vault write auth/kubernetes/config \
    token_reviewer_jwt="$SA_JWT_TOKEN" \
    kubernetes_host=https://192.168.99.100:<your TCP port or blank for 443> \
    kubernetes_ca_cert="$CA_CERT"
```
</details>

