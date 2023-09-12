### Postgres & Docker
1. Устанавливаем Docker:
```
sudo apt install docker.io
```
2. Запускаем Docker и добавляем в автозапуск:
```
sudo systemctl start docker
sudo systemctl enable docker
```
3. Создадим директорию для данных Postgres:
```
mkdir -p $HOME/docker/volumes/postgres
```
4. Получение контейнера PostgreSQL:
```
sudo docker pull postgres
```
5. Запустим контейнер c Postgres-сервером:
```
sudo docker run --rm --name pgdocker -e POSTGRES_PASSWORD=user -e POSTGRES_USER=user -e POSTGRES_DB=test -d -p 5432:5432 -v $HOME/docker/volumes/postgres:/var/lib/postgresql/data postgres
```
6. Узнаем IP сервера БД:
```
docker inspect \
  -f '{{range.NetworkSettings.Networks}}{{.IPAddress}}{{end}}' pgdocker
```
6. Запустим контейнер с клиентом и подключим его к БД:
```
docker run -it --rm jbergknoff/postgresql-client postgresql://user:user@172.17.0.2:5432/test
```

7. Создадим таблицу и добавим в неё пару строк:
```
create table persons(id serial, first_name text, second_name text);
insert into persons(first_name, second_name) values('ivan', 'ivanov');
insert into persons(first_name, second_name) values('petr', 'petrov');
```

8. Установим PostgreSQL Client на сторонний ПК:
```
sudo apt install postgresql-client
```
9. Подключимся со стороннего ПК к контейнеру с сервером:
```
psql -U user -h <pgserverIP>  -d test
```
Подключение успешно выполнено. 

10. Удалим контейнер с сервером:
```
docker rm -f pgdocker
```
11. Создадим контейнер заново:
```
sudo docker run --rm --name pgdocker -e POSTGRES_PASSWORD=user -e POSTGRES_USER=user -e POSTGRES_DB=test -d -p 5432:5432 -v $HOME/docker/volumes/postgres:/var/lib/postgresql/data postgres
```

12. Снова подключимся из контейнера-клиента к контейнеру-серверу:
```
docker run -it --rm jbergknoff/postgresql-client postgresql://user:user@172.17.0.2:5432/test
```
13. Проверим, что данные остались на месте:
```
select * from persons;
```
Все данные остались на месте:
```
 id | first_name | second_name 
----+------------+-------------
  1 | ivan       | ivanov
  2 | petr       | petrov
(2 rows)
```