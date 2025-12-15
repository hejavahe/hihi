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
```

---

## **2.4 Процедура reset_team_standing**

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
```

---

## **2.5 delete_old_scheduled_matches**

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
```

---

## **3.1 Контроль целостности Matches**

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
