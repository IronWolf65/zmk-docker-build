# Локальная сборка прошивки ZMK в Docker-контейнере

[English](README.md) | **Русский**

> ⚠️ Репозиторий создан при помощи ИИ.

Сборка идёт в официальном Docker-контейнере ZMK, а на выходе получаются
файлы прошивки в формате `.uf2`. Дополнительно прописывать флаги не нужно —
всё берётся из файлов конфигурации клавиатуры.

## Требования

- Docker Compose
- `make`
- Python 3 на хосте — нужен для `scripts/select_image.py` (подбор совместимого
  образа) и `scripts/build.py`
- Доступ в интернет: Docker Hub (образ) и GitHub (west-зависимости при первой сборке)

## Быстрый старт

```sh
make build
```

Это соберёт все цели из `build.yaml`. При первом запуске скачивается довольно
объёмный west-workspace, поэтому первая сборка заметно длиннее последующих.

Если нет `build.yaml` или `config/west.yml`, сборка остановится ещё до запуска
Docker и подскажет нужную структуру конфига клавиатуры.

Готовые файлы появятся в `output/`:

- `output/<name>.uf2` — то, что заливается на контроллер (bootloader, двойной сброс);
- `output/other/<name>.*` — прочие артефакты (`.hex`, `.bin`, `.elf`, `.map`).

`<name>` берётся из `artifact-name` в `build.yaml`, иначе собирается из board и
shield.

## Команды

| Команда | Что делает |
|---|---|
| `make build` | Собрать все цели из `build.yaml` |
| `make build DEV=1` | То же, но в dev-образе |
| `make update` | Обновить west-зависимости (`DEV=1` — в dev-образе) |
| `make shell` | Интерактивная bash-оболочка внутри выбранного образа |
| `make clean` | Очистить `output/` (workspace не трогает) |
| `make` / `make help` | Справка по целям |

## Выбор образа

По умолчанию `scripts/select_image.py` подбирает образ под текущую ревизию ZMK:

1. читает проект `zmk` и его `revision` из `config/west.yml`;
2. сверяет её с ref из `.github/workflows/build.yml` и останавливается с ошибкой
   при расхождении — локальная сборка и сборка на GitHub должны использовать одну
   и ту же ревизию ZMK;
3. если в `config/west.yml` ревизии нет — берёт ref из
   `.github/workflows/build.yml`, иначе `main`;
4. скачивает `app/west.yml` этой ревизии ZMK и достаёт версию Zephyr;
5. выбирает `zmkfirmware/zmk-<build|dev>-arm:<Zephyr major.minor>-branch`.

Например, `main` сейчас использует Zephyr 4.1 → `zmkfirmware/zmk-build-arm:4.1-branch`;
ZMK `v0.3` (Zephyr 3.5) → тег `3.5-branch`. Короткая запись `0.3` тоже работает.

Если нужен свой образ, задайте `ZMK_IMAGE` — автоопределение пропускается:

```sh
make build ZMK_IMAGE=zmkfirmware/zmk-build-arm:4.1-branch
```

## Как это устроено

- `build.yaml` — список целей в `include`. Локальная сборка учитывает `board`, `shield`, `cmake-args`, `artifact-name` и `snippet`.
- `config/west.yml` — manifest ZMK и дополнительных модулей.
- `config/xxx.conf`, `config/xxx.keymap` — конфигурация и раскладка.
- `boards/shields/xxx/` — описание шилда.
- `docker-compose.yaml` — контейнер: репозиторий, `./zmk-workspace` и `./output`
  монтируются как есть.
- `scripts/build.py` — инициализирует/обновляет workspace, собирает цели и
  раскладывает артефакты.
- `scripts/select_image.py` — подбор образа (см. выше).

`zmk-workspace/` и `output/` в git не попадают. Удалить workspace, чтобы
скачать его заново, можно простым `rm -rf zmk-workspace`.
