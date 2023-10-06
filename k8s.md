### Postgres в minikube
1. Установим и запустим minikube:
```
curl -Lo minikube https://storage.googleapis.com/minikube/releases/latest/minikube-linux-amd64 \
  && chmod +x minikube
sudo mkdir -p /usr/local/bin/
sudo install minikube /usr/local/bin/
export NO_PROXY=$no_proxy,$(minikube ip) //чтобы всё работало за прокси
minikube start --vm-driver=docker

curl -LO https://dl.k8s.io/release/`curl -LS https://dl.k8s.io/release/stable.txt`/bin/linux/amd64/kubectl
chmod +x ./kubectl
sudo mv ./kubectl /usr/local/bin/kubectl
kubectl version --client
```

2. Запустим веб-консоль:
```
minikube dashboard
```
3. Зададим namespace:
```
kubectl create namespace pg-kub1
kubectl config set-context --current --namespace=pg-kub1
```

4. Посмотреть все поды: 
```
kubectl get pods -A
```

5. Поды в текущем namespace:
```
kubectl get pods
```

6. Развернуть Postgres:
```
kubectl apply -f postgres.yaml
```

Вывод должен быть:
```
service/postgres created
statefulset.apps/postgres-statefulset created
```

7. Проверим, что появился под: 
```
kubectl get pods
```
Вывод:
```
NAME                     READY   STATUS    RESTARTS   AGE
postgres-statefulset-0   1/1     Running   0
```
Под с Postgres успешно развёрнут.

8. Выполним проброс портов:
```
minikube service postgres --url -n pg-kub1
```
В выводе будет указан IP и порт, которые можно использовать для подключения к PSQL.

9. Подключимся к psql, используя логин/пароль из манифеста:
```
psql -h 192.168.49.2 -p 32439 -U myuser -W myapp
```
Успешно подключено к psql:
```
psql (16.0 (Ubuntu 16.0-1.pgdg23.04+1))
Введите "help", чтобы получить справку.

myapp=#
```