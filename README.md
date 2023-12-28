## Комментарий разработчика

Структура и логика приложения разрабатывалась на основании тезисов ТЗ таким образом, чтобы все требования формально были выполнены в рамках одного цикла разработки.

По этой причине код специально в некоторых местах не усложнялся, а именно:

- реализован сидинг на базе миграций
- отсутствует транзакционность методов
- не обеспечена целостность потоков данных
- простое решение проблемы отсутствующего порога температур
- отсутствуют тесты
- некоторая логика намеренно не декомпозировалась

### Отдельно про пункт о консистентности потоков

Кейс "10 превышений подряд создают событие" реализован через хранение счетчика превышений в памяти процесса. При перезагрузке сервера:

1. Временно невозможно получать статистические данные
2. Сбрасывается счетчик

Для решения этой проблемы необходимо перенаправить поток статистических данных в шину данных и реализовать подключение этого приложения к ней. В качестве шины можно использовать:

- kafka
- rabbitmq

Kafka дает гарантию, что каждое сообщение **будет прочитано 1 раз**

RabbitMQ реализцет механизм **возврата изъятых сообщений** при возникновении исключений

В контексте этой задачи RabbitMQ, на мой взгляд, выглядит более предпочтительным


### Отдельно про решение проблемы отсутствующего порога температур

Поскольку источник значение порога находится вне приложения, то его природа становится динамической. Отсюда появляется необходимость хранить порог в базе и синхронизировать состояния рантайма и базы. Может быть такая ситуация, что в базе нет этого значения. Чтобы избежать зависимости от внешних данных и хардкода значения, сделал так, что приложение не запустится без этого значения

В нормальном сценарии необходимо реализовать конфиг или постоянный источник состояний, которому можно доверять


## Инструкция по запуску

1. Зависимости

```json
npm install
```

2. Окружение

```json
cp .env.example .env
docker-compose up -d
```

3. Структура данных + ини

```json
npm run migration:run
```

4. Запуск

```json
npm run start:dev
```

## Техническое задание

### Разработать сервис для обнаружения превышения средних значений температуры

**Интерфейс API:**

1. `POST /measurements`:

```json
{
	temperature: number,
	timestamp: Date
}
```

- Время (timestamp) всегда приходит последовательно, т.е. timestamp не может быть
  меньше, чем в предыдущем запросе.

- Значения нужно сохранять в базу данных.

2. `POST /thresholds`:

```json
{
	type: “temperature”,
	value: number
}
```

- Порог критической температуры. Нужно сохранять в базу данных.

- При превышении порога 10 раз подряд нужно записывать в базу данных событие о
  превышении порога с указанием времени последнего превышения.

- Если пришло 9 значений выше порога, затем нормальное значение – ничего делать не
  нужно.

3. `GET /events?start=1703496282766&end=1703496282799`, где start и end – время UTC в миллисекундах. Start и end – необязательные параметры.

- Возвращается список событий, отсортированных по времени создания (сначала новые),
  начиная от времени start, и заканчивая временем end.

**Дополнительные условия:**

- Не учитывать возможность горизонтального масштабирования, но обеспечить минимальное
  количество запросов к БД.

- Использовать NestJS, Typescript и PostgreSQL.

- Предоставить ссылку на репозиторий с кодом и инструкцию по запуску сервиса.