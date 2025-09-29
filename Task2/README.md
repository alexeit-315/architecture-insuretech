# Спринт 8. Создание highload в realtime-среде

## Задание 2. Динамическое масштабирование контейнеров
### 1. Подготовка окружения
Поднят локальный кластер Kubernetes в Minikube, активирован metrics-server
```bash
minikube start --memory=3000 --cpus=2
minikube addons enable metrics-server
```
Проверен статус minikube и сбор метрик
```bash
minikube status
kubectl get nodes
kubectl top nodes
kubectl top pods
```
### 2. Создание манифестов и запуск приложений
Создан манифест Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: scaletest-deployment
  labels:
    app: scaletest
spec:
  replicas: 1
  selector:
    matchLabels:
      app: scaletest
  template:
    metadata:
      labels:
        app: scaletest
    spec:
      containers:
        - name: scaletest-container
          image: ghcr.io/yandex-practicum/scaletestapp:latest
          ports:
            - containerPort: 8080
          resources:
            limits:
              memory: "30Mi"
            requests:
              memory: "20Mi"
```
Создан манифест Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: scaletest-service
spec:
  selector:
    app: scaletest
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: NodePort
```
Создан манифест HPA
```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: scaletest-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: scaletest-deployment
  minReplicas: 1
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: memory
        target:
          type: Utilization
          averageUtilization: 80
```
Создано приложение locustfile.py
```python
from locust import HttpUser, between, task

class WebsiteUser(HttpUser):
    wait_time = between(1, 5)

    @task
    def index(self):
        self.client.get("/")
```
Запущены приложения
```bash
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f hpa.yaml
```
Проверен запуск hpa и получен URL для доступа
```bash
minikube service scaletest-service --url
kubectl get hpa
```
### 3. Тестирование нагрузки с Locust
Установлен и запущен locust
```bash
pip install locust
locust
```
Запущено тестирование с использованием locust а адрес URL для доступа (user = 1500, hatch rate = 30)  
Прверяем масштабирование
```bash
minikube dashboard
kubectl get hpa -w
kubectl get pods -w
```
Результаты в виде принтскринов сохранены в рабочий каталог.
