# Манифест проверки ADR

## Проверка ADR завершена

- Дата: 2026-07-26
- Проверяющий: qwen-code (шаг `adr` schema `spec-driven-with-adr`)
- Изменение: `add-articles-metrics`

## Проверен контекст действующих ADR

- Нет: существующих ADR на уровне репозитория не было. В корне репозитория (`<repo>/adr/`) папка `adr/` отсутствует — проверка шаблоном `**/adr/**/*.md` от корня `sdd-store` (`/Users/danilbelokurov/Desktop/sdd-store.worktrees/TEST-9999`) возвращает только два нерелевантных совпадения: настоящий манифест `openspec/changes/add-articles-metrics/adr.md` и шаблон схемы `openspec/schemas/spec-driven-with-adr/templates/adr.md`. Цепочка `Supersedes` пуста, наивысший занятый номер последовательности не установлен — следующий доступный для новых ADR — `0001`. Ограничений со стороны ранее принятых архитектурных решений не выявлено.

## Созданные ADR на уровне репозитория

- Нет: это изменение не внесло значимых долгосрочных архитектурных решений. Все решения, принятые в `design.md` для `article-service` (`kotlin-spring-realworld-example-app`), носят тактический локальный характер и укладываются в уже зафиксированные правила `openspec/context/04-engineering-standards.md`. В частности:
  - новый read-only маршрут `GET /api/articles/metrics/count` — аддитивное расширение существующего envelope под префиксом `/api` (§5.1);
  - новые классы `MetricsHandler` (`io.realworld.web`) и `MetricsService` (`io.realworld.service`) размещены в установленных слоях с constructor-injection, без выхода за их границы (§3, §3.6);
  - аутентификация использует существующий `@ApiKeySecured(mandatory = false)` в режиме, уже применённом для публичного листинга статей (`GET /api/articles`), — никакого нового механизма auth не вводится (§6);
  - запрос к БД — derived-метод `countByCreatedAtBetween` на существующем `ArticleRepository` (уже расширяющем `JpaSpecificationExecutor`); альтернативы Specification и `@Query` отклонены обоснованно — без введения новых паттернов хранения (§4.2, §7);
  - валидация и обработка ошибок проходят через существующие `InvalidRequest` / `InvalidRequestHandler` с HTTP 422 и формой `{"errors":{...}}` (§5.3, §5.4, §8);
  - `OffsetDateTime` сохраняется и сравнивается «как есть» в UTC — следуя §4.3 без ввода нового time-моделирования;
  - `MetricsHandler` и `MetricsService` объявлены `class` без `final`, оставаясь совместимыми с all-open для Spring-прокси — соответствие §4.1.
  Ни одно из решений не устанавливает долгосрочного архитектурного обязательства (паттерна, технологии, границы, контракта), не затрагивает иные сервисы, перечисленные в `openspec/context/02-services.md` и `openspec/context/08-services.md`, и не расходится с какими-либо действующими инженерными стандартами. Открытые вопросы, зафиксированные в `design.md` (индекс по `Article.createdAt`, переход аутентификации метрик на `mandatory = true`, возможный cross-service контракт метрик для `service1` / `spring-school`), вынесены в отдельные будущие изменения и не требуют фиксации в форме ADR на этом шаге. Оснований заводить новые файлы под `<repo>/adr/` для `add-articles-metrics` нет.