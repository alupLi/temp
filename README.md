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
