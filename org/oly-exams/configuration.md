---
title: Конфигурация
layout: default
nav_order: 2
parent: Oly-exams
---

# Конфигурация портала oly-exams
{: .no_toc }

<details open markdown="block" class="toc-h2-only">
  <summary>Содержание</summary>
  {: .text-delta }
- TOC
{:toc}
</details>

---

## Загрузка данных пользователей

```bash
# Переходим в директорию с данными
cd ipho_data/

# Созздаём файл "create_db.py"
nano create_db.py
```

```python
import non_install_helper

import ipho_data.django_setup
from ipho_data.test_data_creator import TestDataCreator


def set_up_basic_database():

    tdc = TestDataCreator(db_name="ipho.db", data_path="your_user_data") # your_user_data заменить на путь к папке с данными делегаций

    tdc.init_database()
    tdc.create_groups()
    tdc.create_olyexams_superuser(pw_strategy="create") # создаёт суперюзера(опционально если уже создан)
    tdc.create_organizer_user(pw_strategy="create")
    tdc.create_delegation_user(pw_strategy="create", enforce_iso3166=False)
    tdc.create_students()

    tdc.create_official_delegation()
    exam = tdc.create_exam(name="Theory", code="T")
    tdc.create_exam_phases_for_exam(exam)
    exam = tdc.create_exam(name="Experiment", code="E")
    tdc.create_exam_phases_for_exam(exam)
    tdc.put_students_in_teams(exam) # создаёт одну команду на делегацию и объеденяет всех студентов в неё
    exam = tdc.create_exam(name="Test", code="Q")
    tdc.create_exam_phases_for_exam(exam)
    tdc.put_students_in_teams(exam)

def main():
    set_up_basic_database()

if __name__ == "__main__":
    main()
```

Папка с данными `your_user_data` должна содержать 4 файла, как они должны быть заполнены можно посмотреть в `mock_data`:

* 011_organizer_user.csv
* 010_olyexams_superuser.csv
* 020_delegations.csv
* 022_students.csv

```bash
# Запускаем заполнение базы данных
uv run python create_db.py
```

После окончания работы все пароли будут в папке `your_user_data/pws`

---

## Добавление нового шрифта

Добавление шрифта состоит из трёх шагов: загрузить файлы, создать файл CSS и сделать запись о шрифте в `fonts.py`.

### 1. Подготовьте файлы шрифта

Все шрифты лежат в `static/noto/`. Для **каждого начертания**, которое есть у шрифта (regular, bold, italic, bold-italic), нужно положить **четыре файла** с одинаковым базовым именем и разными расширениями:

| Расширение | Назначение |
|------------|------------|
| `.ttf`  | основной формат |
| `.eot`  | старые версии Internet Explorer |
| `.woff` | широко поддерживаемый веб-формат |
| `.woff2`| современный сжатый формат |

Пример: если у шрифта есть Regular и Bold, в папке должны быть 8 файлов:

```
MyFont-Regular.ttf   MyFont-Regular.eot   MyFont-Regular.woff   MyFont-Regular.woff2
MyFont-Bold.ttf      MyFont-Bold.eot      MyFont-Bold.woff      MyFont-Bold.woff2
```

Для конвертации `.ttf` можно использовать:
- **Font Squirrel Webfont Generator** (онлайн)
- утилиты `ttf2eot`, `sfnt2woff`, `woff2_compress` из пакета `fonttools`

```bash
# Bold
ttf2eot MyFont-Bold.ttf MyFont-Bold.eot
sfnt2woff MyFont-Bold.ttf
woff2_compress MyFont-Bold.ttf
```


### 2. Создайте CSS-файл

В `static/noto/` создаётся CSS-файл `myfont.css`. В нём — **по одному блоку `@font-face` на каждое начертание**.

Правила заполнения полей:

| Поле в `@font-face` | Значение |
|---------------------|----------|
| `font-family` | Одно и то же имя во всех блоках. Должно совпадать со значением поля `font` в `fonts.py` (см. шаг 3). |
| `font-style`  | `normal` для обычного, `italic` для курсива |
| `font-weight` | `400` для regular, `700` для bold |
| `src`         | Сначала `.eot` отдельной строкой, затем цепочка `.eot?#iefix` → `.woff2` → `.woff` → `.ttf`.|

Пример `myfont.css` с двумя начертаниями:

```css
@font-face {
  font-family: 'My Font';
  font-style: normal;
  font-weight: 400;
  src: url(/static/noto/MyFont-Regular.eot);
  src: url(/static/noto/MyFont-Regular.eot?#iefix) format('embedded-opentype'),
       url(/static/noto/MyFont-Regular.woff2) format('woff2'),
       url(/static/noto/MyFont-Regular.woff) format('woff'),
       url(/static/noto/MyFont-Regular.ttf) format('truetype');
}

@font-face {
  font-family: 'My Font';
  font-style: normal;
  font-weight: 700;
  src: url(/static/noto/MyFont-Bold.eot);
  src: url(/static/noto/MyFont-Bold.eot?#iefix) format('embedded-opentype'),
       url(/static/noto/MyFont-Bold.woff2) format('woff2'),
       url(/static/noto/MyFont-Bold.woff) format('woff'),
       url(/static/noto/MyFont-Bold.ttf) format('truetype');
}
```

Если у шрифта только Regular — достаточно одного блока. Если четыре начертания — четыре блока с комбинациями `(400, normal)`, `(700, normal)`, `(400, italic)`, `(700, italic)`.


### 3. Зарегистрируйте шрифт в `fonts.py`

Откройте `ipho_exam/fonts.py` и добавьте запись в словарь `ipho`. Ключом служит внутренний идентификатор (латиницей, без пробелов), значением — словарь с полями:

| Поле | Значение |
|------|------------|
| `name`  | Тот же идентификатор, что и ключ словаря. |
| `font`  | Имя для UI. **Должно совпадать с `font-family` в CSS.**.|
| `css`   | Имя CSS-файла из шага 2 (относительно `static/noto/`). |
| `font_regular` | Имя `.ttf`-файла regular. |
| `font_bold` | Имя `.ttf`-файла bold (если есть). |
| `font_italic` | Имя `.ttf`-файла italic (если есть). |
| `font_bolditalic` | Имя `.ttf`-файла bold-italic (если есть). |
| `cjk`   | `1` для CJK-шрифтов (японский, китайский, корейский), иначе `0`. |

Пример — шрифт с четырьмя начертаниями:

```python
"myfont": {
    "name": "myfont",
    "font": "My Font",
    "css": "myfont.css",
    "font_regular": "MyFont-Regular.ttf",
    "font_bold": "MyFont-Bold.ttf",
    "font_italic": "MyFont-Italic.ttf",
    "font_bolditalic": "MyFont-BoldItalic.ttf",
    "cjk": 0,
},
```

Несколько шрифтов могут делить один CSS, если в нём есть соответствующие `@font-face` с разными `font-family`.

---


