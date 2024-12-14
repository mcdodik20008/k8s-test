Включаем кубер в настройках доккера
- Enable Kubernetes

# Устанавливаем kubectl
```bash
winget install -e --id Kubernetes.kubectl
```
проверяем установку
```bash
kubectl version --client
```
Еще можно чекнуть ~/.kube


# Настраиваем UI

Kubernetes Dashboard
```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
```
Dashboard требует аутентификации. Создадим пользователя с ролью ClusterAdmin. [dashboard-admin.yaml](dashboard-admin.yaml)
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: admin
  namespace: kubernetes-dashboard
```
И применим его
```bash
kubectl apply -f dashboard-admin.yaml
```
Выдадим ему роли [dashboard-admin-role.yaml](dashboard-admin-role.yaml)
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: admin
  namespace: kubernetes-dashboard
```
Применим его
```bash
kubectl apply -f dashboard-admin-role.yaml
```

Получим токен для входа 
```
kubectl -n kubernetes-dashboard create token admin
```
Прокидываем порт
```yaml
kubectl proxy
```
и подключаемся 
```
http://localhost:8001/api/v1/namespaces/kubernetes-dashboard/services/https:kubernetes-dashboard:/proxy/
```

Можно подключить отображение метрик подов (пока не подключал)
```
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml
```

# Разберемся с имагесами
Нужно залогиниться в доккере
```bash
docker login
```
Создать образы - можно использовать мой компоуз файл [docker-compose.yml](docker-compose.yml), только подмени в images мой ник на свой
```yaml
image: mcdodik2008/k8s-mastery-frontend
```
да и вообще создай репозитории в доккер хабе с таким же именем, как и имагесы контейнеров и лупани
```bash
docker push mcdodik2008/k8s-mastery-frontend
```
Имагесы должны появиться в доккер хабе, тогда кубернатес сможет их выкачать и запустить 
(у меня не получилось подсунуть ему локальные образы)

У нас есть образы, давай запилим подик

# Под для фронтенда
[frontend.yaml](frontend.yaml)
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
    - name: frontend
      image: mcdodik2008/k8s-mastery-frontend
      ports:
        - containerPort: 80
```
Заходим в UI и чекаем жива ли пода или пишем 
```bash
kubectl get pods
```
Если пода жива, то давай прокинем порт из кубера в наружу, например займем 88 порт
```bash
kubectl port-forward -n default frontend 88:80
```

Давайте перед входом во фронтс добавим лоад балансер

#Лоадбалансер
[frontend-lb-service.yaml](frontend-lb-service.yaml)
```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-lb
spec:
  type: LoadBalancer
  ports:
    - port: 80
      protocol: TCP
      targetPort: 80
  selector:
    app: frontend
```
применим манифест, теперь на 80 порту у нас сидит лоадбалансер 
и переадресует запрос в наш фронтенд на 80 порт. Прокидывать порт уже не нужно.

# Деплоймент юниты
Дак как же масштабировать подики
[frontend-dep.yaml](frontend-dep.yaml)
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
spec:
  selector:
    matchLabels:
      app: frontend
  replicas: 3
  minReadySeconds: 15
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 1
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: frontend
          image: mcdodik2008/k8s-mastery-frontend:latex
          imagePullPolicy: Always
          ports:
            - containerPort: 80
```
```bash
kubectl apply -f frontend-dep.yaml
```
Предположим нам нужно поменять что-то на фронтсе
Обновляем образ в доккер хабе 
Меняем версию образа в файлике
переапплаим деплоймент юнит
Желаетльео использовать флаг `--record` что бы он запомнил предыдущую версию
```bash
kubectl apply -f frontend-dep.yaml --record
```
Можно чекнуть историю 
```bash
kubectl rollout status deployment frontend
```
Откатить версию можно так 
```bash
kubectl rollout undo deployment frontend --to-revision=1
```

Раскатим все остальные депы ([logic-dep](logic-dep.yaml), [webapp-dep](webapp-dep.yaml))

лоадбалансер для webapp
```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-app-lb
spec:
  type: LoadBalancer
  ports:
    - port: 81
      protocol: TCP
      targetPort: 8080
  selector:
    app: web-app
```
и для логики
```yaml
apiVersion: v1
kind: Service
metadata:
  name: logic
spec:
  ports:
    - port: 80
      protocol: TCP
      targetPort: 5000
  selector:
    app: logic
```





