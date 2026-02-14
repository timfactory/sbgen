# 🧩 sbgen — генератор конфигураций sing-box из YAML-шаблонов

**Автор:** Ivan Tarasov  
**Год:** 2025  
**Лицензия:** MIT  

---

## 📘 Общее описание

`sbgen` — это утилита для генерации JSON-конфигураций [sing-box](https://sing-box.sagernet.org/) из шаблонов (`.tpl`) и YAML-профилей.

Она позволяет:
- описывать маршруты, правила и списки доменов в YAML;
- подключать внешние списки доменов (`includes`);
- объединять несколько профилей (например, `russia.yml`, `china.yml`);
- управлять приоритетами и fallback-логикой в шаблоне;
- гарантировать корректность итогового JSON.

---

## ⚙️ Использование

```bash
sbgen <template.tpl> <config.yml> [more.yml...] [опции]
```

**Опции:**
- `-v, --verbose` — Включить отладочный вывод в stderr
- `-b, --base-dir DIR` — Базовая директория для разрешения относительных путей `includes` (используется после базовых директорий item/tid)
- `-c, --cache-dir DIR` — Директория кэша для загружаемых по URL списков (по умолчанию: `~/.cache/sbgen`)
- `-r, --refresh` — Принудительно перезагружать списки по URL (при ошибке используется старый кэш)
- `-a, --append RULE` — Добавить сырой фрагмент (строка или JSON) в `route.rules[]` или `routing.rules[]` перед обработкой плейсхолдеров
- `-x, --xray` — Генерировать конфигурацию в стиле Xray/V2Ray вместо sing-box

**Примеры:**
```bash
# Базовое использование
./sbgen nekobox-russia.tpl profiles.yml > nekobox-russia.json

# С несколькими YAML файлами
./sbgen standalone-world.tpl profiles.yml local.yml > config.json

# Режим отладки
./sbgen nekobox-russia.tpl profiles.yml -v > config.json

# Режим Xray
./sbgen template.tpl config.yml -x > xray-config.json

# Добавление пользовательского правила
./sbgen template.tpl config.yml -a '{"domain":["example.com"],"outbound":"proxy"}' > config.json
```

Если один из указанных файлов не существует — утилита завершит работу с ошибкой.

---

## 🧱 Структура YAML

Каждый YAML-файл описывает один или несколько **профилей** (например, `russia`, `china`, `office`).

Пример:

```yaml
russia:
  base_dir: ./lists
  default_direct: true
  lists:
    - name: world
      out: [ proxy, proxy2 ]
      patterns:
        - "google.com"
        - "youtube.com"
        - "linkedin.com"
      includes:
        - "world.list"
      urls: [ "https://core.telegram.org/resources/cidr.txt" ]
  direct:
    - "myserver.local"
    - includes: ["direct.list"]
  blocked:
    - includes: ["block.list"]
```

---

## 📄 Параметры профиля

| Параметр | Тип | Описание |
|-----------|------|----------|
| `base_dir` | `string` | Опционально. Базовая директория для относительных путей `includes`. |
| `default_direct` | `bool` | Если `true`, `%%FINAL%%` становится `direct`. Если `false`, используется первый доступный `out` из `lists`. По умолчанию: `true`. |
| `lists` | `list<object>` | Набор маршрутизируемых групп доменов с указанием outbound'ов. |
| `direct` | `list<mixed>` | Домены и/или CIDR, направляемые напрямую (`outbound = direct`). |
| `blocked` / `block` | `list<mixed>` | Домены и/или CIDR, блокируемые (`outbound = block`). Поддерживаются оба ключа. |

---

### 🔸 Элементы `lists`

Каждый элемент может включать:

| Поле | Тип | Описание |
|-------|------|----------|
| `name` | `string` | Необязательное имя списка. |
| `out` | `string` или `list<string>` | Один или несколько `outbound`, куда направлять трафик. |
| `patterns` | `list<string>` | Список шаблонов доменов и/или CIDR (IPv4/IPv6 с префиксом). |
| `includes` | `list<string>` | Пути к внешним файлам (по одному домену на строку, `#` — комментарий). |
| `urls` | `list<string>` | URL'ы списков в том же формате, что и `includes`; загружаются по HTTP(S) с кэшем (по умолчанию `~/.cache/sbgen`; `-c`/`--cache-dir`, `-r`/`--refresh`; при ошибке загрузки используется старый кэш). |

---

## 🧩 Поддерживаемые шаблоны доменов

| Пример | Интерпретация | Сгенерированное правило |
|--------|----------------|-------------------------|
| `example.com` | `domain_suffix` | `"domain_suffix": ["example.com"]` |
| `.example.com` | `domain_suffix` | `"domain_suffix": ["example.com"]` |
| `*.example.com` | `domain_suffix` | `"domain_suffix": ["example.com"]` |
| `rutracker.*` | `domain_regex` | `"domain_regex": ["^(.+\\.)?rutracker\\.[^.]+$"]` |
| `cdn.*.example.com` | `domain_regex` | `"domain_regex": ["^cdn\\.[^.]+\\.example\\.com$"]` |
| `*cdn.com` | `domain_regex` | `"domain_regex": ["^[^.]*cdn\\.com$"]` |
| `localhost` | `domain_suffix` | `"domain_suffix": ["localhost"]` |

**CIDR.** Строки в формате CIDR (IPv4 или IPv6 с префиксом) распознаются и выводятся в **отдельных** правилах:

| Пример | Интерпретация | Сгенерированное правило |
|--------|----------------|-------------------------|
| `192.168.0.0/16` | правило по IP (CIDR) | `"ip_cidr": ["192.168.0.0/16"]` |
| `10.0.0.0/8`, `fd00::/8` | правило по IP (CIDR) | `"ip_cidr": ["10.0.0.0/8", "fd00::/8"]` |

В режиме Xray (`-x`) для CIDR используется поле `ip` вместо `ip_cidr`.

**Примечание:** В режиме Xray (`-x`) доменные поля становятся `domainSuffix` и `domainRegex` соответственно.

---

## 📂 Формат файлов `includes` и контента по `urls`

Содержимое файлов `includes` и загружаемых по полю `urls` списков имеет один и тот же формат. Пример `world.list`:

```text
# streaming
youtube.com
netflix.com
*.hulu.com

# social
twitter.com
facebook.com
```

- Каждая строка — домен или шаблон. Строка может быть также CIDR (например `192.168.0.0/16`, `fd00::/8`); такие записи попадают в правила с `ip_cidr` (sing-box) или `ip` (Xray).  
- `#` — комментарий (полная строка или inline с `\#` для экранирования).  
- Пустые строки игнорируются.  
- Поддерживаются inline комментарии: `example.com # комментарий`  
- Пути разрешаются относительно (в порядке приоритета):
  1. Директории YAML файла, из которого загружен элемент
  2. `base_dir` профиля (из YAML)
  3. Опции CLI `--base-dir`
  4. Текущей рабочей директории

---

## 🧠 Логика работы sbgen

1. Загружает `.tpl` шаблон и все YAML-файлы.  
2. Санитизирует JSON (удаляет trailing commas, заключает незакавыченные плейсхолдеры в кавычки).  
3. Объединяет все верхнеуровневые ключи (`tid`), например, `russia`, `china`.  
4. Собирает доступные теги outbound'ов из шаблона.  
5. Ищет плейсхолдеры вида `%%russia:in-tun%%` или `"%%russia:in-tun%%"` в шаблоне.  
6. Для каждого плейсхолдера:
   - Загружает паттерны из секций `lists`, `direct` и `blocked`
   - Разрешает пути `includes` (относительно `base_dir`, базовой директории элемента или CLI `-b`)
   - Разделяет паттерны на доменные (`domain_suffix`/`domain_regex`) и CIDR; для CIDR генерирует отдельные правила с `ip_cidr` (или `ip` в Xray)
   - Генерирует правила маршрутизации для указанного inbound
7. Заменяет плейсхолдеры на сгенерированные массивы правил.  
8. Вычисляет `%%FINAL%%` на основе `default_direct` и доступных outbound'ов.  
9. Выводит валидный JSON в stdout.  

---

## 🧩 Пример 1 — Standalone сервер

**Шаблон (`server.tpl`):**
```json
{
  "route": {
    "rules": [
      %%russia:in-tun%%
    ],
    "final": "%%FINAL%%"
  }
}
```

**YAML (`russia.yml`):**
```yaml
russia:
  lists:
    - out: proxy
      patterns: [ "youtube.com", "twitter.com" ]
  direct: [ "intranet.local" ]
```

**Результат:**
```json
{
  "route": {
    "rules": [
      { "inbound": "in-tun", "outbound": "proxy", "domain_suffix": ["twitter.com", "youtube.com"] },
      { "inbound": "in-tun", "outbound": "direct", "domain_suffix": ["intranet.local"] }
    ],
    "final": "direct"
  }
}
```

---

## 🌐 Пример 2 — NekoBox (режим TUN)

**Шаблон (`nekobox.tpl`):**
```json
{
  "inbounds": [
    {
      "tag": "in-tun",
      "type": "tun",
      "endpoint_independent_nat": true,
      "inet4_address": ["172.19.0.1/28"],
      "mtu": 9000,
      "sniff": true,
      "stack": "mixed"
    }
  ],
  "route": {
    "rules": [ %%russia:in-tun%% ],
    "final": "%%FINAL%%"
  }
}
```

---

## 🌐 Пример 3 — Standalone V2Ray/Xray

**Шаблон (`v2ray.tpl`):**
```json
{
  "log": { "level": "warn" },
  "inbounds": [
    {
      "tag": "in-socks",
      "port": 1080,
      "protocol": "socks",
      "settings": {
        "auth": "noauth"
      }
    }
  ],
  "outbounds": [
    { "tag": "proxy", "protocol": "vmess", ... },
    { "tag": "direct", "protocol": "freedom" },
    { "tag": "block", "protocol": "blackhole" }
  ],
  "routing": {
    "rules": [
      %%myprofile:in-socks%%
    ],
    "domainStrategy": "IPIfNonMatch"
  }
}
```

**Генерация конфигурации Xray/V2Ray:**
```bash
./sbgen v2ray.tpl config.yml -x > v2ray-config.json
```

**Примечание:** В режиме Xray (`-x`):
- Используется `routing` вместо `route`
- Используется `routing.rules[]` вместо `route.rules[]`
- Имена полей в camelCase: `outboundTag`, `domainSuffix`, `domainRegex`, `inboundTag`
- `%%FINAL%%` заменяется на выбранный тег outbound (например, `"direct"` или `"proxy"`) и может использоваться в `routing.final` или в любом месте шаблона

---

## 🌍 Пример 4 — Несколько YAML профилей

```bash
./sbgen standalone-world.tpl profiles.yml > config.json
```

- Плейсхолдеры `%%russia:in-tun%%` и `%%world:in-tun%%` будут разрешены, если присутствуют в шаблоне.  
- YAML-файлы объединяются по верхнеуровневым идентификаторам (`tid`).  
- Если один и тот же `tid` встречается в нескольких файлах, их секции объединяются (lists, direct, blocked комбинируются).

---

## 🔧 Пример 5 — Использование опции `-a` (--append)

Опция `-a` позволяет добавить пользовательские правила в `route.rules[]` (или `routing.rules[]` в режиме Xray) **перед** обработкой плейсхолдеров. Это полезно для добавления приоритетных правил или правил, которые не вписываются в структуру YAML.

### Пример 5.1 — Добавление одного правила (плейсхолдер-строка)

Добавить плейсхолдер, который будет обработан позже:

```bash
./sbgen nekobox-russia.tpl profiles.yml -a '%%russia:in-extra%%' > config.json
```

Это добавит `%%russia:in-extra%%` в массив правил, который будет заменен во время обработки плейсхолдеров, если он существует в контексте шаблона.

### Пример 5.2 — Добавление одного правила (JSON объект)

Добавить пользовательское правило напрямую:

```bash
./sbgen standalone-russia.tpl profiles.yml \
  -a '{"domain":["example.com","test.com"],"outbound":"direct"}' \
  > config.json
```

**Результат:** Правило добавляется в конец `route.rules[]`. Если плейсхолдеры обрабатываются, они заменяются на месте, поэтому порядок зависит от расположения плейсхолдеров в шаблоне.

---

## 📁 Пример структуры проекта

```
.
├── sbgen                    # Python скрипт
├── profiles.yml            # YAML конфигурация с несколькими профилями
├── nekobox-russia.tpl      # Шаблон для NekoBox (мобильный)
├── standalone-russia.tpl   # Шаблон для standalone (десктоп)
├── lists/                  # Директория со списками доменов
│   ├── world.list          # Международные сервисы
│   ├── russia.list         # Российские сервисы
│   ├── israel.list         # Израильские сервисы
│   ├── direct.list         # Домены для прямого подключения
│   └── block.list          # Заблокированные домены
└── README_RU.md
```

**Типичный рабочий процесс:**
```bash
# Генерация конфигурации NekoBox
./sbgen nekobox-template.tpl config.yml > nekobox-config.json

# Генерация конфигурации standalone
./sbgen standalone-template.tpl config.yml > standalone-config.json
```

---

## 📱 Пример работы с NekoBox

Этот раздел показывает, как модифицировать существующую конфигурацию NekoBox с помощью `sbgen`.

**Примечание:** По умолчанию в NekoBox используются:
- Тег inbound: `tun-in`
- Тег outbound: `proxy`

При создании шаблонов или использовании плейсхолдеров используйте эти стандартные имена (например, `%%myprofile:tun-in%%`).

**Пример YAML конфигурации (`config.yml`):**

```yaml
myprofile:
  base_dir: ./lists
  default_direct: true
  lists:
    - name: world
      out: [ proxy ]
      includes: [ world.list ]
      urls: [ "https://core.telegram.org/resources/cidr.txt" ]
      patterns:
        - "google.com"
        - "youtube.com"
  direct:
    - includes: [ direct.list ]
  block:
    - includes: [ block.list ]
```

Эта конфигурация сгенерирует правила для плейсхолдера `%%myprofile:tun-in%%`, направляя домены из `world.list` через outbound `proxy`.

### Шаг 1: Экспорт конфигурации из NekoBox

1. Откройте приложение NekoBox
2. Выберите конфигурацию вашего сервера
3. Нажмите иконку **Share** (📤)
4. Выберите **Configuration** → **Export as file**
5. Сохраните экспортированный JSON файл (например, `exported-config.json`)

### Шаг 2: Обработка конфигурации

У вас есть два варианта для добавления правил маршрутизации:

#### Вариант A: Создать шаблон

1. **Извлеките базовую конфигурацию:**
   - Скопируйте экспортированный JSON
   - Удалите или замените секцию `route.rules[]` на плейсхолдеры
   - Сохраните как шаблон (например, `custom-template.tpl`)

   **Пример шаблона:**
   ```json
   {
     "log": { "level": "warn" },
     "dns": { ... },
     "inbounds": [ ... ],
     "outbounds": [ ... ],
     "route": {
       "rules": [
         %%myprofile:tun-in%%,
         { "inbound": ["tun-in"], "outbound": "proxy" }
       ],
       "final": "%%FINAL%%"
     }
   }
   ```

2. **Сгенерируйте новую конфигурацию:**
   ```bash
   ./sbgen custom-template.tpl config.yml > updated-config.json
   ```

#### Вариант B: Использовать `-a` для добавления правил

Добавьте правила напрямую в экспортированную конфигурацию. Можно передать плейсхолдер или JSON правило через `-a`:

```bash
# Добавить плейсхолдер, который будет обработан
./sbgen exported-config.json config.yml \
  -a '%%myprofile:tun-in%%' \
  > updated-config.json

# Или добавить JSON правило напрямую
./sbgen exported-config.json config.yml \
  -a '{"domain_suffix":["example.com"],"outbound":"direct"}' \
  > updated-config.json
```

**Примечание:** При использовании `-a` с экспортированной конфигурацией скрипт:
- Обработает любые плейсхолдеры в файле (если присутствуют)
- Обработает плейсхолдеры, переданные через `-a` (если есть)
- Добавит ваше пользовательское правило в `route.rules[]`
- Сохранит все остальные настройки (DNS, inbounds, outbounds и т.д.)

### Шаг 3: Импорт обратно в NekoBox

1. Откройте приложение NekoBox
2. Нажмите иконку **Add** (+)
3. Выберите **Manual Settings** → **Custom Config**
4. Выберите **Import from file** или вставьте содержимое JSON
5. Выберите сгенерированный файл (например, `updated-config.json`)
6. Сохраните и активируйте конфигурацию

### Полный пример

```bash
# 1. Экспорт из NekoBox → exported-config.json

# 2. Обработка с sbgen (используя опцию -a с плейсхолдером)
./sbgen exported-config.json config.yml \
  -a '%%myprofile:tun-in%%' \
  > updated-config.json

# 3. Импорт updated-config.json обратно в NekoBox
```

**Совет:** Используйте флаг `-v` для просмотра добавляемых правил:
```bash
./sbgen exported-config.json config.yml -a '...' -v > updated-config.json
```

---

## 🧩 Отладка

Для включения отладочного вывода:
```bash
./sbgen template.tpl config.yml -v
```

Выводит в stderr:
- Пути к шаблону и YAML файлам
- Базовые директории для каждого профиля
- Доступные теги outbound'ов
- Найденные плейсхолдеры
- Загрузку паттернов из includes
- Количество сгенерированных правил
- Выбор FINAL outbound

---

## ⚠️ Типичные ошибки

| Ошибка | Причина | Решение |
|--------|----------|---------|
| `Template not found` | Файл шаблона не существует | Проверьте путь к файлу |
| `JSON error: ...` | Неверный синтаксис JSON | Проверьте незакрытые скобки, кавычки или неверные плейсхолдеры |
| `Include not found: ...` | Путь к файлу `includes` не может быть разрешен | Проверьте `base_dir` или используйте абсолютные пути |
| `Skip list out='...' (not present in template outbounds)` | Тег outbound в YAML отсутствует в шаблоне | Убедитесь, что теги outbound'ов совпадают между шаблоном и YAML |

**Примечание:** Скрипт автоматически обрабатывает trailing commas в JSON шаблонах, но другие ошибки синтаксиса JSON должны быть исправлены вручную.

---

## 📜 Лицензия

MIT © Ivan Tarasov, 2025  
Свободное использование, модификация и распространение с указанием автора.