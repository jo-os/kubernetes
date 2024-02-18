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

## Helm
Позиционирует себя как пакетный менеджер для кубернетеса.

**Helm 3** (2019)
- хранение конфигурации в Secrets
- к Go-шаблонам добавили Lua
- убрали Tiller

- Умеет рендерить YAML файлы из шаблонов
- Можно использовать как пакетный менеджер
- Декларативный
- Не нужно ставить ничего в кластер дополнительно
- Поддерживает плагины

```bash
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh
```
```
helm repo add stable https://charts.helm.sh/stable - добавим репозиторий
helm repo list - проверим список репозиториев
helm repo update - обновим
helm search repo stable - список что есть в репозитории
```
Основные объекты Helm
- Chart - пакет информации необходимый для создания приложения в кластере kubernetes
- Репозиторий - группа chart'ов опубликованных удаленно
- Файл с параметрами - информация о настройках и параметрах, которые используются чартом для создания приложений и управления его релизами
- Релиз - работающий инстанс чарта связанный с определенным файлом с параметрами

Chart
- charts - помещены зависимости чарта (чарты которые будут установлены с основным)
- templates - содержит шаблоны из которого будут создаваться манифесты
- Chart.yml - содержит метаинформацию о чарте (имя, версия, документация...)
- values.yml - содержит значения для шаблонов
- licens - лицензии чарта
- readme - информация для пользователей чарта

Hub репозиториев - artifacthub.io - можно поискать нужные чарты и получить инфо где они

Releases - конкретная конфигурация чарта в кластере - при обновлении или изменении чарта создается новый релиз, что позволяет осуществить rollback
```
helm search hub grafana --max-col-width 80
https://artifacthub.io/packages/helm/grafana/grafana
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm show values grafana/grafana --version 7.3.0 > values.yml
kubectl create ns monitoring
helm upgrade --install grafana grafana/grafana -f values.yml -n monitoring --version 7.3.0
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
export POD_NAME=$(kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace monitoring port-forward $POD_NAME 3000 --address='0.0.0.0'

helm history grafana -n monitoring
helm rollback grafana 1 -n monitoring
```
```
sudo helm create billing
helm template .
https://artifacthub.io/packages/helm/bitnami/postgresql
```
<details>
  <summary>Helm</summary>

Chart.yml
```yml
apiVersion: v2
name: billing
description: A Helm chart for my app

type: application

version: 1.0.0

appVersion: "1.0.0"

dependencies:
  - name: postgresql
    version: "14.0.3"
    repository: oci://registry-1.docker.io/bitnamicharts
```
values.yml
```yml
replicaCount: 1

image:
  name: jooos/test-server
  tag: v1.0

envs:
  - name: DEBUG
    value: "True"
  - name: DATABASE_URL
    value: postgresql://user:password@postgres/db

service:
  port: 8080

postgresql:
  global:
    postgresql:
      postgresqlDatabase: db
      postgresqlUsername: user
      postgresqlPassword: password
  fullnameOverride: postgres
```
deployment.yml
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: {{ include "billing.fullname" . }}
  labels:
    {{- include "billing.labels" . | nindent 4 }}
spec:
  replicas: {{ .Values.replicaCount }}
  selector:
    matchLabels:
      {{- include "billing.selectorLabels" . | nindent 6 }}
  template:
    metadata:
      labels:
        {{- include "billing.selectorLabels" . | nindent 8 }}
    spec:
      initContainers:
        - name: check-db-ready
          image: postgres:9.6
          command: [ 'sh', '-c', 'until pg_isready -h postgres -p 5432;
          do echo database is not ready; sleep 2 done;']
      containers:
        - name: billing
          image: {{ .Values.image.name }}:{{ .Values.image.tag }}
          env:
            {{- with .Values.envs }}
            {{- toYaml . | nindent 10 }}
            {{- end }}
```
service.yml
```yml
apiVersion: v1
kind: Service
metadata:
  name: {{ include "billing.fullname" . }}
  labels:
    {{- include "billing.labels" . | nindent 4 }}
spec:
  type: {{ .Values.service.type }}
  ports:
    - port: {{ .Values.service.port }}
      targetPort: http
      protocol: TCP
      name: http
  selector:
    {{- include "billing.selectorLabels" . | nindent 4 }}
```
</details>

## Запросы и лимиты
```yml
resources:
  requests: # ресурсы запрашиваемые контейнером
    memory: "64Mi"
    cpu: "250m"
```
```yml
resources:
  requests: 
    limits: "128Mi" # лимиты установленные для контейнера
    cpu: "500m"
```
pod.yml
```yml
apiVersion: v1
kind: Pod
metadata:
  name: web
spec:
  containers:
    - name: nginx
      image: nginx
      resources:
        requests:
          memory: "50Mi"
          cpu: "100m"
        limits:
          memory: "100Mi"
          cpu: "500m"
```
QoS-классы
- Best Effort - самый низкий приоритет - без заданых requests, limit 1 на удаление 
- Burstable - поды с одним контейнером, или под где несколько контейнеров и один с requests но без лимиты requests < limit - 2ые на удаление
- Guaranteed - requests = limit - контейнер получает запрошенное, но не может получить больше

**LimitRange** - можно задать возможный диапазон ресурсов

Можно устанавливать:
- Минимальные и максимальные значения CPU/памяти
- Минимальные и максимальные дисковые запросы
- Указывать соотношения между requests и limits
- Устанавливать значения по умолчанию для requests/limits

```yml
apiVersion: v1
kind: LimitRange
metadata:
  name: resource-limits
spec:
  limits:
    - type: Pod
      max:
        cpu: "2"
        memory: 1Gi
      min:
        cpu: 200m
        memory: 6Mi
    - type: Container
      max:
        cpu: "2"
        memory: 2Gi
      min:
        cpu: 100m
        memory: 4Mi
      default:
        cpu: 300m
        memory: 200Mi
    - type: PersistentVolumeClaim
      min:
        storage: 1Gi
      max:
        storage: 10Gi
```
```
sudo kubectl describe limitrange
sudo kubectl delete limitrange resource-limits
```
**Priority-классы** - показывает важность пода относительно других подов
```yml
apiVersion: scheduling.k8s.io/v1
kind: PriorityClass
metadata:
  name: high-prioriry
value: 1000
```
```yml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-high-priority
spec:
  replicas: 20
  selector:
    matchLabels:
      app: nginx-high-priority
  template:
    metadata:
      labels:
        app: nginx-high-priority
    spec:
      priorityClassName: high-priority
      containers:
        - name: nginx-high-priority
          image: nginx
          resources:
            requests:
              memory: "200Mi"
```
**ResourceQuota** - Устанавливает ограничение на использование ресурсвом в неймспейсе.

Может ограничивать:
- общее число объектов определенных типов:
  - persistenvolumeclaims
  - secrets
  - services
  - configmaps
  - replicationscontrollers
  - deployments.apps
  - replicasets.apps
  - statefulsets
  - jobs
  - pods
- CPU/memory/storage для всех объектов в неймспейсе

```yml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources-with-priority
spec:
  hard:
    pods: "4"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
```
Квоты только на поды с классом low-priority
```yml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: compute-resources
spec:
  hard:
    pods: "4"
    requests.cpu: "1"
    requests.memory: 1Gi
    limits.cpu: "2"
  scopeSelector:
    matchExpressions:
      - operator: In
        scopeName: PriorityClass
        values: [ "low-priority" ]
```
```
kubectl describe resourcequotas # посмотреть квоты
```
## Автоматическое масштабирование
- что масштабировать - ноды, поды, запросы ресурсов
- когда масшатабировать - нужен metrics server (собирает информацию с kubelet)


Metrics server (-n kube-system)
```
kubectl apply -f https://raw.githubusercontent.com/pythianarora/total-practice/master/sample-kubernetes-code/metrics-server.yaml
```
Pod Disruption Budget (PDB) - это минимальное количество реплик приложения  которое кубернетес будет сохранять для работы, что полезно при автомасштабировании нод в кластере
```
kubectl drain node7 --ignore-daemonsets # drain - осушать - для вывода ноды из кластера или обновления на ноде компонентов - отключает ноду и на нее ничего не ставится

kubectl create pdb pdbdemo --min-available 2 --selector "app=nginx" # drain не может сразу удалить все поды, пока не запустятя минимум нужных нод

kubectl uncordon node7 - возвращет ноду в строй после drain
```

**Horizontal Pod Autoscaler (HPA)**
- Стардартные metrics.k8s.io (metrics-server)
- Метрики от адаптера в кластере custom.metrics.k8s.io (Prometheus Adapter, Microsoft Azure Adapter, Google Stackdriver)
- Метрики от внешней системы external.metrics.k8s.io (AWS CloudWatch)

```yml
apiVersion: autoscaling/v2beta1
kind: HorizontalPodAutoscaler
metadata:
  name: server-hpa
spec:
  scalaTargetRef: # указываем цель которую будем масштабировать - ReplicaSet, Deployment
    apiVersion: apps/v1
    kind: Deployment
    name: server
  minReplicas: 2
  maxReplicas: 10
  metrics: # на основе какой метрики масштабируемся
    - type: Resource
      resource:
        name: cpu
        targetAverageValue: 50m
```
**Vertical Pod Autoscaling (VPA)**
- масштабирует вертикально
- устанавливает ресурсные запросы и лимиты в контейнерах
- устанавливается в кластер отдельно (ссылка git ниже)

```yml
apiVersion: autoscaling.k8s.io/v1
kind: VerticalPodAutoscaler
metadata:
  name: server-vpa
spec:
  targetRef: # указываем цель (Deployment, Statefullset, Cronjob, Daemonset)
    apiVersion: apps/v1
    king: Deployment
    name: server
  updatePolicy:
    updateMode: Auto # off, initial, Recreate, Auto
```
Режимы работы:
- off - работа в рекомендательном режиме, рекомендации можно найти в поле статус объекта VPA
- initial - VPA устанавливает request только при создании пода и не меняет их потом
- Recreate - VPA устанавливает request при создании и обновляет для существующих подов пересоздавая их
- Auto - старается обновлять request при перезапуске подов (в бете)

```
git clone https://github.com/kubernetes/autoscaler.git
cd /autoscaler/ertical-pod-autoscaler/
./hack/vpa-up.sh
```
**Cluster Autoscaler**
- создает и удаляет ноды в кластере
- работает в облаках
- реализация зависит от конкретного провайдера
- для продакшена необходим PDB

**Общая схема работы** - через заданный интервал времени (по умолчанию 10 секунд) cluster autoscaling проверяет поды в статусе pending - если такие поды есть, то сервис оценивает реквесты на cpu и памяти этих подов и разварачивает то количество нод которое покроет запросы, если запросы не указаны, то ноды будут добавляться по одной. Для подсчетов cluster autoscaler использует только заданные запросы для подов и не оценивает реальную утилизацию ресурсов.

## Мониторинг
Что мониторить:
- Хосты
- Кластер
  - Элементы управления
  - etcd
  - Деплойменты/поды
- Приложения

**Типы метрик**
- counter - счетчик - значения, учеличивающиеся со временим
- gauge - шкала - значения могут увеличиваться или уменьшаться со временем
- histogram - гистограмма - хранит информацию об изменении некоторого параметра в течении определенного времени
- summary - сводка результатов - расширенная гистограмма, которая также позволяет рассчитывать квантили для скользящих временных интервалов

Подготовка (решение проблем с PV)
- создание nfs сервера - https://creodias.docs.cloudferro.com/en/latest/kubernetes/Create-and-access-NFS-server-from-Kubernetes-on-Creodias.html
- установка на ноды - sudo apt install nfs-common
- установка через helm nfs provisioner - https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner
- запуск prometheus с nfs - https://github.com/prometheus-community/helm-charts/issues/2313
```
helm install prometheus prometheus-community/prometheus --set server.persistentVolume.storageClass=nfs-client --set alertmanager.persistentVolume.storageClass=nfs-client
```

**Prometheus**
```
helm search hub prometheus --max-col-width 80
https://artifacthub.io/packages/helm/prometheus-community/prometheus
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
kubectl create ns monitoring
helm upgrade --install prometheus prometheus-community/prometheus -n monitoring
export POD_NAME=$(sudo kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=prometheus,app.kubernetes.io/instance=prometheus" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace monitoring port-forward $POD_NAME 9090 --address='0.0.0.0'
```

**Конфигурация Prometheus** - в ConfigMap prometheus-server в установленном namespace

Секции конфиг файлы
- global - глобальные настройки для всех целей (интервал скрейпинга)
- rule_files - список директорий где лежат правила, которые необходимо загружать
- alerting - правило по которому prometheus должен находить алерт менеджер в кластере
- scrape_configs - настройки поиска целей для мониторинга

Scrape_config - в этой секции указан список scrape заданий, в каждом из которых настраиваются правила нахождения целей для мониторинга (цели можно указывать статически, но есть механизм автонахождения целей)

Вместе с настроенным механизмом автонахождения указваем в описании аннотации.

```yml
service:
  type: clusterIP
  port: 80
  annotations:
    prometheus.io/scrape: 'true' # говорит prometheus что нужно влючить эндпоинты этого сервиса в список целей для сбора метрик
    prometheus.io/path: '/v1/metrics' # переопределяем путь для сбора метрик - если не указан то стандартный путь /metrics
    prometheus.io/port: 8080 # указываем порт по которому будет производиться скрейпинг
```
В общем алгоритм автонахождения целей выглядит так - prometheus читает секции config и jobs согласно которым настраивает свой механизм автообнаружения сервисов. Этот механизм взаимодействует с kubernetes api из которого получает список подходящих целей, на основании этих данных мезанизм автообнаружения обновляет список целей targets - таким образом prometheus сам отслеживает добавление и удаление подов, так как при добавлении и удалении подов kubernetes изменяет endpoints а prometheus это замечает и удаляет/добавляет свои цели.
```
kubectl describe service -n kube-system coredns
```
```
Name:              coredns
Namespace:         kube-system
Labels:            addonmanager.kubernetes.io/mode=Reconcile
                   k8s-app=kube-dns
                   kubernetes.io/name=coredns
```
```
kubectl tmp-pod run --rm -it --image nicolaka/netshoot -n monitoring -- /bin/bash
curl http://prometheus-prometheus-node-exporter:9100/metrics
```
**Alertmanager**
```
kubectl edit cm -n monitoring prometheus-server # добавляем алерт
```
```
alerting_rules.yml: |
groups:
  - name: prometheus-app
    rules:
      - alert: NoPushGateway
        expr: up{job="promehteus-pushgateway"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          description: PashGateway is down
          summary: PashGateway is down
```
Slack
```
apiVersion: v1
kind: ConfigMap
metadata:
  name: alertmanager-config
  namespace: monitoring
  labels:
    app: prometheus
    app.kubernetes.io/managed-by: Helm
    chart: prometheus-15.0.1
    component: alertmanager
    heritage: Helm
    release: prom
  annotations:
    meta.helm.sh/release-name: prom
    meta.helm.sh/release-namespace: monitoring
data:
  alertmanager.yml: |
    global:
      resolve_timeout: 1m
      slack_api_url: 'https://hooks.slack.com/services/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'
    receivers:
    - name: 'slack-notificaions'
      slack_configs:
      - channel: '#notifications'
        send_resolved: true
    route:
      receiver: 'slack-notificaions'
```
**Grafana**
```
https://artifacthub.io/packages/helm/grafana/grafana
sudo helm install grafana grafana/grafana -n monitoring --set grafana.persistentVolume.storageClass=nfs-client ## test
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode ; echo
export POD_NAME=$(sudo kubectl get pods --namespace monitoring -l "app.kubernetes.io/name=grafana,app.kubernetes.io/instance=grafana" -o jsonpath="{.items[0].metadata.name}")
kubectl --namespace monitoring port-forward $POD_NAME 3000 --address='0.0.0.0'

Prometheus Server URL:   http://prometheus-server
```
Prometheus библиотеки официальные есть:
- Go
- Java or Scala
- Python (например prometheus_flask_exporter)
- Ruby

**Сбор логов в Kubernetes:**
- Нет истории
- Теряются при удалении контейнера или ноды

Системы логирования:
- Управляемый облачный сервис (AWS, Azure, Google)
- SaaS - сервис мониторинга (DataDog, NewRelic)
- Свой сервис - ELK, Loki, Graylog
```
helm search hub loki-stack --max-col-width 80
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
helm upgrade --install loki grafana/loki-stack -n monitoring
```
## Service mesh
**Монолит**

**Плюсы**
- легче реализовать
- легче тестировать
- легко деплоить и масштабировать
- сильня связность

**Минусы**
- замедляется пароцесс разработки
- масштабировать сложнее со временем
- нет изоляции

**Микросервисы**

**Плюсы**
- разные сервисы принадлежат разным командам
- не привязаны друг к другу технологически
- слабая связность компонентов

**Минусы**
- появляется межсетевое взаимодействие
- сложнее тестировать
- нужно дублировать функциональность

**Межсервисное взаимодействие**
- выбор протокола для взаимодействия
- service discovery (чтобы сервисы находили друг друга)
- rate limiting (для защиты от ddos атак)
- компонент автоматических повторов (для восстановления соединения после обрыва связи)
- безопасность (аутентификация, шифрование данных которе ходят между сервисами)
- трейсинг запросов (для наглядности - наблюдать все на графиках)

**Как решить:**
- написать клиентские библиотеки (надо писать под каждый язык)
- поставить прокси перед каждым микросервисом

**Service  mesh** - выделенный инфраструктурный уровень, который добавляет функии(например:) в сеть между сервисами
- Балансировка трафика
- TLS termination
- HTTP/2 и gRPC прокси
- Circuit breakers - автоматический предохранитель в случае падения сервиса, - плохой или сбоящий сервис будет выключен
- Health check
- Blue/Green деплойменты
- Метрики и трассировка

**envoy** - как прокси сервер - динамический - есть протокол который позволяет ежесекундно добавлять изменения в конфигурацию.

Какие service mesh существуют:
- Istio
- Linkerd
- Consul
- другие - Traefik Mesh, Kuma, AWS App Mesh

**Istio архиктура**
- Data plane (уровень данных) - это множество прокси серверов, которые располагаются перед контейнерами приложений и получают небходимые настройки из control plane чтобы управлять трафиком
- Controle plane (уровень управления)

Контейнер с istio:
- istio-init - инит контейнер который запускается перед запуском всего пода и настраивает правила iptables таким отбразом чтобы вест входящий трафик шел в sidecar
- sidecar injection - через label istio-injection=enabled istio с помощью вебхука будет менять спецификацию всех новых подов этого namespace добавляя контейнер с envoy 

Для конфигурирования invoy используются компоненты control plane с версии 1.6 стали одним бинарником istiod:
- Pilot - центральный компонент, отвечающий за коммуникацию с sidecar используя invoy api, он считывает правила описанные в манифестах istio и отправляет их в envoy для настройки прокси сервисов. Pilot отвечает за service discovery, routing, управление трафиком, устойчивость сети 
- Citadel - identity и access manager, шифрование трафика, аутентификация сервисов и пользователей, управление ключами шифрования
- Galley - компонент для управления кофигурацией, валидация обработки новых настроей, отправка их остальным компонентам системы

**Способы установки Istio**
- istioctl
  - install (есть доп проверки - для выявления проблем)
  - генерация манифеста (без доп проверок)
- helm (без доп проверок)
- istio operator (не рекомендуется для новых инсталляций)

https://istio.io/latest/docs/setup/getting-started/

profiles - https://istio.io/latest/docs/setup/additional-setup/config-profiles/
```
istioctl install --set profile=demo -y
kubectl get all -n istio-system
istioctl kube-inject -f pod_nginx.yml > pod_nginx_istio.yml # получаем yml с добавленым istio sidecar
nsenter -t pid -n iptables -t nat -S - утилита для просмотра правил по pid
```
**Istio Custrom Resource**

Istio ingress, egress gateways - 
- комбинация объектов Service и Deployment
- Ingress gateway служит получения трафика извне кластера и его роутинга внутри service mesh
- Egress gateway служит для обработки исходящего трафика

Gateway - для конфигурации ingress gateway
- конфигурирует istio ingress на 4-6 уронвнях
- настраивает TLS
- Открываются порты и настраиваются протоколы

Vortual Service - с его помощью настраиваем маршрутизацию трафика
- Конфигурирует L7 уровень
- Настраивает роутинг на конкретные сервисы
- Можно добавлять - таймауты, ретраи, балансировать трафик по процентам

Destination Rules - с его помощью можно группировать поды по какому то признаку в группы
- Настривает поведение трафика после выполнения роутинга
- Группирует версии приложения в сабсеты
- Добавляет mTLS

**Общая схема работы:**
- LoadBalancer
- Istio Service
- Поды Istio ingress
- Поды настраиваются с помощью:
  -  Gateway
  -  VirtualService
- Istio Ingress отправляет пакет в приложение

**Практика с демо приложением BookInfo**
```
kubectl label namespace default istio-injection=enabled
kubectl run nginx --image=nginx
kubectl describe pod nginx
```
https://ru.linux-console.net/?p=20192 - MetalLB - как внешний балансировщик для железа (нужен для istio ingress-gateway)
```
kubectl apply -f istio-1.0.0/samples/bookinfo/platform/kube/bookinfo.yaml 
kubectl get po
kubectl get service -n istio-system # смотрим внешний адрес - не нем нет ответа, так как там не слушают - надо настроить gateway 
kubectl describe service -n istio-system istio-ingressgateway # смотрим цепочку обработки нашего запроса
```
gw.yml - говорим слушать от всех на 80 порту
```yml
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: bookinfo-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "*"
```
```
kubectl create -f gw.yml
получаем на внешнем ip 404 - слушают, но не настроен ответ
kubectl get po -n istio-system
kubectl logs -n istio-system istio-ingressgateway-54ccd8b799-62m6z # видим 404 - теперь надо перенаправить трафик в сервисы
```
vs_productpage.yml
```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: bookinfo
spec:
  hosts:
  - "*"
  gateways:
  - bookinfo-gateway
  http:
  - match:
    - uri:
        exact: /productpage
    - uri:
        prefix: /static
    - uri:
        exact: /login
    - uri:
        exact: /logout
    - uri:
        prefix: /api/v1/products
    route:
    - destination:
        host: productpage
        port:
          number: 9080
```
```
http://192.168.0.50/productpage
kubectl describe service reviews - видим что запросы идут одинаково на все endponts (v1,v2,v3)
```
dr.yml - определяем labels для подов
```yml
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: reviews
spec:
  host: reviews
  trafficPolicy:
    tls:
      mode: DISABLE
  subsets:
  - name: v1
    labels:
      version: v1
  - name: v2
    labels:
      version: v2
  - name: v3
    labels:
      version: v3
```
vs_reviews.yml - направляем все только на v1
```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
```
```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v1
      weight: 80
    - destination:
        host: reviews
        subset: v3
      weight: 20
```
По заголовку - end-user: qa-test - перенаправляем все на v3
```yml
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: reviews
spec:
  hosts:
    - reviews
  http:
  - match:
    - headers:
        end-user:
          exact: qa-test
    route:
    - destination:
        host: reviews
        subset: v3
  - route:
    - destination:
        host: reviews
        subset: v1
```

https://istio.io/latest/docs/reference/config/networking/gateway/#ServerTLSSettings-TLSmode

ISTIO_MUTUAL - istio организует все сам - включается в DestinationRule
```
  trafficPolicy:
    tls:
      mode: ISTIO_MUTUAL
```
