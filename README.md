# **1.2 Триггер: контроль диапазона capacity (1000–120000)**

Создать триггер, который запрещает вставку и изменение записей в таблицу Stadiums, если значение capacity находится вне диапазона от 1000 до 120000:
— создать триггерную функцию, которая выполняет проверку;
— создать триггер BEFORE INSERT OR UPDATE на таблицу Stadiums;
— при нарушении ограничений на вставляемые значения триггер должен выбрасывать понятное исключение.
Продемонстрировать работу триггера на примерах DML-операций, которые как успешно выполняются, так и корректно прерываются триггером (два запроса).

## **Триггерная функция**

```sql
CREATE OR REPLACE FUNCTION kr1_check_stadium_capacity()
RETURNS trigger AS $$
BEGIN
    IF NEW.capacity < 1000 OR NEW.capacity > 120000 THEN
        RAISE EXCEPTION
            'Вместимость стадиона должна быть в диапазоне от 1000 до 120000';
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## **Триггер**

```sql
CREATE TRIGGER trg_check_stadium_capacity
BEFORE INSERT OR UPDATE ON kr1_Stadiums
FOR EACH ROW
EXECUTE FUNCTION kr1_check_stadium_capacity();
```

## **Демонстрация**

❌ Ошибка:

```sql
INSERT INTO kr1_Stadiums (name, capacity, city)
VALUES ('Test Stadium', 500, 'Test City');
```

Результат:

```
ERROR:  Вместимость стадиона должна быть в диапазоне от 1000 до 120000
```

✅ Успех:

```sql
UPDATE kr1_Stadiums
SET capacity = 50000
WHERE stadium_id = 1;
```

---

## **1.3 Триггер: запрет scheduled-матча в прошлом**

Создать триггер, который запрещает вставлять матч со статусом scheduled, если его дата проведения раньше текущего момента:
— создать триггерную функцию, выполняющую проверку;
— создать триггер BEFORE INSERT на таблицу Matches;
— при нарушении ограничений на вставляемые значения триггер должен выбрасывать понятное исключение.
Продемонстрировать работу триггера на примерах DML-операций, которые как успешно выполняются, так и корректно прерываются триггером (два запроса).

### **Триггерная функция**

```sql
CREATE OR REPLACE FUNCTION kr1_check_scheduled_match_date()
RETURNS trigger AS $$
BEGIN
    IF NEW.status = 'scheduled'
       AND NEW.match_date < CURRENT_TIMESTAMP THEN
        RAISE EXCEPTION
            'Нельзя вставлять матч со статусом scheduled с датой в прошлом';
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### **Триггер**

```sql
CREATE TRIGGER trg_check_scheduled_match_date
BEFORE INSERT ON kr1_Matches
FOR EACH ROW
EXECUTE FUNCTION kr1_check_scheduled_match_date();
```

### **Демонстрация**

❌ Ошибка:

```sql
INSERT INTO kr1_Matches (home_team_id, away_team_id, stadium_id, match_date, status)
VALUES (1, 2, 1, '2020-01-01 12:00:00', 'scheduled');
```

✅ Успех:

```sql
INSERT INTO kr1_Matches (home_team_id, away_team_id, stadium_id, match_date, status)
VALUES (1, 2, 1, CURRENT_TIMESTAMP + INTERVAL '1 day', 'scheduled');
```

---

## **1.4 Триггер: games_played = wins + draws + losses**

Необходимо создать триггер, который запрещает вставлять и изменять записи в Standings, если games_played не равно сумме wins + draws + losses:
— создать триггерную функцию check_standings_games_played();
— создать триггер trg_check_standings_games_played типа BEFORE INSERT OR UPDATE на таблицу Standings;
— при нарушении ограничений на вставляемые или изменяемые значения триггер должен выбрасывать понятное исключение.
Продемонстрировать работу триггера на примерах DML-операций, которые как успешно выполняются, так и корректно прерываются триггером (два запроса).

### **Функция**

```sql
CREATE OR REPLACE FUNCTION check_standings_games_played()
RETURNS trigger AS $$
BEGIN
    IF NEW.games_played <> (NEW.wins + NEW.draws + NEW.losses) THEN
        RAISE EXCEPTION
            'games_played должен быть равен wins + draws + losses';
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### **Триггер**

```sql
CREATE TRIGGER trg_check_standings_games_played
BEFORE INSERT OR UPDATE ON kr1_Standings
FOR EACH ROW
EXECUTE FUNCTION check_standings_games_played();
```

### **Демонстрация**

❌ Ошибка:

```sql
UPDATE kr1_Standings
SET games_played = 10
WHERE team_id = 1;
```

✅ Успех:

```sql
UPDATE kr1_Standings
SET games_played = wins + draws + losses
WHERE team_id = 1;
```

---

## **2.1 Процедура пересчёта Standings**

Реализовать хранимшую процедуру recalculate_standings(), которая полностью пересчитывает таблицу Standings по данным о сыгранных матчах из таблицы Matches:
— учитывать только матчи со статусом 'completed';
Для каждой команды из таблицы Teams вычислить:
— games_played — количество сыгранных матчей;
— wins — число побед;
— draws — число ничьих;
— losses — число поражений;
— goals_for — забитые голы;
— goals_against — пропущенные голы;
— points — очки (3 за победу, 1 за ничью).
Привести пример вызова процедуры.

```sql
CREATE OR REPLACE PROCEDURE recalculate_standings()
LANGUAGE plpgsql
AS $$
BEGIN
    TRUNCATE kr1_Standings;

    INSERT INTO kr1_Standings (
        team_id, games_played, wins, draws, losses,
        goals_for, goals_against, points
    )
    SELECT
        t.team_id,
        COUNT(m.match_id),
        SUM(CASE WHEN
            (t.team_id = m.home_team_id AND m.home_score > m.away_score) OR
            (t.team_id = m.away_team_id AND m.away_score > m.home_score)
        THEN 1 ELSE 0 END),
        SUM(CASE WHEN m.home_score = m.away_score THEN 1 ELSE 0 END),
        SUM(CASE WHEN
            (t.team_id = m.home_team_id AND m.home_score < m.away_score) OR
            (t.team_id = m.away_team_id AND m.away_score < m.home_score)
        THEN 1 ELSE 0 END),
        SUM(CASE WHEN t.team_id = m.home_team_id THEN m.home_score ELSE m.away_score END),
        SUM(CASE WHEN t.team_id = m.home_team_id THEN m.away_score ELSE m.home_score END),
        SUM(CASE
            WHEN m.home_score = m.away_score THEN 1
            WHEN (t.team_id = m.home_team_id AND m.home_score > m.away_score)
              OR (t.team_id = m.away_team_id AND m.away_score > m.home_score)
            THEN 3 ELSE 0 END)
    FROM kr1_Teams t
    JOIN kr1_Matches m
        ON t.team_id IN (m.home_team_id, m.away_team_id)
    WHERE m.status = 'completed'
    GROUP BY t.team_id;
END;
$$;
```

### **Вызов**

```sql
CALL recalculate_standings();

SELECT * FROM kr1_Standings ORDER BY team_id;
```

---

# **2.3 Процедура `set_stadium_capacity`**

Создать процедуру set_stadium_capacity(), которая изменяет вместимость выбранного стадиона:
— процедура должна принимать stadium_id и новое значение вместимости;
— если стадион не найден — генерировать понятную ошибку.
— если переданное значение вместимости НЕ находится в диапазоне от 1000 до 12000 — генерировать понятную ошибку
Привести пример вызова процедуры.

## **Процедура**

```sql
CREATE OR REPLACE PROCEDURE set_stadium_capacity(
    p_stadium_id INTEGER,
    p_capacity INTEGER
)
LANGUAGE plpgsql
AS $$
BEGIN
    IF p_capacity < 1000 OR p_capacity > 12000 THEN
        RAISE EXCEPTION
            'Вместимость должна быть в диапазоне от 1000 до 12000';
    END IF;

    IF NOT EXISTS (
        SELECT 1 FROM kr1_Stadiums WHERE stadium_id = p_stadium_id
    ) THEN
        RAISE EXCEPTION
            'Стадион с stadium_id % не найден', p_stadium_id;
    END IF;

    UPDATE kr1_Stadiums
    SET capacity = p_capacity
    WHERE stadium_id = p_stadium_id;
END;
$$;
```

## **Примеры вызова**

✅ Успех:

```sql
CALL set_stadium_capacity(1, 60000);
```

❌ Ошибка (неверный диапазон):

```sql
CALL set_stadium_capacity(1, 500);
```

Результат:

```
ERROR:  Вместимость должна быть в диапазоне от 1000 до 12000
```

❌ Ошибка (стадион не найден):

```sql
CALL set_stadium_capacity(999, 5000);
```

---

## **2.4 Процедура reset_team_standing**

Реализовать процедуру reset_team_standing, которая:
— получает на вход team_id;
— обнуляет все числовые показатели для указанной команды в таблице Standings;
— если запись для team_id отсутствует, генерирует ошибку.
Привести пример вызова процедуры.

```sql
CREATE OR REPLACE PROCEDURE reset_team_standing(p_team_id INTEGER)
LANGUAGE plpgsql
AS $$
BEGIN
    IF NOT EXISTS (
        SELECT 1 FROM kr1_Standings WHERE team_id = p_team_id
    ) THEN
        RAISE EXCEPTION 'Запись для team_id % не существует', p_team_id;
    END IF;

    UPDATE kr1_Standings
    SET games_played = 0,
        wins = 0,
        draws = 0,
        losses = 0,
        goals_for = 0,
        goals_against = 0,
        points = 0
    WHERE team_id = p_team_id;
END;
$$;
```

### **Вызов**

```sql
CALL reset_team_standing(1);

SELECT
    t.name,
    s.games_played,
    s.wins,
    s.draws,
    s.losses,
    s.goals_for,
    s.goals_against,
    s.points
FROM kr1_Standings s
JOIN kr1_Teams t ON t.team_id = s.team_id
WHERE s.team_id = 1;
```

---

## **2.5 delete_old_scheduled_matches**

Реализовать процедуру хранимую процедуру delete_old_scheduled_matches(), которая:
— принимает на вход p_before_date;
— удаляет из таблицы Matches все матчи со статусом 'scheduled' и датой match_date < p_before_date.
Привести пример вызова процедуры.

```sql
CREATE OR REPLACE PROCEDURE delete_old_scheduled_matches(p_before_date TIMESTAMP)
LANGUAGE plpgsql
AS $$
BEGIN
    DELETE FROM kr1_Matches
    WHERE status = 'scheduled'
      AND match_date < p_before_date;
END;
$$;
```

### **Вызов**

```sql
CALL delete_old_scheduled_matches('2024-01-01');

SELECT
    match_id,
    match_date,
    status
FROM kr1_Matches
WHERE status = 'scheduled'
ORDER BY match_date;
```

---

## **3.1 Контроль целостности Matches**

Реализовать триггер, который контролирует целостность данных о матчах в таблице Matches в зависимости от выполняемой операции:
3.1 Создать триггерную функцию, где используется переменная TG_OP для определения типа операции (INSERT, UPDATE, DELETE). Для операций INSERT и UPDATE обеспечить:
— статус матча может быть только 'scheduled' или 'completed';
— если статус 'scheduled', то счёт должен быть обнулён (0:0);
— если статус 'completed', то home_score и away_score не могут быть NULL и не могут быть отрицательными.
Для операции DELETE запретить удаление уже завершённых матчей (status = 'completed').
3.2 Создать триггер:
— BEFORE INSERT OR UPDATE OR DELETE на таблицу Matches,
— FOR EACH ROW,
— который вызывает функцию, созданную в пункте 3.1
3.3 Продемонстрировать работу триггера на примерах DML-операций, которые как успешно выполняются, так и корректно прерываются триггером (два запроса).

### **Функция**

```sql
CREATE OR REPLACE FUNCTION kr1_matches_integrity_control()
RETURNS trigger AS $$
BEGIN
    IF TG_OP IN ('INSERT', 'UPDATE') THEN
        IF NEW.status NOT IN ('scheduled', 'completed') THEN
            RAISE EXCEPTION 'Недопустимый статус матча';
        END IF;

        IF NEW.status = 'scheduled' THEN
            IF NEW.home_score <> 0 OR NEW.away_score <> 0 THEN
                RAISE EXCEPTION 'Для scheduled счёт должен быть 0:0';
            END IF;
        END IF;

        IF NEW.status = 'completed' THEN
            IF NEW.home_score IS NULL OR NEW.away_score IS NULL
               OR NEW.home_score < 0 OR NEW.away_score < 0 THEN
                RAISE EXCEPTION 'Для completed счёт обязателен и >= 0';
            END IF;
        END IF;
    END IF;

    IF TG_OP = 'DELETE' THEN
        IF OLD.status = 'completed' THEN
            RAISE EXCEPTION 'Нельзя удалять завершённые матчи';
        END IF;
        RETURN OLD;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

### **Триггер**

```sql
CREATE TRIGGER trg_matches_integrity
BEFORE INSERT OR UPDATE OR DELETE ON kr1_Matches
FOR EACH ROW
EXECUTE FUNCTION kr1_matches_integrity_control();
```

### **Демонстрация**

❌ Ошибка:

```sql
DELETE FROM kr1_Matches WHERE match_id = 1;
```

✅ Успех:

```sql
INSERT INTO kr1_Matches (home_team_id, away_team_id, stadium_id, match_date, status)
VALUES (1, 2, 1, CURRENT_TIMESTAMP + INTERVAL '2 days', 'scheduled');
```

---

## **3.2 Контроль Stadiums**

Создать триггер, который контролирует операции над Stadiums:
3.1 Создать триггерную функцию, которая:
— при DELETE запрещает удалять стадион, если он упоминается в таблице Matches;
— при UPDATE выводит уведомление (NOTICE) об изменении вместимости;
— использовать TG_OP для определения операции.
3.2 Создать триггер:
— BEFORE UPDATE OR DELETE на таблицу Stadiums,
— который вызывает функцию, созданную в пункте 3.1
3.3 Продемонстрировать работу триггера на примерах DML-операций, которые как успешно выполняются, так и корректно прерываются триггером (два запроса).

### **Функция**

```sql
CREATE OR REPLACE FUNCTION kr1_stadiums_control()
RETURNS trigger AS $$
BEGIN
    IF TG_OP = 'DELETE' THEN
        IF EXISTS (
            SELECT 1 FROM kr1_Matches
            WHERE stadium_id = OLD.stadium_id
        ) THEN
            RAISE EXCEPTION 'Нельзя удалить стадион, используемый в матчах';
        END IF;
        RETURN OLD;
    END IF;

    IF TG_OP = 'UPDATE' THEN
        IF NEW.capacity <> OLD.capacity THEN
            RAISE NOTICE
                'Вместимость стадиона изменена: % -> %',
                OLD.capacity, NEW.capacity;
        END IF;
        RETURN NEW;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

### **Триггер**

```sql
CREATE TRIGGER trg_stadiums_control
BEFORE UPDATE OR DELETE ON kr1_Stadiums
FOR EACH ROW
EXECUTE FUNCTION kr1_stadiums_control();
```

### **Демонстрация**

❌ Ошибка:

```sql
DELETE FROM kr1_Stadiums WHERE stadium_id = 1;
```

✅ Уведомление:

```sql
UPDATE kr1_Stadiums
SET capacity = capacity + 1000
WHERE stadium_id = 1;
```

---

# **3.5 Защита завершённых матчей от изменений**

Реализовать триггер, который защищает завершённые матчи от изменений:
3.1 Создать триггерную функцию, где используется переменная TG_OP для определения типа операции (INSERT, UPDATE, DELETE). Для операции UPDATE обеспечить:
— запрещать изменение статуса с 'completed' на любой другой.
Для операции DELETE запретить:
— удаление записей, у которых status = 'completed'.
3.2 Создать триггер:
— BEFORE UPDATE OR DELETE на таблицу Matches,
— который вызывает функцию, созданную в пункте 3.1
3.3 Продемонстрировать работу триггера на примерах DML-операций, которые как успешно выполняются, так и корректно прерываются триггером (два запроса).

## **Триггерная функция**

```sql
CREATE OR REPLACE FUNCTION kr1_protect_completed_matches()
RETURNS trigger AS $$
BEGIN
    IF TG_OP = 'UPDATE' THEN
        IF OLD.status = 'completed'
           AND NEW.status <> 'completed' THEN
            RAISE EXCEPTION
                'Запрещено изменять статус завершённого матча';
        END IF;
    END IF;

    IF TG_OP = 'DELETE' THEN
        IF OLD.status = 'completed' THEN
            RAISE EXCEPTION
                'Запрещено удалять завершённые матчи';
        END IF;
        RETURN OLD;
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## **Триггер**

```sql
CREATE TRIGGER trg_protect_completed_matches
BEFORE UPDATE OR DELETE ON kr1_Matches
FOR EACH ROW
EXECUTE FUNCTION kr1_protect_completed_matches();
```

---

## **Демонстрация**

❌ Ошибка (UPDATE):

```sql
UPDATE kr1_Matches
SET status = 'scheduled'
WHERE match_id = 1;
```

Результат:

```
ERROR:  Запрещено изменять статус завершённого матча
```

❌ Ошибка (DELETE):

```sql
DELETE FROM kr1_Matches
WHERE match_id = 1;
```

Результат:

```
ERROR:  Запрещено удалять завершённые матчи
```

✅ Успех:

```sql
UPDATE kr1_Matches
SET match_date = match_date + INTERVAL '1 day'
WHERE status = 'scheduled';
```
