# temp

#1
```
-- Создание триггерной функции
CREATE OR REPLACE FUNCTION check_positive_price()
RETURNS TRIGGER AS
$$
BEGIN
    IF NEW.price <= 0 THEN
        RAISE EXCEPTION 'Цена товара должна быть положительной';
    END IF;
    RETURN NEW;
END;
$$
LANGUAGE plpgsql;

-- Создание триггера
CREATE TRIGGER trg_check_positive_price
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION check_positive_price();
```
#2
```
-- Создание триггерной функции
CREATE OR REPLACE FUNCTION log_salary_change()
RETURNS TRIGGER AS
$$
BEGIN
    IF OLD.salary IS DISTINCT FROM NEW.salary THEN
        INSERT INTO salary_log (employee_id, old_salary, new_salary, change_date)
        VALUES (NEW.id, OLD.salary, NEW.salary, NOW());
    END IF;
    RETURN NEW;
END;
$$
LANGUAGE plpgsql;

-- Создание триггера
CREATE TRIGGER trg_log_salary_change
AFTER UPDATE OF salary ON employees
FOR EACH ROW
EXECUTE FUNCTION log_salary_change();
```
#3
```
-- Создание триггерной функции
CREATE OR REPLACE FUNCTION prevent_delete_customer_with_orders()
RETURNS TRIGGER AS
$$
DECLARE
    order_count INTEGER;
BEGIN
    SELECT COUNT(*) INTO order_count
    FROM orders
    WHERE customer_id = OLD.id;

    IF order_count > 0 THEN
        RAISE EXCEPTION 'Невозможно удалить клиента с существующими заказами';
    END IF;

    RETURN OLD;
END;
$$
LANGUAGE plpgsql;

-- Создание триггера
CREATE TRIGGER trg_prevent_delete_customer
BEFORE DELETE ON customers
FOR EACH ROW
EXECUTE FUNCTION prevent_delete_customer_with_orders();
```
#4
```
-- Создание триггерной функции
CREATE OR REPLACE FUNCTION calculate_total()
RETURNS TRIGGER AS
$$
BEGIN
    IF NEW.total IS NOT NULL THEN
        RAISE EXCEPTION 'Нельзя вручную задавать значение total';
    END IF;

    NEW.total := NEW.quantity * NEW.price;
    RETURN NEW;
END;
$$
LANGUAGE plpgsql;

-- Создание триггера
CREATE TRIGGER trg_calculate_total
BEFORE INSERT OR UPDATE ON order_items
FOR EACH ROW
EXECUTE FUNCTION calculate_total();
```


```
-- Подготовка таблицы
DROP TABLE IF EXISTS products CASCADE;
CREATE TABLE products(
    id SERIAL PRIMARY KEY,
    product_name TEXT NOT NULL,
    price NUMERIC NOT NULL
);

-- Создание триггера (функция из предыдущего ответа)
CREATE OR REPLACE FUNCTION check_positive_price()
RETURNS TRIGGER AS
$$
BEGIN
    IF NEW.price <= 0 THEN
        RAISE EXCEPTION 'Цена товара должна быть положительной';
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_check_positive_price
BEFORE INSERT OR UPDATE ON products
FOR EACH ROW
EXECUTE FUNCTION check_positive_price();

-- ✅ Успешная вставка
INSERT INTO products (product_name, price) VALUES ('Ноутбук', 50000);

-- ❌ Ошибочная вставка
INSERT INTO products (product_name, price) VALUES ('Неверный товар', 0);
-- Ожидаем: ERROR: Цена товара должна быть положительной

-- ❌ Ошибочное обновление
UPDATE products SET price = -100 WHERE product_name = 'Ноутбук';
-- Ожидаем: ERROR: Цена товара должна быть положительной

-- Проверка успешного обновления
UPDATE products SET price = 45000 WHERE product_name = 'Ноутбук';




-- Подготовка таблиц
DROP TABLE IF EXISTS salary_log CASCADE;
DROP TABLE IF EXISTS employees CASCADE;

CREATE TABLE employees(
    id SERIAL PRIMARY KEY,
    full_name TEXT,
    salary NUMERIC
);

CREATE TABLE salary_log(
    log_id SERIAL PRIMARY KEY,
    employee_id INTEGER,
    old_salary NUMERIC,
    new_salary NUMERIC,
    change_date TIMESTAMP
);

-- Создание триггера
CREATE OR REPLACE FUNCTION log_salary_change()
RETURNS TRIGGER AS
$$
BEGIN
    IF OLD.salary IS DISTINCT FROM NEW.salary THEN
        INSERT INTO salary_log (employee_id, old_salary, new_salary, change_date)
        VALUES (NEW.id, OLD.salary, NEW.salary, NOW());
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_log_salary_change
AFTER UPDATE OF salary ON employees
FOR EACH ROW
EXECUTE FUNCTION log_salary_change();

-- Вставка сотрудника
INSERT INTO employees (full_name, salary) VALUES ('Иван Петров', 50000);

-- Обновление зарплаты (должно записать в лог)
UPDATE employees SET salary = 60000 WHERE id = 1;

-- Обновление с той же зарплатой (лог не должен появиться)
UPDATE employees SET salary = 60000 WHERE id = 1;

-- Проверка лога
SELECT * FROM salary_log;
-- Ожидаем: 1 запись (old_salary=50000, new_salary=60000)




-- Подготовка таблиц
DROP TABLE IF EXISTS orders CASCADE;
DROP TABLE IF EXISTS customers CASCADE;

CREATE TABLE customers(
    id SERIAL PRIMARY KEY,
    full_name TEXT
);

CREATE TABLE orders(
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id)
);

-- Создание триггера
CREATE OR REPLACE FUNCTION prevent_delete_customer_with_orders()
RETURNS TRIGGER AS
$$
DECLARE
    order_count INTEGER;
BEGIN
    SELECT COUNT(*) INTO order_count
    FROM orders
    WHERE customer_id = OLD.id;

    IF order_count > 0 THEN
        RAISE EXCEPTION 'Невозможно удалить клиента с существующими заказами';
    END IF;

    RETURN OLD;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_prevent_delete_customer
BEFORE DELETE ON customers
FOR EACH ROW
EXECUTE FUNCTION prevent_delete_customer_with_orders();

-- Вставка клиента и заказа
INSERT INTO customers (full_name) VALUES ('Анна Сидорова');
INSERT INTO orders (customer_id) VALUES (1);

-- ❌ Попытка удалить клиента с заказами
DELETE FROM customers WHERE id = 1;
-- Ожидаем: ERROR: Невозможно удалить клиента с существующими заказами

-- ✅ Удаление заказа, затем клиента
DELETE FROM orders WHERE customer_id = 1;
DELETE FROM customers WHERE id = 1;
-- Успешно



-- Подготовка таблицы
DROP TABLE IF EXISTS order_items CASCADE;

CREATE TABLE order_items(
    id SERIAL PRIMARY KEY,
    order_id INTEGER,
    quantity INTEGER,
    price NUMERIC,
    total NUMERIC
);

-- Создание триггера
CREATE OR REPLACE FUNCTION calculate_total()
RETURNS TRIGGER AS
$$
BEGIN
    IF NEW.total IS NOT NULL THEN
        RAISE EXCEPTION 'Нельзя вручную задавать значение total';
    END IF;

    NEW.total := NEW.quantity * NEW.price;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_calculate_total
BEFORE INSERT OR UPDATE ON order_items
FOR EACH ROW
EXECUTE FUNCTION calculate_total();

-- ✅ Вставка без total (должен вычислиться автоматически)
INSERT INTO order_items (order_id, quantity, price) VALUES (101, 3, 500);
-- total должен стать 1500

-- ✅ Обновление (пересчёт total)
UPDATE order_items SET quantity = 5 WHERE order_id = 101;
-- total должен стать 2500

-- ❌ Попытка вручную задать total
INSERT INTO order_items (order_id, quantity, price, total) VALUES (102, 2, 1000, 2000);
-- Ожидаем: ERROR: Нельзя вручную задавать значение total

-- ❌ Попытка обновить total вручную
UPDATE order_items SET total = 9999 WHERE order_id = 101;
-- Ожидаем: ERROR: Нельзя вручную задавать значение total

-- Проверка результатов
SELECT * FROM order_items;


```



