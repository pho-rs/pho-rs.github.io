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

## Подготовка сервера к олимпиаде

### 1. Загрузка данных пользователей

```bash
# Переходим в директорию с данными
cd ipho_data/

# Созздаём файл "create_db.py"
nano create_db.py
```

```Python
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

Папка с данными "your_user_data" должна содержать 4 файла, как они должны быть заполнены можно посмотреть в "mock_data":

* 011_organizer_user.csv
* 010_olyexams_superuser.csv
* 020_delegations.csv
* 022_students.csv

```bash
# Запускаем заполнение базы данных
uv run python create_db.py
```

После окончания работы все пароли будут в папке "your_user_data/pws"

### 2. Загрузка шрифтов




## Тестирование системы перед запуском

