Интеграция GitLab и Sonarqube проверена только по http, без https (нужно подложить самоподписанный ca.crt в хранилище для Sonarqube)
Создаем Ingress для http-соединений с Gitlab
```text
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: gitlab-http-api
  namespace: gitlab
spec:
  ingressClassName: system-ingress
  rules:
    - host: api.gitlab.systemoteh.ru
      http:
        paths:
          - pathType: Prefix
            path: /
            backend:
              service:
                name: gitlab-webservice-default
                port:
                  number: 8181
EOF
```
Добавляем запись в /etc/hosts в файл хоста, где запущен Docker Compose с Sonarqube
```text
192.168.1.30 api.gitlab.systemoteh.ru
```
Для запуска Sonarqube в Docker Compose используем файл - [docker-compose.yml](docker-compose.yml)
После запуска Sonarqube переходим по ссылке http://localhost:9000/
Логин admin, пароль admin. Меняем старый пароль на новый password.
Переходим на вкладку Administration -> ALM Integration -> Gitlab -> Create configuration:
Configuration name: api.gitlab.systemoteh.ru
GitLab API URL: http://api.gitlab.systemoteh.ru/api/v4
Personal Access token: glpat-SHYnGptvDdwbDLexaXmo (актуальный токен из Gitlab: User Settings -> Access Tokens)

