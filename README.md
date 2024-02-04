# Запуск Minikube с Jenkins и создание Job для Nginx

Репозиторий содержит файлы и инструкции для запуска Minikube с Jenkins и создания Job для развертывания Nginx в Kubernetes.

## Требования:

1. kubectl
2. гипервизор(я использовал docker)
3. Minikube
4. Helm v3   

## Шаги:

### 1. Запуск Minikube

```bash
minikube start
```
### 2. Установка Jenkins
```bash
# Установите Jenkins с использованием Helm
helm repo add jenkins https://charts.jenkins.io
helm repo update
helm install my-jenkins jenkins/jenkins
# для проверки установки используйте команду
kubectl get pods
# Status должен быть Running
```

### 3. настройки RBAC для jenkins
  Создайте файл rbac-admin.yaml с таким содержанием, либо используйте с репозитория 
  ```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
    name: jenkins-admin-binding
subjects:
- kind: ServiceAccount
    name: default
    namespace: default
roleRef:
    kind: ClusterRole
    name: admin
    apiGroup: rbac.authorization.k8s.io
```
После чего примените его
```bash
kubectl apply -f rbac-admin.yaml
```
### 4. Запуск Jenkins

1. Получите пароль от jenkins используя эту команду
```bash
kubectl exec --namespace default -it svc/my-jenkins -c jenkins -- /bin/cat /run/secrets/additional/chart-admin-password && echo
```
2. Для того чтобы зайти на него извне используйте данную команду
```bash
kubectl port-forward svc/my-jenkins 8080:8080
```
3. В своём браузере перейдите на http://127.0.0.1:8080

4. Используйте пароль из первого пункта и username: admin для того чтобы залогиниться

### 5. Создание Jenkins Job для Nginx

1. После того как вошли, нажимаем на кнопку Create a job.

    Вводим имя item'а, например nginx-deploy

    Выбираем pipeline и нажимаем OK

2. копируем содержимое jenkinsfile в pipeline script и нажимаем сохранить
```groovy
   pipeline {
    agent {
        kubernetes {
                 containerTemplate {
                   name 'helm'
                   image 'alpine/helm:3.14.0'
                   ttyEnabled true
                   command 'cat'
      }
     }
    }
    stages {
        stage('clone git') {
            steps {
                git 'https://github.com/Flashwidow/nginx-chart.git'
            }
        }
        stage('deploy chart') {
            steps {
                container('helm') {
                sh 'helm version'
                sh 'helm install my-nginx-0 ./nginx-charts'
                }
        }
    }
}
} 
```
3. Заходим в dashboard и запускаем job

После некоторого времени видим зеленую галочку, а это значит, что мы успешно развернули nginx на minikube

### 6. Проверка
Для проверки что мы установили используйте данную команду
```bash
helm list
#появился my-nginx-0
```
Для проверки что ngix доступен на порту 32080 на ноде k8s
```bash
kubectl get svc -o wide
```
Для того чтобы зайти на nginx используйте это
```bash
minikube service nginx-service --url
```
перейдите в браузер и вставьте что выдал терминал, вы должны попасть на приветственную страницу nginx 

Либо если вы используете другой драйвер(не docker), то можете использовать
```bash
minikube ip
```
И для перехода на приветственную страницу nginx в браузере введите этот ip с портом:32080

http://'MINIKUBE-IP':32080
