# Рантайм API и возможности библиотеки clarity

Модуль `clarity_rt` обеспечивает исполнение байткода скомпилированных схем, синтаксический анализ аргументов командной строки, автоматическое форматирование вывода справки и версии, исправление опечаток, валидацию значений, генерацию скриптов автодополнения и интеграцию с конфигурационными файлами.

---

## 1. Подключение: load clarity

```yukino
load clarity: parse, parse_args, try_parse, try_parse_args, Command, ClarityRuntimeError, Config, config_open, config_open_file, generate_bash, generate_zsh, generate_fish
```

---

## 2. Структуры данных и ошибок

### 2.1 ClarityRuntimeError

Структура ошибки времени выполнения, вызываемой через `panic` при критических сбоях валидации значений или повреждении байткода:

```yukino
public struct ClarityRuntimeError {
    message: string
    code: int32
}
```

Коды ошибок валидаторов рантайма:
- `101` — значение целого числа не входит в допустимый диапазон `range(min, max)` (`validate_range_int`).
- `102` — строковое значение не соответствует регулярному выражению `pattern("...")` (`validate_pattern`).
- `103` — целевой путь не существует на диске `exists` (`validate_exists`).
- `104` — родительская директория пути не существует `parent_exists` (`validate_parent_exists`).
- `105` — длина строки меньше допустимого минимума `min_len(n)` (`validate_min_len`).
- `106` — длина строки превышает допустимый максимум `max_len(n)` (`validate_max_len`).
- `107` — переданное значение не входит в список разрешенных вариантов `one_of(...)` (`validate_one_of`).

### 2.2 Коды завершения процесса утилиты

Библиотека `clarity` следует строгим стандартам поведения Unix CLI:
- **Код `0` (Успех):** нормальное завершение программы, а также завершение после вывода встроенной справки (`--help`, `-h`, `help <cmd>`) или версии приложения (`--version`, `-V`, `-v`).
- **Код `2` (Синтаксическая ошибка CLI):** передан неизвестный флаг, не указан обязательный позиционный аргумент, отсутствует значение опции, нарушено условие взаимного исключения флагов или допущена опечатка в имени подкоманды.

---

## 3. Справочник функций синтаксического анализа

### 3.1 parse() -> Command

Выполняет разбор реальных аргументов командной строки текущего процесса, полученных системным интринсиком `args()`.

```yukino
public fun parse() -> Command
```

- **Возвращаемое значение:** вариант сгенерированного перечисления `Command` со всеми распарсенными структурами параметров.
- **Поведение при `--help` и `--version`:** автоматически выводит цветную ANSI-справку или версию и завершает процесс с кодом `0`.
- **Поведение при ошибках аргументов:** выводит форматированное сообщение об ошибке, подсказку с похожей командой или флагом (на основе расстояния Левенштейна) и завершает процесс с кодом `2`.

Пример использования:

```yukino
load std.io: println
load clarity: *

#[clarity::scheme("cli.cty")]
fun clarity_init() { space }

fun main() {
    set cmd = parse()
    switch cmd {
        Command::Build(args) -> println("Сборка: " + args.output)
        Command::Run(args) -> println("Запуск: " + args.file)
    }
}
```

---

### 3.2 parse_args(cli_args: string[]) -> Command

Выполняет синтаксический анализ пользовательского массива строк вместо системных аргументов текущего процесса.

```yukino
public fun parse_args(cli_args: string[]) -> Command
```

- **Параметры:** `cli_args` — массив строк без имени самой программы (например, `["build", "--release", "arm64"]`).
- **Назначение:** используется для модульного тестирования логики аргументов без создания внешнего системного процесса.
- **Ошибки:** при некорректных аргументах ведет себя аналогично `parse()` (завершает процесс с кодом `2`).

Пример тестирования:

```yukino
#[test]
fun test_cli_parsing() {
    set cmd = parse_args(["build", "--release", "x86_64"])
    switch cmd {
        Command::Build(args) -> {
            assert(args.release == true)
            assert(args.target == "x86_64")
        }
        else -> panic("ожидалась команда build")
    }
}
```

---

### 3.3 try_parse() -> Command?

Безопасный вариант разбора аргументов текущего процесса без принудительного завершения программы при ошибках.

```yukino
public fun try_parse() -> Command?
```

- **Возвращаемое значение:** возвращает разобранную команду `Command`, либо `null`, если при синтаксическом анализе произошла ошибка.

---

### 3.4 try_parse_args(cli_args: string[]) -> Command?

Безопасный вариант синтаксического анализа произвольного среза строк.

```yukino
public fun try_parse_args(cli_args: string[]) -> Command?
```

- **Возвращаемое значение:** возвращает `Command`, либо `null` при ошибке анализа переданных аргументов.

---

## 4. Работа с файлами конфигураций: класс Config

Библиотека `clarity` предоставляет встроенный класс `Config` для чтения и сохранения настроек в формате TOML.

### 4.1 Фабричные функции создания Config

#### config_open(app_name: string) -> Config

Открывает или автоматически создает конфигурационный файл приложения в системной директории пользователя `~/.config/clarity-cli/<app_name>/config.toml`.

```yukino
set cfg = config_open("mytool")
```

#### config_open_file(file_path: string) -> Config

Открывает конфигурационный файл по произвольному указанному пути. Если родительские папки отсутствуют, они создаются автоматически.

```yukino
set cfg = config_open_file("/etc/mytool/settings.toml")
```

---

### 4.2 Методы класса Config

#### get_str(key: string, fallback: string) -> string
Возвращает строковое значение по ключу, либо значение `fallback`, если ключ отсутствует.

#### get_int(key: string, fallback: int64) -> int64
Возвращает 64-битное целое число по ключу либо значение `fallback`.

#### get_float(key: string, fallback: float64) -> float64
Возвращает число с плавающей точкой по ключу либо `fallback`.

#### get_bool(key: string, fallback: bool) -> bool
Возвращает логическое значение по ключу либо `fallback`.

#### get_str_array(key: string) -> string[]
Возвращает массив строк по ключу. Если ключ отсутствует, возвращает пустой срез.

#### set_str(key: string, val: string)
Записывает строковое значение по ключу в документ конфигурации.

#### set_int(key: string, val: int64)
Записывает целочисленное значение по ключу.

#### set_bool(key: string, val: bool)
Записывает логическое значение по ключу.

#### has(key: string) -> bool
Проверяет наличие ключа в документе конфигурации.

#### keys() -> string[]
Возвращает список всех верхнеуровневых ключей документа.

#### remove(key: string)
Удаляет ключ из документа конфигурации.

#### save() -> bool
Сохраняет текущее состояние документа в файл на диске. Возвращает `true` при успешной записи.

Пример использования:

```yukino
load clarity: config_open

fun save_user_preference() {
    set cfg = config_open("mytool")
    cfg.set_str("user.theme", "dark")
    cfg.set_int("user.font_size", 14)
    cfg.set_bool("telemetry.enabled", false)
    cfg.save()
}
```

---

## 5. Генерация скриптов автодополнения для терминалов

Библиотека включает функции генерации скриптов автодополнения команд и флагов для популярных командных оболочек:

### 5.1 generate_bash(ref schema: RtSchema) -> string

Генерирует сценарий автодополнения для командной оболочки Bash (совместим с `bash-completion`).

### 5.2 generate_zsh(ref schema: RtSchema) -> string

Генерирует сценарий автодополнения для Zsh с поддержкой описаний команд и подсказок в формате `_describe`.

### 5.3 generate_fish(ref schema: RtSchema) -> string

Генерирует скрипт автодополнения для Fish shell с подсветкой и описаниями всех опций.

Пример подкоманды для экспорта скрипта автодополнения:

```yukino
// В схеме cli.cty:
// define completions { param shell: string "Имя оболочки (bash, zsh, fish)"; }

Command::Completions(args) -> {
    // Получение схемы рантайма через vm
    if args.shell == "bash" {
        println(generate_bash(ref vm.schema))
    } elif args.shell == "zsh" {
        println(generate_zsh(ref vm.schema))
    } elif args.shell == "fish" {
        println(generate_fish(ref vm.schema))
    }
}
```

---

[Назад: 1. Язык схем Clarity (`.cty`)](1-schema.md) | [Наверх: 0. Обзор и быстрый старт](0-overview.md)
