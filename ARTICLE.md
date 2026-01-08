# Автоматизация деплоя Yandex Cloud Functions через GitHub Actions

Пошаговая инструкция по настройке CI/CD для Python функций в Яндекс Облаке.

## 1. Настройка Cursor IDE Project Rules

Cursor IDE использует Project Rules (вместо устаревшего `.cursorrules`). Создайте структуру:

```bash
mkdir -p .cursor/rules
```

Создайте файлы правил в `.cursor/rules/`:

### `python-style.mdc`
```markdown
---
description: Python coding standards and best practices
globs: ["**/*.py"]
alwaysApply: true
---

You are an expert Python developer following modern best practices.

- Follow PEP 8 style guidelines; use Black for formatting (100 char line length)
- Use type hints for all function parameters and return values
- Prefer f-strings for string formatting
- Use dataclasses or Pydantic models for data containers
- Prefer pathlib over os.path for file operations
- Follow PEP 257 for docstrings (Google style preferred)
```

### `lambda-functions.mdc`
```markdown
---
description: Yandex Cloud Functions specific rules
globs: ["**/*.py", "index.py"]
alwaysApply: true
---

## Yandex Cloud Functions

- Keep handler functions simple; extract logic to separate modules
- Validate input parameters early in the handler
- Return appropriate HTTP status codes and error messages
- Use environment variables for configuration (never hardcode secrets)
- Use structured logging (JSON format)
- Place internal libraries in `lib/` directory
- Use relative imports: `from lib.utils import helper`
```

### `code-style.mdc`
```markdown
---
description: Pragmatic programming principles
globs: ["**/*.py"]
alwaysApply: true
---

- Prioritize readability and maintainability
- Start simple; add complexity only when required
- Keep functions focused; do one thing well
- Prefer explicit over implicit
```

### `ai-behavior.mdc`
```markdown
---
description: AI assistant response style
alwaysApply: true
---

- Provide minimal, working code without unnecessary explanation
- Omit comments unless essential
- Focus on the core solution
- Let code speak for itself through clear naming
```

## 2. Создание структуры проекта

```bash
mkdir -p lib .github/workflows
touch index.py requirements.txt .gitignore
```

### `.gitignore`
```
__pycache__/
*.py[cod]
.venv/
venv/
.env
*.log
*.zip
package/
*.key.json
key.json
SETUP.md
```

### `index.py` (пример)
```python
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)


def handler(event: dict, context: dict) -> dict:
    """Обрабатывает HTTP запрос от API Gateway.
    
    Args:
        event: Событие от API Gateway с HTTP запросом
        context: Контекст выполнения функции
        
    Returns:
        Словарь с HTTP ответом (statusCode, headers, body)
    """
    try:
        logger.info(f"Received event: {json.dumps(event)}")
        
        method = event.get("httpMethod", "GET")
        path = event.get("path", "/")
        
        response_body = {
            "message": "Hello from Yandex Cloud Function",
            "method": method,
            "path": path,
        }
        
        return {
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps(response_body, ensure_ascii=False),
        }
    except Exception as e:
        logger.error(f"Error: {str(e)}", exc_info=True)
        return {
            "statusCode": 500,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"error": "Internal server error"}),
        }
```

## 3. Создание GitHub репозитория

1. Откройте [GitHub](https://github.com) и создайте новый репозиторий
2. **НЕ** инициализируйте README, .gitignore или лицензию
3. Скопируйте HTTPS URL репозитория

## 4. Настройка Yandex Cloud

### 4.1. Установка YC CLI

**macOS:**
```bash
brew install yandex-cloud-cli
```

**Linux:**
```bash
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash
```

Проверка:
```bash
yc --version
```

### 4.2. Инициализация YC CLI

```bash
yc init
```

При настройке:
- На вопрос про Compute zone ответьте `n` (не нужно для Serverless Functions)
- Выберите облако и каталог

### 4.3. Создание функции

```bash
yc serverless function create --name learn-words
```

**Скопируйте выведенный `id`** — это `YC_FUNCTION_ID`.

### 4.4. Получение FOLDER_ID

```bash
yc config get folder-id
```

**Скопируйте выведенный ID** — это `YC_FOLDER_ID`.

### 4.5. Создание сервисного аккаунта

```bash
# Создать сервисный аккаунт
yc iam service-account create --name github-actions-deploy

# Получить ID
SA_ID=$(yc iam service-account get --name github-actions-deploy --format json | jq -r '.id')

# Назначить роли (замените <FOLDER_ID>)
yc resource-manager folder add-access-binding <FOLDER_ID> \
  --role serverless.functions.invoker \
  --subject serviceAccount:$SA_ID

yc resource-manager folder add-access-binding <FOLDER_ID> \
  --role editor \
  --subject serviceAccount:$SA_ID

# Создать ключ
yc iam key create --service-account-name github-actions-deploy --output key.json
```

## 5. Настройка GitHub Secrets

Откройте репозиторий → **Settings** → **Secrets and variables** → **Actions** → **New repository secret**

Добавьте секреты:

**Секрет 1: `YC_FUNCTION_ID`**
- Name: `YC_FUNCTION_ID`
- Secret: вставьте ID функции из шага 4.3

**Секрет 2: `YC_FOLDER_ID`**
- Name: `YC_FOLDER_ID`
- Secret: вставьте FOLDER_ID из шага 4.4

**Секрет 3: `YC_SERVICE_ACCOUNT_KEY`**
- Name: `YC_SERVICE_ACCOUNT_KEY`
- Secret: откройте файл `key.json` и скопируйте весь JSON (включая `{` и `}`)

## 6. Настройка GitHub Actions

Создайте `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Yandex Cloud

on:
  push:
    branches:
      - main
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Prepare deployment directory
        run: |
          mkdir -p package
          cp *.py package/ 2>/dev/null || true
          [ -d "lib" ] && cp -r lib package/ || true
          [ -f "requirements.txt" ] && cp requirements.txt package/ || true
          
          if [ ! "$(ls -A package)" ]; then
            echo "Error: Package directory is empty"
            exit 1
          fi
          
          if [ ! -f "package/index.py" ]; then
            echo "Error: index.py not found"
            exit 1
          fi
      
      - name: Deploy to Yandex Cloud Functions
        uses: yc-actions/yc-sls-function@v4
        with:
          yc-sa-json-credentials: ${{ secrets.YC_SERVICE_ACCOUNT_KEY }}
          folder-id: ${{ secrets.YC_FOLDER_ID }}
          function-id: ${{ secrets.YC_FUNCTION_ID }}
          runtime: 'python311'
          memory: '256Mb'
          entrypoint: 'index.handler'
          execution-timeout: '3s'
          environment: |
            PYTHONUNBUFFERED=1
          include: |
            package/**/*
```

## 7. Первый коммит и пуш

```bash
git init
git add .
git commit -m "Initial commit: setup YC function project"
git branch -M main
git remote add origin https://github.com/username/learn-words.git
git push -u origin main
```

**Примечание:** При первом push GitHub может запросить авторизацию. Используйте Personal Access Token (не пароль):
- GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
- Права: `repo` и `workflow`
- Используйте токен вместо пароля

## 8. Проверка деплоя

После пуша:
1. Перейдите в **Actions** → **Deploy to Yandex Cloud**
2. Дождитесь завершения workflow
3. При успехе функция автоматически задеплоится

Для ручного запуска: **Actions** → **Deploy to Yandex Cloud** → **Run workflow**

## Рекомендации

### Использование готовых Actions

Используйте официальные GitHub Actions от Yandex Cloud:
- `yc-actions/yc-sls-function@v4` — для Serverless Functions
- Поддерживает JSON ключи сервисных аккаунтов
- Автоматически упаковывает код

### Внутренние библиотеки

Размещайте внутренние библиотеки в `lib/`:
```
lib/
  utils.py
  helpers.py
```

Использование:
```python
from lib.utils import helper_function
```

### Переменные окружения

Добавляйте через параметр `environment`:
```yaml
environment: |
  DEBUG=true
  API_KEY=${{ secrets.API_KEY }}
  LOG_LEVEL=INFO
```

## Решение проблем

### Ошибка аутентификации
- Проверьте, что секрет `YC_SERVICE_ACCOUNT_KEY` содержит полный JSON
- Убедитесь, что сервисный аккаунт имеет нужные роли

### Функция не найдена (404)
- Проверьте `YC_FUNCTION_ID` в секретах
- Убедитесь, что функция создана в правильном каталоге

### Package directory is empty
- Проверьте, что `index.py` есть в корне проекта
- Убедитесь, что файлы не игнорируются `.gitignore`

## Выводы

Автоматизация деплоя через GitHub Actions упрощает процесс:
- ✅ Автоматический деплой при пуше в `main`
- ✅ Возможность ручного запуска через `workflow_dispatch`
- ✅ Безопасное хранение секретов в GitHub
- ✅ Использование официальных Actions от Yandex Cloud
