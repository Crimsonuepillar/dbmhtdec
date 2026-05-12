# Steam Trading Bot — ARCHITECTURE.md
> Вставляй этот файл в начало каждой новой сессии с ИИ вместо кода.

---

## Стек
- Python 3.11+, asyncio
- **aiosteampy** ≥ 0.7 — Steam API (login, inventory, listings, orders, history)
- **aiohttp** + **httpx** (для CSFloat)
- **SQLite** (через stdlib sqlite3) — локальный кеш
- **protobuf** ≥ 5.26 — CS2 asset properties
- **python-dotenv** — env-переменные

---

## Файловая структура
```
simple.py           — главный скрипт: CLI-меню, логин, sweep, все команды
cache.py            — SQLite-кеш: аккаунты, балансы, листинги, ордера, инвентарь, история
item_info.py        — просмотр предмета: histogram, price_history, ASCII-график, листинги с флоатами
patterns.py         — детектор редких паттернов CS2 (7patterns.txt / 7patterns.json)
steam_errors.py     — классификатор ошибок Steam → SteamError(category, short, fatal_for_batch)

accounts/<name>/
  account.json      — {label, username, password, steam_id}
  *.maFile          — Steam Desktop Authenticator file

data/
  cache.sqlite3     — SQLite база
  7patterns.txt     — base_name'ы скинов с редкими паттернами (через запятую)
  7patterns.json    — точные paint_seed по тирам [{base_name, tiers:[{note, patterns:[]}]}]

proxies.txt         — список прокси для sweep (по одному на строку, схема http/socks5)
.steam_session/     — кешированные cookies (username.cookies)
secrets/            — legacy: одиночный maFile
```

---

## Модули — публичный API

### cache.py
```python
open_db() -> sqlite3.Connection
record_account(username, label)
record_balance(username, balance_cents, on_hold_cents, currency_code)
record_listings(username, listings_iter, currency_code, partial=True) -> int
record_buy_orders(username, orders_iter, currency_code) -> int
record_history_events(username, events_iter, currency_code, price_extractor=None) -> int
record_inventory(username, app_context_name, items_iter, *, paint_seed_extractor, paint_wear_extractor, state_extractor, partial=False) -> int
get_last_refresh(username, resource) -> datetime | None
get_latest_balance(username) -> dict | None
get_listed_asset_ids(username) -> set[str]
find_listing_by_asset_id(username, asset_id) -> str | None
get_listing_by_asset_id(username, asset_id) -> dict | None
insert_placed_listing(username, listing_id, *, unowned_id, market_hash_name, price_cents, currency_code)
delete_listing(username, listing_id)
delete_buy_order(username, order_id)
mark_inventory_state_by_asset_id(username, asset_id, new_state) -> int
iter_inventory(username=None, app_context=None) -> list[dict]
iter_account_summaries() -> list[dict]
get_buy_orders_total(username) -> dict | None
get_known_event_ids(username) -> set[str]
get_cached_nameid(app_id, market_hash_name) -> int | None
cache_nameid(app_id, market_hash_name, item_nameid)
```

**SQLite схема (таблицы):**
- `accounts` (username PK, label, label_num, last_seen_at)
- `wallet_snapshots` (username, snapshot_at, balance_cents, on_hold_cents, currency_code)
- `listings_cache` (username+listing_id PK, asset_id, unowned_id, market_hash_name, price_cents, currency_code, time_created)
- `buy_orders_cache` (username+order_id PK, market_hash_name, price_cents, qty_remaining, qty_total)
- `market_history` (username+event_id PK, event_type, market_hash_name, time_event, price_cents) — append-only
- `inventory_cache` (username+app_context+asset_id PK, market_hash_name, amount, paint_seed, paint_wear, state, tradable_after, extra_json)
- `market_nameids` (app_id+market_hash_name PK, item_nameid)
- `refresh_log` (username+resource PK, last_refresh_at)

**inventory_cache.state** (4 значения):
- `"free"` — свободен, можно выставлять
- `"on_market"` — уже на листинге
- `"trade_protect"` — получен через трейд, 7-дн. защита (context=16)
- `"trade_hold"` — market-hold после покупки с ТП

---

### patterns.py
```python
load_pattern_db(force=False) -> dict   # {danger_zone: set[str], rare_patterns: dict}
is_rare_pattern(market_hash_name, paint_seed) -> RarePatternResult
  # .is_rare: True=редкий | False=не редкий | None=uncertain (в danger_zone но нет номеров)
  # .tier_note: "Tier 1" если редкий
  # .base_name: имя без (wear)
is_charm(market_hash_name) -> bool     # брелоки/чармы — все редкие, не выставляем массово
```

---

### steam_errors.py
```python
classify_steam_error(exc) -> SteamError
  # .category: max_wallet | rate_limited | item_unavailable | price_too_low | price_too_high
  #            need_mobile_confirm | session_expired | not_logged_in | network | unknown
  # .short: человекочитаемая строка (рус.)
  # .fatal_for_batch: True → прерывать bulk-цикл
  # .retryable: True → имеет смысл повторить
format_for_log(err, prefix="") -> str  # одно- или двустрочная строка для print()
```

---

### item_info.py
```python
# Главная точка входа:
await show_item_info_menu(client, market_hash_name, app, currency_enum, currency_code, ask=None, currency_sym="")

# Вспомогательные (используются и самостоятельно):
await resolve_item_nameid(client, app_id, market_hash_name) -> int | None  # кеш→Steam HTML
render_histogram_block(histogram, sym, max_rows=10) -> list[str]
render_price_chart_block(history, label, sym) -> list[str]   # label = "7d"|"30d"|"all"
render_sales_volume_block(history) -> list[str]
render_data_table(history, label, sym, limit=30) -> list[str]
render_full_stack_block(histogram, sym, side="sell"|"buy", limit=None) -> list[str]
render_listings_page(listings, sym, start_idx, total, floats=None) -> list[str]
```

---

## simple.py — ключевые функции

### Логин и мульти-аккаунт
```python
_discover_accounts() -> list[dict]     # сканирует accounts/<name>/account.json + *.maFile
_connect_account(account, force_relogin, proxy=None) -> (client, currency_code, cookies_file) | None
_try_resume(client, cookies_file) -> bool  # восстановление сессии из .cookies
_full_login(client, ...) -> bool
```

### CS2-экстракция
```python
_cs2_extract_wear_seed(item) -> (float|None, int|None)  # paint_wear, paint_seed
_cs2_extract_stickers(item) -> [(name, wear|None), ...]
_cs2_extract_charms(item)   -> [(name, pattern|None), ...]
# propertyid: 1=paint_seed, 2=wear, 4=sticker_wear
```

### Inventory state
```python
_inventory_state(item, listed_asset_ids, protected_asset_ids) -> str  # "free"|"on_market"|"trade_protect"|"trade_hold"
_is_trade_protected(item, protected_asset_ids) -> bool
# CS2_PROTECTED = AppContext context=16 (трейд-защита)
```

### Sweep (batch refresh всех аккаунтов)
```python
await _run_sweep(accounts, sessions, force_relogin)
await _sweep_one_account(account, sessions, force_relogin, fetch_history=True, proxy=None) -> dict
# Последовательно: balance → get_my_listings (все страницы) → buy_orders → inventories (4 контекста + CS2_PROTECTED) → history delta
# proxy: round-robin из proxies.txt (только для sweep; place_sell/cancel идут с main IP)
```

### Bulk-операции
```python
await _bulk_list_group(client, group, currency_enum, currency_code, ...)  # выставить всю группу одной ценой
await _bulk_cancel_listings(client, listings, ask_confirm=True) -> int     # снять список листингов
await _bulk_sell_cross_account(name, candidates, accounts_lookup, sessions, ...)   # выставить с разных акков
await _bulk_cancel_cross_account(name, listed_rows, accounts_lookup, sessions, ...) # снять с разных акков
```

### Пагинация
```python
await _paginate(items, page_size, render, extra_commands, bulk_commands)  # in-memory список
await _paginate_lazy(total, page_size, fetch_more, render, extra_commands)  # lazy-load по страницам
```

### Глобальная статистика
```python
await _show_global_stats(accounts, sessions, force_relogin)
await _show_cs2_subgroups(rows, ...)  # 4 группы: free/on_market/trade_protect/trade_hold
await _show_grouped_items(rows, title, ...) # пагинация grouped: i<N>/s<N>/c<N>
```

---

## Конфигурация
```python
# В шапке simple.py (или через .env / account.json):
STEAM_PASSWORD = "..."
MAFILE_PATH    = "secrets/xxx.maFile"
FORCE_RELOGIN  = False
INVENTORY_PAGE_SIZE  = 25
HISTORY_PAGE_SIZE    = 10
LISTINGS_PAGE_SIZE   = 10
BUY_ORDER_LIMIT_MULTIPLIER = 10  # Steam позволяет ордера до ~10× баланса

# Env-переменные:
STEAM_PASSWORD_<USERNAME>  # пароль конкретного акка
SWEEP_PROXY / SWEEP_PROXY_FILE  # прокси для sweep
```

---

## Важные нюансы
1. **aiosteampy 0.7** — monkey-patch `ItemDescription._set_d_id` (баг парсинга d_id)
2. **Троттлинг**: 0.3–0.4с между place_sell_listing, 0.5с между аккаунтами
3. **429 retry**: 3 попытки с задержками 3/8/20с (`_with_retry`)
4. **listed_asset_ids cross-ref**: Steam оставляет предмет в инвентаре после выставления — cross-ref по unowned_id/asset_id
5. **CSFloat API**: ~30 req/min, пауза 2с между запросами, httpx предпочтительнее aiohttp для TLS fingerprint
6. **History delta**: при sweep'е идём страницами до первого known event_id (максимум 20 страниц = 1000 событий)
7. **CS2_PROTECTED** (context=16): предметы после трейда, недоступны для продажи 7 дней

---

## Текущая задача
<!-- ЗАПОЛНЯЙ ЭТО ПОЛЕ КАЖДУЮ СЕССИЮ -->
_Опиши что сейчас делаешь: какой модуль правишь, какая фича нужна, что не работает_
