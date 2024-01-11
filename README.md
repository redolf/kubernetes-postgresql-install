Установка Postgresql в кластере kubernetes
=========

Создание Persistent Volume для баз данных
--------------
> В данном случае будет рассматриваться создание Persistent Volume на домашнем kubernetes кластере используя локальный каталог одной из нод

- Создайте на одной из worker-нод кластера каталог для Persistent Volume:

  ```
  mkdir -p /devkube/postgresql
  ```
- Создайте StorageClass для Persistent Volume с помощью манифеста sc.yaml:

  ```
  kubectl apply -f sc.yaml
  ```
- Отредактируйте манифест pv.yaml под себя и создайте с помощью него Persistent Volume: 

  ```
  kubectl apply -f pv.yaml
  ```
- Отредактируйте манифест pvc.yaml под себя и создайте с помощью него запрос (Persistent Volume Claim) на получение Persistent Volume:

  ```
  kubectl apply -f pvc.yaml
  ```
- Убедитесь что Persistent Volume создан:

  ```
  kubectl get pv
  ```
- Убедитесь что Persistent Volume Claim создан и имеет статус Bound с привязкой к нашему созданному Volume:

  ```
  kubectl get pvc
  ```
Установка Postgresql и создание базы данных
--------------
- Добавим в список репозиториев репозиторий с чартами bitnami:

  ```
  helm repo add bitnami https://charts.bitnami.com/bitnami && helm repo update
  ```
- Установим чарт Postgresql:
  > **primary.persistence.existingClaim=pvc-postgresql** (Указываем наш Persistent Volume Claim);
  > **volumePermissions.enabled=true** (Включаем назначение необходимых прав на каталог с базами данных); 
  > **auth.enablePostgresUser=true** (Включаем создание пользователя по умолчанию).

  ```
  helm --install postgresql bitnami/postgresql --set primary.persistence.existingClaim=pvc-postgresql --set volumePermissions.enabled=true --set auth.enablePostgresUser=true
  ```
- Получим и запишем пароль пользователя postgres в переменную POSTGRES_PASSWORD:

  ```
  export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgresql -o jsonpath="{.data.postgres-password}" | base64 -d)
  ```
- Перенаправим порт сервиса Postgresql на локальную машину:

  ```
  kubectl port-forward --namespace default svc/postgresql 5432:5432
  ```
- Подключимся к СУБД Postgresql:
  > На машине с которой подключаетесь должен быть установлен Client Postgresql

  ```
  PGPASSWORD="$POSTGRES_PASSWORD" psql --host 127.0.0.1 -U postgres -d postgres -p 5432
  ```
- Создадим роль (пользователя) для нужного нам сервиса:

  ```
  CREATE ROLE имя_пользователя WITH LOGIN ENCRYPTED PASSWORD 'пароль';
  ```
- Создадим базу данных для нужного нам сервиса и назначим владельца базы данных:

  ```
  CREATE DATABASE имя_базы_данных OWNER имя_пользователя;
  ```
- Выйдем из Postgresql:

  ```
  \q
  ```