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

