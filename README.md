# STALZONE Auction Monitor (RU)

Монитор аукциона STALZONE (бывш. STALCRAFT: X) с уведомлениями в Telegram.
**Ничего не покупает и не нажимает в игре** — только смотрит рынок и уведомляет.

## Источник данных
Официальный API EXBO, метод *Active Item Lots*:
`GET https://eapi.stalzone.com/ru/auction/{ITEM_ID}/lots`, заголовок `Authorization: Bearer <token>`.
Нужен токен вашего приложения, одобренного EXBO (https://eapi.stalcraft.net). Rate limit не обходится:
запросы разделены паузой `MIN_REQUEST_GAP`, на HTTP 429 программа ждёт `Retry-After`.

## Установка (Python 3.12+, Tkinter входит в стандартный установщик Python)
```
python -m venv .venv
.venv\Scripts\activate          # Linux/macOS: source .venv/bin/activate
pip install -r requirements.txt
copy .env.example .env          # Linux/macOS: cp .env.example .env
```
Заполните в `.env`: `STALZONE_API_TOKEN`, `TELEGRAM_BOT_TOKEN`, `TELEGRAM_CHAT_ID`.
Telegram: создайте бота у @BotFather, напишите ему `/start`, chat id узнайте у @userinfobot.

## Запуск
```
python -m app              # окно программы
python -m app --headless   # без окна, только консоль/логи
python -m unittest discover -s tests -t . -v   # тесты
```

## Как пользоваться
1. «+ Добавить предмет»: название (для уведомления), **ID предмета** (например `g43rp` — см. репозиторий
   EXBO-Studio/stalcraft-database), макс. цена за 1 шт., по желанию мин. количество и макс. цена лота.
2. Выберите интервал и нажмите «Запустить».

## Логика
`unit_price = buyoutPrice / amount` (Decimal). Уведомление, если предмет совпал по ID, `amount ≥ min_amount`,
`unit_price ≤ лимита`, (опц.) `buyoutPrice ≤ макс. цена лота`, и лот ещё не был отправлен.
Лоты без цены выкупа пропускаются (их нельзя купить сразу).

**Защита от повторов.** В API нет ID лота, поэтому лот идентифицируется отпечатком SHA-256 от неизменяемых
полей: `itemId, amount, startPrice, buyoutPrice, startTime, endTime, additional` (`currentPrice` исключён —
он меняется при ставках). Отпечатки хранятся в SQLite (`detected_lots`); если Telegram недоступен, лот
не помечается отправленным и повторится в следующем цикле. Новый лот с другим временем создания = новое уведомление.

## Архитектура
`app/auction` (client, parser, models, pricing) · `app/monitoring` (monitor, filters, deduplication, runner) ·
`app/notifications` (base, telegram, console — добавьте Discord/email, унаследовав `Notifier`) ·
`app/database` (SQLite, stdlib) · `app/ui` (Tkinter). SQLite выбран как надёжное хранилище без внешних
зависимостей; Tkinter — чтобы стек был минимальным (httpx — единственная зависимость).

## Ограничения
- API отдаёт лоты по одному предмету → один запрос на уникальный ID за цикл (не один на весь рынок).
  При большом списке реальный цикл может быть длиннее выбранного интервала (показывается в окне).
- За запрос — до 200 лотов; `MAX_PAGES` задаёт число страниц (параметр `offset`). Если лотов больше,
  в логе будет предупреждение. Сортировку (`LOTS_SORT`/`LOTS_ORDER`) включайте только после проверки
  значений в Swagger.
- Цены считаются в тех единицах, в которых их отдаёт API.
