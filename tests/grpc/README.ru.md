# gRPC smoke-тесты

**[English version](README.md)**

Smoke-проверки без coordinator: поведение агента напрямую через `grpcurl`.

## Команды

Из корня репозитория:

```bash
task run:test
```

Запуск агента с локальным тестовым конфигом и сертификатами.

```bash
task test:smoke
```

Нужен уже запущенный тестовый агент (`task run:test` в другом терминале). Три проверки `grpcurl`; опционально отчёты в `tests/grpc/reports/` (в gitignore): `runchecks-valid.out`, `runchecks-invalid-token.out`, `runchecks-invalid-payload.out`.

## Живой запрос (real flow)

Один запрос, сгенерированный логикой coordinator: дамп делается в [mytonprovider-backend](https://github.com/mytonprovider/mytonprovider-backend), JSON копируется сюда как `tests/grpc/runchecks-live.json` (и `runchecks-live-meta.json` рядом, если нужна гео в сводке).

```bash
task run:test
```

В другом терминале:

```bash
task test:live:replay
```

JSON уходит в `RunChecks` через `grpcurl`.

Дамп делает инструмент координатора `dump-runchecks` (in-memory или из Postgres); как его запускать — в [mytonprovider-backend](https://github.com/mytonprovider/mytonprovider-backend).

## RunStorageRates (provider GetStorageRates)

Агент слушает UDP **ADNL** для DHT и transport (порт по умолчанию `16167`, `AGENT_ADNL_PORT`; в тестовом task — `36167`, чтобы не конфликтовать с локальным coordinator).

### Replay (как RunChecks)

`dump-runchecks` умеет писать и **`runchecks-live-storage-rates.json`** (pubkeys из RunChecks payload, тот же порядок). Скопировать его в `tests/grpc/` вместе с `runchecks-live.json`, затем при запущенном агенте (`task run:test`) — **`task test:live:storage-rates:replay`**; ответ в `tests/grpc/reports/runchecks-storage-rates-last.json`.

Пример `grpcurl` (замените `PROVIDER_PUBKEY_HEX64` на 64 hex символа):

```bash
grpcurl -insecure \
  -import-path contracts/proto \
  -proto providerchecks/v1/provider_checks.proto \
  -H "authorization: Bearer test-token" \
  -d '{"jobId":"rates-smoke-1","providerPubkeys":["PROVIDER_PUBKEY_HEX64"],"querySize":1,"timeouts":{"queryTimeoutMs":14000,"totalMs":60000}}' \
  127.0.0.1:8443 \
  providerchecks.v1.ProviderChecksService/RunStorageRates
```

## Требования

- `grpcurl` в `PATH`
- `openssl`
- для live dump: исходящая сеть до TON lite / config URL
- для `--storage=postgres`: Postgres со схемой coordinator
