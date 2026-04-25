КЛЮЧ МОЖНО ВЗЯТЬ В https://github.com/moevm/bsc_Muravin/blob/dev_ci/Generator_Tests/config/.env

ЛИБО СДЕЛАТЬ САМОМУ

# 🤖 LLM Test Generator Action

GitHub Action для автоматической генерации pytest-тестов с помощью языковой модели (LLM).  
Анализирует исходный код, создаёт unit-тесты, запускает их в изолированном окружении, оценивает покрытие
и при необходимости отправляет pull request с результатами.

## 🚀 Быстрый старт

```yaml
name: Auto-generate tests

on:
  pull_request:
    types: [opened, synchronize]
  workflow_dispatch:

jobs:
  generate-tests:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Генерация тестов
        uses: Egor213/llm-test-generator-action@main
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          ai_api_key: ${{ secrets.OPENAI_API_KEY }}
          project_root: 'my_project'            # папка с вашим кодом
          config_path: 'config/config.yaml'     # путь к конфигурационному файлу
          model: 'gpt-4o-mini'
          target_line_coverage: 70

      - name: Загрузка отчёта
        uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-report
          path: my_project/test_analysis_report/   # путь к отчёту, зависит от project_root
```

## 🔑 Обязательные параметры

| Параметр | Описание |
|----------|----------|
| `github_token` | GitHub-токен для создания PR и коммитов |
| `ai_api_key` | API-ключ к LLM (OpenAI / OpenRouter / прокси) |

Все остальные параметры опциональны и имеют значения по умолчанию (см. таблицу ниже).

## ⚙️ Конфигурационный файл (config.yaml)

Действие **требует** наличия конфигурационного файла. Если путь не указан (`config_path` оставлен пустым), 
будет использован файл из дистрибутива генератора – **это крайне не рекомендуется**, потому что он не содержит
вашего ключа и может не подходить под вашу модель.

Минимальный рабочий шаблон `config.yaml`:

```yaml
app:
  max_async_workers: 1

ai:
  llm_provider: openai          # openai / lmstudio / request и др.
  model: "gpt-4o-mini"          # обязательно!
  temperature: 0.5
  base_url: "https://api.openai.com/v1"   # или ваш прокси
  max_generate_retries: 3
  max_fix_attempts: 4
  target_line_coverage: 60

logger:
  log_out: both
  console_level: debug
  file_level: debug
  log_file: logs/app.log
```

Поле **`ai.model` является обязательным** – без него упадёт ошибка валидации Pydantic.

## 📥 Входные параметры (inputs)

| Параметр | Тип | По умолчанию | Описание |
|----------|-----|--------------|----------|
| `github_token` * | `string` | – | GitHub-токен |
| `ai_api_key` * | `string` | – | API-ключ к LLM |
| `generator_version` | `string` | `main` | Версия/тег генератора (ветка или тег в репо `Generator_Tests`) |
| `generator_repo` | `string` | `https://github.com/Egor213/Generator_Tests.git` | URL репозитория генератора |
| `project_root` | `string` | `.` | Корень проекта, где лежит исходный код |
| `tests_dir` | `string` | `tests` | Папка для сгенерированных тестов (относительно `project_root`) |
| `config_path` | `string` | `''` | Путь к `config.yaml` относительно `project_root`. Если пуст – будет использован встроенный конфиг (не рекомендуется) |
| `model` | `string` | `gpt-4o-mini` | Название модели LLM |
| `temperature` | `string` | `0.5` | Temperature для генерации |
| `api_base_url` | `string` | `''` | Базовый URL API (если не указан в конфиге, передаётся через переменную окружения) |
| `max_generate_retries` | `string` | `3` | Максимальное число попыток генерации тестов |
| `max_fix_attempts` | `string` | `4` | Максимальное число попыток починки сломанных тестов |
| `target_line_coverage` | `string` | `70` | Целевой процент покрытия строк |
| `max_async_workers` | `string` | `1` | Количество параллельных воркеров |
| `target_dir` | `string` | `''` | Ограничить генерацию конкретной папкой |
| `target_file` | `string` | `''` | Конкретный файл |
| `target_class` | `string` | `''` | Конкретный класс |
| `target_function` | `string` | `''` | Конкретная функция |
| `pr_mode` | `string` | `true` | Создавать ли pull request с изменениями |
| `base_branch` | `string` | `main` | Базовая ветка для PR |
| `python_version` | `string` | `3.11` | Версия Python в раннере |

* – обязательные

## 📤 Выходные данные (outputs)

| Output | Описание |
|--------|----------|
| `pr_url` | URL созданного pull request (если `pr_mode` включён) |
| `report_path` | Путь к HTML-отчёту внутри раннера (можно использовать для загрузки артефакта) |

## 📁 Куда сохраняются отчёты?

После выполнения генерации в папке проекта создаётся каталог `test_analysis_report/`.  
Пример полного пути:  
```
<project_root>/test_analysis_report/index.html
```

**Чтобы сохранить отчёт как артефакт**, добавьте шаг с `actions/upload-artifact`:

```yaml
- uses: actions/upload-artifact@v4
  if: always()
  with:
    name: test-report
    path: my_project/test_analysis_report/   # замените my_project на свой project_root
```

⚠️ Если такой папки не существует, предупреждение `Warning: No files were found` говорит о том,
что генератор не создал отчёт (например, из-за ошибки до запуска). Проверьте логи шага `Generator run`.

## 🔐 Необходимые разрешения (Permissions)

Для создания pull request и коммита в репозиторий джоба должна иметь:

```yaml
permissions:
  contents: write
  pull-requests: write
```

## 🧪 Примеры рабочих процессов

### 1. Автогенерация на каждый Pull Request

```yaml
on:
  pull_request:
    types: [opened, synchronize]

jobs:
  generate-tests:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: Egor213/llm-test-generator-action@main
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          ai_api_key: ${{ secrets.OPENAI_API_KEY }}
          project_root: 'src'
          config_path: '.github/config.yaml'
          target_line_coverage: 80
          pr_mode: true
```

### 2. Ручной запуск с выбором цели

```yaml
on:
  workflow_dispatch:
    inputs:
      target_file:
        description: 'Файл'
        required: false
      target_function:
        description: 'Функция'
        required: false

jobs:
  generate-tests:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - uses: Egor213/llm-test-generator-action@main
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          ai_api_key: ${{ secrets.OPENAI_API_KEY }}
          project_root: '.'
          target_file: ${{ inputs.target_file }}
          target_function: ${{ inputs.target_function }}
          pr_mode: false   # только генерация, без PR
      - uses: actions/upload-artifact@v4
        if: always()
        with:
          name: test-report
          path: test_analysis_report/
```

## ❗ Часто встречающиеся ошибки

### 1. `Field required [type=missing, input_value=..., input_type=dict]` для `ai.model`
В конфиге не указано поле `model`. Добавьте его в секцию `ai`.

### 2. `ModuleNotFoundError: No module named 'managers'` при запуске сгенерированных тестов
Эта ошибка возникает внутри генератора и **не требует действий от вас**. Генератор сам анализирует
окружение и добавляет нужные пути. Если тесты упали, генератор предпримет несколько попыток исправить их.

### 3. Отчёт не загружается (Warning: No files were found)
Убедитесь, что путь в `actions/upload-artifact` совпадает с `project_root` + `/test_analysis_report/`.
Если папка действительно не создана, проверьте логи шага генерации – возможно, он завершился с ошибкой.

### 4. PR создаётся, но тесты не работают
Генератор по умолчанию сам не запускает итоговый прогон тестов на GitHub. Рекомендуем настроить CI,
который будет запускать тесты в созданном PR (например, через `pytest`).

## 🛠 Как это работает

1. Устанавливается генератор из указанного репозитория (`Generator_Tests`).
2. Собираются исходные файлы, контекст функций, генерируются тесты через LLM.
3. Каждый сгенерированный тест запускается во временной песочнице – ошибки исправляются автоматически.
4. Вычисляется покрытие, при необходимости генерируются дополнительные тесты.
5. Создаётся HTML-отчёт в `test_analysis_report/`.
6. При включённом `pr_mode` создаётся ветка и открывается Pull Request с новыми тестами.
