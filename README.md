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
```
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
![image](https://github.com/user-attachments/assets/e24d20aa-3bbd-462a-98d9-7c857da33bf9)

### Спецификация API

Открыть в браузере:  
[http://127.0.0.1:8000/docs](http://127.0.0.1:7124/swagger)

SWAGGER YAML-файл: `swagger.json` генерируется автоматически.

### Безопасность

Использована куки-аунтефикация:
```
private async Task Authenticate(string userName, Role role)
{
    var claims = new List<Claim>
    {
        new Claim(ClaimsIdentity.DefaultNameClaimType, userName),
        new Claim("http://schemas.microsoft.com/ws/2008/06/identity/claims/role", role.ToString())
    };

    ClaimsIdentity id = new ClaimsIdentity(claims, "ApplicationCookie", ClaimsIdentity.DefaultNameClaimType, ClaimsIdentity.DefaultRoleClaimType);

    // Создание куки с AuthenticationProperties
    await HttpContext.SignInAsync(CookieAuthenticationDefaults.AuthenticationScheme, new ClaimsPrincipal(id), new AuthenticationProperties
    {
        IsPersistent = true,
        ExpiresUtc = DateTimeOffset.UtcNow.AddYears(1)
    });
}
```

### Оценка качества кода

![image](https://github.com/user-attachments/assets/b760eb15-b0ff-472f-87d9-bc63a8622d50)


---

## **Тестирование**

### Unit-тесты

Покрытие:
- Проверка работы с рисками
- Проверка оценки рисков
- Проверка работы отчетов аудита
- Проверка получения данных о подразделениях
- Проверка работы с пользователями

![image](https://github.com/user-attachments/assets/e3d59399-df44-42ae-85a0-38e5708b34ec)


### Интеграционные тесты

```C#

using CorporateRiskManagementSystemBack.Infrastructure.Data;
using Microsoft.AspNetCore.Mvc.Testing;
using Microsoft.Extensions.DependencyInjection;
using Microsoft.EntityFrameworkCore; // <-- Не забудьте добавить
using System.Net.Http.Json;
using System.Net;
using FluentAssertions;
using Xunit;
using CorporateRiskManagementSystemBack.Domain.Entites.DataTransferObjects.RequestModels;
using CorporateRiskManagementSystemBack.Domain.Entites;
using CorporateRiskManagementSystemBack;

namespace TestCRMS.Integrations
{
    public class ReportControllerIntegrationTests
    {
        private readonly HttpClient _client;
        private readonly WebApplicationFactory<Program> _factory;

        public ReportControllerIntegrationTests()
        {
            _factory = new WebApplicationFactory<Program>()
                .WithWebHostBuilder(builder =>
                {
                    builder.ConfigureServices(services =>
                    {
                        // Здесь нужно использовать InMemoryDatabase для тестов
                        var connection = "InMemoryDbForTesting"; // строка для имитации базы данных в памяти
                        services.AddDbContext<RiskDbContext>(options =>
                            options.UseInMemoryDatabase(connection));
                    });
                });

            _client = _factory.CreateClient();
        }

        [Fact]
        public async Task CreateReport_ReturnsBadRequest_WhenAnyRiskHasNoAssessment()
        {
            // Arrange: создаём риски и департамент с необходимыми данными
            var scopeFactory = _factory.Services.GetRequiredService<IServiceScopeFactory>();
            using var scope = scopeFactory.CreateScope();
            var dbContext = scope.ServiceProvider.GetRequiredService<RiskDbContext>();

            // Начинаем транзакцию
            var transaction = await dbContext.Database.BeginTransactionAsync();

            try
            {
                var userId = dbContext.Users.FirstOrDefault().UserId;
                // Создаём риски
                var department = new Department {  Name = "FinanceForTests" };
                dbContext.Departments.Add(department);

                var risk1 = new Risk
                {
                    Title = "Risk 1",
                    CreatedById = userId,
                    CreatedAt = DateTime.Now,
                    Description = "wake up on buggati",
                    Severity = "High",  // Установите значение для поля severity
                    Likelihood = "Low",
                };

                var risk2 = new Risk
                {
                    Title = "Risk 2",
                    CreatedById = userId,
                    Severity = "High",
                    CreatedAt = DateTime.Now,
                    Description = "wake up on buggati",
                    Likelihood = "Low",
                };

                dbContext.Risks.Add(risk1);
                dbContext.Risks.Add(risk2);
                dbContext.SaveChanges();

                var request = new CreateReportRequest
                {
                    Username = "admin",
                    DepartmentId = department.DepartmentId,
                    Content = "Контент"
                };

                var response = await _client.PostAsJsonAsync("/Report/CreateReport", request);

                // Assert: проверка на BadRequest с нужным сообщением
                response.StatusCode.Should().Be(HttpStatusCode.BadRequest);
                var content = await response.Content.ReadAsStringAsync();
                content.Should().Be("Необходимо выполнить оценку всех существующих рисков для отдела");
            }
            finally
            {
                // Откатываем все изменения, сделанные в рамках транзакции
                await transaction.RollbackAsync();
            }
        }
    }
}

```
---

## **Установка и запуск**

### Манифесты для сборки docker образов

Для корректной работы программного средства необходимы следующие требования:
1.	операционная система Windows 10, 11;
2.	Microsoft .NET 6 SDK;
3.	оперативная память – 2048 Мб;
4.	свободное место на жестком диске – 500 Мб; 

Пошаговая инструкция по запуску:
1.	Откройте Visual Studio и загрузите решение, выбрав файл CorparateRiskmanagment.sln.
2.	В обозревателе решений установите проект CorparateRiskmanagment Backкак стартовый (ПКМ по проекту → «Установить как стартовый проект»). Нажмите F5 или кнопку «Start». После запуска серверное API будет доступно по адресу http://localhost:7111/swagger.
3.	Клиентская часть проекта реализована как Razor Pages-приложение, установите этот проект как стартовый и запустите его также через F5 или кнопку «Start». После запуска клиентское приложение будет доступно по адресу http://localhost:7124.
4.	Зарегистрироваться и авторизация в системе: Обычный пользователь может быть создан через форму регистрации на клиентской части (Razor Pages).

```
