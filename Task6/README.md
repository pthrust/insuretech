# Настройка Rate Limiting

```bash
http {
    # ============================================================
    # Зона Rate Limiting
    # ============================================================
    # $binary_remote_addr — ключ по IP клиента (можно заменить на $http_x_api_key
    # для ограничения по API-ключу партнёра — что правильнее для B2B).
    # zone=partner_zone:10m — 10 МБ памяти (~160 тыс. IP-адресов).
    # rate=10r/m — 10 запросов в минуту.
    limit_req_zone $binary_remote_addr zone=partner_zone:10m rate=10r/m;

    # Код ответа при превышении лимита — 429 Too Many Requests
    limit_req_status 429;

    # Логирование отказов по лимиту в отдельный уровень
    limit_req_log_level warn;

    # ============================================================
    # Upstream для балансировки нагрузки
    # ============================================================
    upstream backend_servers {
        server backend1.example.com;
        server backend2.example.com;
        server backend3.example.com;
    }

    server {
        listen 80;

        location / {
            # Применяем ограничение:
            #   burst=10 — разрешаем очередь из 10 запросов сверх лимита;
            #   nodelay — обрабатываем их сразу, а не с задержкой.
            # Итог: при всплеске клиент может отправить 10 мгновенно,
            # далее — не более 10 запросов в минуту.
            limit_req zone=partner_zone burst=10 nodelay;

            proxy_pass http://backend_servers;
        }
    }
}
```