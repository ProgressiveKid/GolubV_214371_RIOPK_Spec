# GolubV_214371_RIOPK_Client
# ** Программное средство управления корпоративными рисками на основе внутреннего аудита **
Цель проекта — автоматизировать процессы выявления, оценки и мониторинга корпоративных рисков на основании данных внутреннего аудита. Система предоставляет удобный интерфейс для внутренних аудиторов, управляющих организацие и администраторов, а также реализует механизмы безопасности, авторизации пользователей и централизованного хранения информации о рисках.

**Сервер**: [ProgressiveKid/GolubV_214371_RIOPK_Server](https://github.com/ProgressiveKid/GolubV_214371_RIOPK_Server)
**Клиент**: [ProgressiveKid/GolubV_214371_RIOPK_Client](https://github.com/ProgressiveKid/GolubV_214371_RIOPK_Client)

---
## **Содержание**

1. [Архитектура](#архитектура)
2. [Функциональные возможности](#функциональные-возможности)
3. [Детали реализации](#детали-реализации)
4. [Тестирование](#тестирование)
5. [Установка и запуск](#установка-и-запуск)
6. [Лицензия](#лицензия)
7. [Контакты](#контакты)

---

## **Архитектура**

### C4-модель

#### Контекстный уровень
![image](https://github.com/user-attachments/assets/55fea538-845e-48b1-84d4-4b5c0cdda771)


#### Контейнерный уровень
![image](https://github.com/user-attachments/assets/7d51e7d4-9941-4837-945e-83d4b78fb5af)


#### Компонентный уровень
![image](https://github.com/user-attachments/assets/d8d49aaa-a3b6-4c04-a699-20109198a787)


### Схема данных
![image](https://github.com/user-attachments/assets/09b68576-134c-403f-8cb8-aebc16c41241)


```sql
-- SQL-скрипт для создания базы данных
-- Сброс схемы для чистой генерации
DROP SCHEMA IF EXISTS corp_risk_management CASCADE;
CREATE SCHEMA corp_risk_management;
SET search_path TO corp_risk_management;

-- Таблица пользователей
CREATE TABLE users (
    user_id SERIAL PRIMARY KEY,
    username VARCHAR(100) NOT NULL UNIQUE,
    password_hash TEXT NOT NULL,
    email VARCHAR(150) NOT NULL UNIQUE,
    full_name VARCHAR(200) NOT NULL,
    role VARCHAR(50) NOT NULL CHECK (role IN ('Auditor', 'Manager', 'Administrator'))
);

-- Таблица подразделений
CREATE TABLE departments (
    department_id SERIAL PRIMARY KEY,
    name VARCHAR(150) NOT NULL UNIQUE,
    description TEXT
);

-- Таблица рисков
CREATE TABLE risks (
    risk_id SERIAL PRIMARY KEY,
    title VARCHAR(150) NOT NULL,
    description TEXT,
    severity VARCHAR(20) NOT NULL CHECK (severity IN ('Low', 'Medium', 'High')),
    likelihood VARCHAR(20) NOT NULL CHECK (likelihood IN ('Low', 'Medium', 'High')),
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    created_by_id INT NOT NULL REFERENCES users(user_id) ON DELETE CASCADE
);

-- Таблица оценок рисков
CREATE TABLE risk_assessments (
    assessment_id SERIAL PRIMARY KEY,
    risk_id INT NOT NULL REFERENCES risks(risk_id) ON DELETE CASCADE,
    assessed_by_id INT NOT NULL REFERENCES users(user_id) ON DELETE SET NULL,
    assessment_date DATE NOT NULL DEFAULT CURRENT_DATE,
    impact_score SMALLINT CHECK (impact_score BETWEEN 1 AND 10),
    probability_score SMALLINT CHECK (probability_score BETWEEN 1 AND 10),
    notes TEXT
);

-- Связь многие-ко-многим между рисками и подразделениями
CREATE TABLE risk_departments (
    risk_id INT NOT NULL REFERENCES risks(risk_id) ON DELETE CASCADE,
    department_id INT NOT NULL REFERENCES departments(department_id) ON DELETE CASCADE,
    PRIMARY KEY (risk_id, department_id)
);

-- Таблица аудиторских отчётов
CREATE TABLE audit_reports (
    report_id SERIAL PRIMARY KEY,
    title VARCHAR(200) NOT NULL,
    created_at TIMESTAMP NOT NULL DEFAULT NOW(),
    author_id INT REFERENCES users(user_id) ON DELETE SET NULL,
    content TEXT,
    department_id INT NOT NULL REFERENCES departments(department_id) ON DELETE RESTRICT
);

-- Связующая таблица: один отчёт содержит много оценок
CREATE TABLE audit_report_assessments (
    report_id INT NOT NULL REFERENCES audit_reports(report_id) ON DELETE CASCADE,
    assessment_id INT NOT NULL REFERENCES risk_assessments(assessment_id) ON DELETE CASCADE,
    PRIMARY KEY (report_id, assessment_id)
);

-- Индексы для ускорения поиска по FK
CREATE INDEX idx_risks_created_by ON risks(created_by_id);
CREATE INDEX idx_assessment_risk ON risk_assessments(risk_id);
CREATE INDEX idx_assessment_user ON risk_assessments(assessed_by_id);
CREATE INDEX idx_audit_author ON audit_reports(author_id);

---

## **Функциональные возможности**

### Диаграмма вариантов использования
![image](https://github.com/user-attachments/assets/8e06d35c-c00d-46d2-85b1-8c6a57de4bc2)


### User-flow диаграмма

      
![image](https://github.com/user-attachments/assets/9c8dc499-5be4-4f9f-bb77-f1d327fcddae)
Рисунок 2.1 – user flow аудитора

 ![image](https://github.com/user-attachments/assets/8c976c8c-001a-42c9-b796-684ba95433dd)
Рисунок 2.2 – user flow руководителя

 ![image](https://github.com/user-attachments/assets/d22bd19a-29b6-4fee-a1cd-343f1f8062b1)
Рисунок 2.3 – user flow администратора



---

## **Детали реализации**

### UML-диаграммы

#### Диаграмма классов
![image](https://github.com/user-attachments/assets/330a5d5e-6bd4-4728-8586-d8642c4b45d9)

#### Диаграмма последовательностей
![image](https://github.com/user-attachments/assets/2cf8e538-64f7-4561-a178-923e4ccbc979)
![image](https://github.com/user-attachments/assets/e24d20aa-3bbd-462a-98d9-7c857da33bf9)

#### Диаграмма компонентов
![Диаграмма компонентов](docs/UML_Component_Diagram.png)

### Спецификация API

Открыть в браузере:  
[http://127.0.0.1:8000/docs](http://127.0.0.1:8000/docs)

OpenAPI YAML-файл: `openapi.json` генерируется автоматически.

### Безопасность

Использована JWT-аутентификация:
```python
from jose import jwt

def create_access_token(data: dict, expires_delta: timedelta = None):
    to_encode = data.copy()
    expire = datetime.utcnow() + (expires_delta or timedelta(minutes=ACCESS_TOKEN_EXPIRE_MINUTES))
    to_encode.update({"exp": expire})
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)
```

### Оценка качества кода

Анализ с использованием `flake8`, `pylint` и `radon`:
- Cyclomatic Complexity (radon): нормальный уровень
- Ошибки статики (`flake8`): отсутствуют
- Code Rating (`pylint`): > 8.0

![Оценка_качества_кода](docs/Оценка_качества_кода.png)

---

## **Тестирование**

### Unit-тесты

Покрытие:
- Проверка хеширования пароля
- Проверка генерации токена
- Проверка корневого эндпоинта `/`
- Проверка `/me`
- Проверка обработки входа

Файл: `app/tests/test_unit_auth.py`
```python
def test_verify_password():
    assert verify_password("123", get_password_hash("123"))
```

### Интеграционные тесты

Файл: `app/tests/test_integration_auth.py`, `app/tests/test_integration_protected.py`
```python
def test_login_user():
    response = client.post("/login", data={"email": "test@example.com", "password": "test"})
    assert response.status_code == 200
```

![Тесты](docs/Тесты.png)

---

## **Установка и запуск**

### Манифесты для сборки docker образов

Файл `Dockerfile`:
```dockerfile
FROM python:3.10
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Манифесты для развертывания k8s кластера

Файл `deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: riopk-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: riopk
  template:
    metadata:
      labels:
        app: riopk
    spec:
      containers:
        - name: riopk
          image: riopk:latest
          ports:
            - containerPort: 8000
```
