## Triggers, Support for Data Mart


### Preparatory Work

``` sql
DROP SCHEMA IF EXISTS pract_functions CASCADE;
CREATE SCHEMA pract_functions;

SET search_path = pract_functions, public;

DROP TABLE IF EXISTS goods CASCADE;
CREATE TABLE goods
(
    good_id    integer PRIMARY KEY,
    good_name   varchar(63) NOT NULL,
    good_price  numeric(12, 2) NOT NULL CHECK (good_price > 0.0)
);
INSERT INTO goods (good_id, good_name, good_price)
VALUES 	(1, 'Matches', 0.50),
		(2, 'Ferrari FXX K', 185000000.01);

DROP TABLE IF EXISTS sales CASCADE;
CREATE TABLE sales
(
    sales_id    integer GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    good_id     integer REFERENCES goods (good_id),
    sales_time  timestamp with time zone DEFAULT now(),
    sales_qty   integer CHECK (sales_qty > 0)
);
INSERT INTO sales (good_id, sales_qty) VALUES (1, 10), (1, 1), (1, 120), (2, 1);

-- Data Mart table
DROP TABLE IF EXISTS good_sum_mart;
CREATE TABLE good_sum_mart
(
	good_name   varchar(63) NOT NULL,
	sum_sale	numeric(16, 2) NOT NULL
);

```

### Trigger for Data Mart Maintenance

``` sql
CREATE OR REPLACE FUNCTION tf_maintain_showcase()
RETURNS trigger
AS
$$
BEGIN
    IF TG_LEVEL = 'ROW' THEN
        CASE TG_OP
            WHEN 'DELETE' THEN
            	UPDATE good_sum_mart
				SET sum_sale = sum_sale - diff.items_sale
				FROM (
					SELECT G.good_name, G.good_price * OLD.sales_qty as items_sale
					FROM goods G
					WHERE G.good_id = OLD.good_id
				) as diff
				WHERE 
					good_sum_mart.good_name = diff.good_name;
				RETURN OLD;
            WHEN 'UPDATE' THEN
            	UPDATE good_sum_mart
				SET sum_sale = sum_sale + diff.items_sale
				FROM (
					SELECT G.good_name, G.good_price * (NEW.sales_qty - OLD.sales_qty) as items_sale
					FROM goods G
					WHERE G.good_id = NEW.good_id
				) as diff
				WHERE 
					good_sum_mart.good_name = diff.good_name;
				RETURN NEW;
            WHEN 'INSERT' THEN
            	IF EXISTS (
            		SELECT 1
					FROM goods G
					INNER JOIN good_sum_mart S ON S.good_name = G.good_name
					WHERE G.good_id = NEW.good_id
            	)
            	THEN
            		UPDATE good_sum_mart
					SET sum_sale = sum_sale + diff.items_sale
					FROM (
						SELECT G.good_name, G.good_price * NEW.sales_qty as items_sale
						FROM goods G
						WHERE G.good_id = NEW.good_id
					) as diff
					WHERE 
					    good_sum_mart.good_name = diff.good_name;
            	ELSE
            		INSERT INTO good_sum_mart
					SELECT G.good_name, G.good_price * NEW.sales_qty
					FROM goods G
					WHERE G.good_id = NEW.good_id;
            	END IF;
            	RETURN NEW;
        END CASE;
    END IF;
END;
$$ LANGUAGE plpgsql;

-- Create the trigger
CREATE TRIGGER trg_after_r
AFTER INSERT OR UPDATE OR DELETE
ON sales
FOR EACH ROW
EXECUTE FUNCTION tf_maintain_showcase();
```

### Tests

``` sql
-- Insert a new value
INSERT INTO sales (good_id, sales_qty) VALUES (1, 1); -- sales_id=7
Name     |Value               |
---------+--------------------+
good_name|Matches             |
sum_sale |0.50                |

-- Update the quantity of purchased items by +1
UPDATE sales SET sales_qty = 2 WHERE sales_id = 7;
good_name           |sum_sale|
--------------------+--------+
Matches             |    1.00|

-- Delete the record
DELETE FROM sales WHERE sales_id = 7;
good_name           |sum_sale|
--------------------+--------+
Matches             |    0.00|

```