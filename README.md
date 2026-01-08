# Yandex Cloud Functions Python Template

Шаблон для создания Lambda-функций на Python для Yandex Cloud с автоматическим деплоем через GitHub Actions.

## Использование шаблона

1. Нажмите кнопку **"Use this template"** на странице репозитория
2. Создайте новый репозиторий на основе этого шаблона
3. Следуйте инструкциям в статье для настройки деплоя

## Быстрый старт

### 1. Настройка Yandex Cloud

```bash
# Установка YC CLI
brew install yandex-cloud-cli  # macOS
# или
curl -sSL https://storage.yandexcloud.net/yandexcloud-yc/install.sh | bash  # Linux

# Инициализация
yc init

# Создание функции
yc serverless function create --name your-function-name

# Получение FOLDER_ID
yc config get folder-id

# Создание сервисного аккаунта (см. ARTICLE.md для деталей)
yc iam service-account create --name github-actions-deploy
yc iam key create --service-account-name github-actions-deploy --output key.json
```

### 2. Настройка GitHub Secrets

В настройках репозитория → **Secrets and variables** → **Actions** добавьте:

- `YC_FUNCTION_ID` - ID созданной функции
- `YC_FOLDER_ID` - ID каталога
- `YC_SERVICE_ACCOUNT_KEY` - содержимое файла `key.json`

### 3. Настройка workflow

Отредактируйте `.github/workflows/deploy.yml`:
- Измените `function-name` на название вашей функции
- При необходимости измените `runtime`, `memory`, `execution-timeout`

### 4. Деплой

После пуша в ветку `main` функция автоматически задеплоится через GitHub Actions.

## Разработка

### Установка

```bash
python -m venv .venv
source .venv/bin/activate  # Linux/Mac
pip install -r requirements.txt
```

### Структура

- `index.py` - точка входа (handler function)
- `lib/` - внутренние библиотеки (опционально)

### Pull Requests

1. Создайте форк репозитория
2. Создайте ветку для ваших изменений: `git checkout -b feature/your-feature`
3. Внесите изменения и закоммитьте: `git commit -m "Description of changes"`
4. Запушьте ветку: `git push origin feature/your-feature`
5. Создайте Pull Request в основном репозитории
