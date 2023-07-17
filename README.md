
# Мониторинг доставки событий в Sentry

Цель:
* убедиться что мы можем отправлять события в _Sentry_ (получаем `20X` код ответа при отправке)
* убедиться что события доставлены в конкретные `issue`


## Отправка фейковых событий

Отправка события на [store endpoint](https://develop.sentry.dev/sdk/store/):
```bash
PROJECT_ID=0;
HOST=''
SENTRY_KEY=''

curl -v \
    -d '{"exception":{"values":[{"type":"Test issue","value":"store"}]}}' \
    -H 'Content-type: application/json' \
    -H "X-Sentry-Auth: Sentry sentry_version=7, sentry_client=td-agent, sentry_key=$SENTRY_KEY" \
    "https://$HOST/api/$PROJECT_ID/store/"
```

Отправка события на [envelope endpoint](https://develop.sentry.dev/sdk/envelopes/):
```bash

PROJECT_ID=0;
HOST=''
SENTRY_KEY=''

# подготовка тела запроса
echo '{"sdk":{"name":"sentry.javascript.vue","version":"7.54.0"},"trace":{"environment":"monitoring"}}' > body.json
echo '{"type":"event"}' >> body.json
echo '{"exception":{"values":[{"type":"Test issue","value":"frontend"}]}}' >> body.json

curl -v \
  -H 'Content-type: text/plain' \
  --data-binary @body.json \
  "https://$HOST/api/$PROJECT_ID/envelope/?sentry_key=$SENTRY_KEY&sentry_version=7"
```

Отправка события через [tunnel](https://docs.sentry.io/platforms/javascript/troubleshooting/#dealing-with-ad-blockers) на `envelope endpoint`:
```bash
PROJECT_ID=0;
HOST=''
SENTRY_KEY=''

# подготовка тела запроса
# авторизация в теле прямо из клиента, нам она не нужна, но для правдоподобности оставим
echo '{"sdk":{"name":"sentry.javascript.vue","version":"7.54.0"},"trace":{"environment":"monitoring"},"dsn":"https://'$SENTRY_KEY'@'$HOST'/'$PROJECT_ID'"}' > body.json
echo '{"type":"event"}' >> body.json
echo '{"exception":{"values":[{"type":"Test issue","value":"frontend tunnel"}]}}' >> body.json

# мы отправляем туда же, куда обычно шлем запросы, потому что под капотом свой обработчик
curl -v \
  -H 'Content-type: text/plain' \
  --data-binary @body.json \
  "https://$HOST/api/$PROJECT_ID/envelope/$SENTRY_KEY/"
```

> Теперь у нас есть 3 разных `issue`, которые нужно мониторить на предмет давности доставки событий.


## Мониторинг доставляемости событий по проектам

Проверка сколько секунд назад было последнее событие в `issue` через [API](https://docs.sentry.io/api/events/retrieve-an-issue/) (нужно выполнить для каждого `issue` сгенерированных на прошлых шагах):
```bash
ISSUE_ID=0
SENTRY_TOKEN=""
HOST=''

lastSeen=$(curl -s -H "Authorization: Bearer $sentryToken" "https://$HOST/api/0/issues/$ISSUE_ID/" | grep -o -m 1 '"lastSeen":"[^"]*' | grep -o -m 1 '[^"]*$')
timeDiff=$(( $(date -d "$lastSeen" +"%s") - $(date +%s) ))
timeDiff=${timeDiff#-}

echo $timeDiff
```
