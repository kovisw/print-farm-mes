# Модель данных

Производственная часть схемы. Таблицы каталога и заказов магазина здесь показаны только там, где они связаны с производством.

```mermaid
erDiagram
    product_variants ||--o{ variant_parts : "состоит из"
    print_parts ||--o{ variant_parts : "входит в"
    orders ||--o{ order_items : "состав"
    order_items ||--o{ print_jobs : "изготовление"
    print_parts ||--o{ print_jobs : "что печатать"
    printers |o--o{ print_jobs : "где печатать"
    print_jobs ||--o{ job_status_history : "журнал"
    filaments |o--o{ print_parts : "пластик"
    filaments |o--o{ printers : "заправлен"

    print_parts {
        string code "PART-A"
        int version "1 → файл _v01"
        int est_time_s
        decimal est_filament_g
    }
    variant_parts {
        int variant_id
        int part_id
        int quantity
    }
    print_jobs {
        string gcode_filename "снимок версии"
        string status
        int priority
        datetime queued_at
        datetime started_at
        datetime finished_at
        int actual_time_s
        decimal actual_filament_g
        string error_message
    }
    job_status_history {
        string from_status
        string to_status
        string source "site / mes / operator / bot"
        datetime changed_at
    }
    printers {
        string code "PRN-01"
        string status
        bool has_camera
        int total_print_seconds
        int maintenance_interval_h
        datetime last_maintenance_at
    }
```

## Решения

**Стол — единица производства.** `print_parts` — один G-code на стол принтера. `variant_parts` — спецификация: из каких столов и в каком количестве собирается товар. Заказ создает по заданию на каждый стол × количество.

```
Комплект из двух частей
  → PART-A   (первая часть)
  → PART-B   (вторая часть)
```

Новый товар из уже существующих столов не требует нового G-code — только спецификации.

**Версии файлов.** Имя файла на принтере — `КОД_vNN.gcode`. Изменилась модель или настройки печати — новая версия, старый файл не перезаписывается. Задание хранит имя файла, которым печаталось, поэтому видно, чем печатали каждый заказ, а уже поставленные задания допечатываются своей версией.

**Журнал статусов.** Каждый переход задания пишется в `job_status_history` вместе с источником (сайт, MES, оператор, бот). Переходы проверяются автоматом состояний, недопустимый переход — ошибка, повтор того же статуса игнорируется.

**Нет IP в базе.** У принтера в БД только код `PRN-01`. Сетевые адреса Moonraker живут в конфиге MES — база магазина не знает о сети фермы.

**Учет и ТО.** По заданиям копится фактическое время и расход пластика, по принтеру — наработка. Когда наработка с последнего ТО превышает интервал, принтер просится на обслуживание.
