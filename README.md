# Проект базы данных для страховой компании

## Описание

Данный проект предназначен для разработки базы данных для страховой компании, которая имеет множество филиалов по всей стране. База данных будет включать в себя информацию о клиентах, договорах, страховых полисах, филиалах и агентах, с возможностью расчета заработной платы для агентов.

## Задача

Создание базы данных, которая будет включать следующие сущности:
- **Клиенты**
- **Договоры**
- **Виды страхования**
- **Филиалы**
- **Агенты**

Каждая сущность будет иметь свои атрибуты и взаимосвязи с другими сущностями. 

## Содержание

1. [Создание табличного пространства](#создание-табличного-пространства)
2. [Создание схемы и таблиц](#создание-схемы-и-таблиц)
3. [Создание пользователей и назначение привилегий](#создание-пользователей-и-назначение-привилегий)
4. [Хранимая процедура для расчета заработной платы](#хранимая-процедура-для-расчета-заработной-платы)
5. [Создание триггера для ограничения обновления договоров](#создание-триггера-для-ограничения-обновления-договоров)
6. [Создание представления](#создание-представления)
7. [Создание составного фильтрованного индекса](#создание-составного-фильтрованного-индекса)

## Создание табличного пространства

В начале создаем табличное пространство для хранения всех данных базы:

```sql
CREATE TABLESPACE insurance_space LOCATION '/path/to/your/location';
```
## Создание-схемы-и-таблиц
```sql
CREATE SCHEMA InsuranceCompany;

CREATE TABLE InsuranceCompany.Client (
    client_id SERIAL PRIMARY KEY,
    last_name VARCHAR(50) NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    patronymic VARCHAR(50),
    address VARCHAR(100),
    phone VARCHAR(15)
);

CREATE TABLE InsuranceCompany.Contract (
    contract_id SERIAL PRIMARY KEY,
    contract_date DATE NOT NULL,
    insurance_amount DECIMAL(10, 2) NOT NULL,
    tariff_rate DECIMAL(5, 2) NOT NULL,
    branch_id INT REFERENCES InsuranceCompany.Branch(branch_id),
    insurance_type_id INT REFERENCES InsuranceCompany.InsuranceType(insurance_type_id)
);

CREATE TABLE InsuranceCompany.InsuranceType (
    insurance_type_id SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL
);

CREATE TABLE InsuranceCompany.Branch (
    branch_id SERIAL PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    address VARCHAR(100),
    phone VARCHAR(15)
);

CREATE TABLE InsuranceCompany.Agent (
    agent_id SERIAL PRIMARY KEY,
    last_name VARCHAR(50) NOT NULL,
    first_name VARCHAR(50) NOT NULL,
    patronymic VARCHAR(50),
    address VARCHAR(100),
    phone VARCHAR(15),
    branch_id INT REFERENCES InsuranceCompany.Branch(branch_id)
);
```

## Создание пользователей и назначение привилегий
```sql
CREATE USER admin WITH PASSWORD 'admin_password';
CREATE USER staff WITH PASSWORD 'staff_password';

GRANT ALL PRIVILEGES ON SCHEMA InsuranceCompany TO admin;
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA InsuranceCompany TO staff;
```

## Хранимая процедура для расчета заработной платы
```sql
CREATE OR REPLACE FUNCTION CalculateAgentSalary()
RETURNS VOID AS $$
DECLARE
    agent RECORD;
BEGIN
    FOR agent IN SELECT a.agent_id, SUM(c.insurance_amount * c.tariff_rate) AS total_payment
                  FROM InsuranceCompany.Agent a
                  JOIN InsuranceCompany.Contract c ON a.agent_id = c.agent_id
                  GROUP BY a.agent_id LOOP
        -- Логика для расчета заработной платы на основе процента
        -- Например: UPDATE Agent SET salary = agent.total_payment * 0.02 WHERE agent_id = agent.agent_id;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```
## Создание триггера для ограничения обновления договоров
```sql
CREATE OR REPLACE FUNCTION CheckContractModification()
RETURNS TRIGGER AS $$
BEGIN
    IF NEW.contract_date < NOW() - INTERVAL '3 days' THEN
        RAISE EXCEPTION 'Cannot update contract older than 3 days';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER ContractBeforeUpdate
BEFORE UPDATE ON InsuranceCompany.Contract
FOR EACH ROW EXECUTE FUNCTION CheckContractModification();
```

## Создание представления
```sql
CREATE VIEW View_Agent_Salaries AS
SELECT a.agent_id, a.last_name, a.first_name, SUM(c.insurance_amount * c.tariff_rate) AS salary
FROM InsuranceCompany.Agent a
JOIN InsuranceCompany.Contract c ON a.agent_id = c.agent_id
GROUP BY a.agent_id;
```
## Создание составного фильтрованного индекса
```sql
CREATE INDEX idx_contract_recent
ON InsuranceCompany.Contract (contract_date, insurance_amount)
WHERE contract_date >= '2024-01-01';
```