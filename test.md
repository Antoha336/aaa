# 1
```
CREATE OR REPLACE FUNCTION auto_calc_points()
RETURNS TRIGGER AS $$
BEGIN
    -- Проверяем, что значения не отрицательные
    IF NEW.wins < 0 OR NEW.draws < 0 OR NEW.losses < 0 OR 
       NEW.goals_for < 0 OR NEW.goals_against < 0 THEN
        RAISE EXCEPTION 'Значения не могут быть отрицательными';
    END IF;
    
    -- Проверяем, что games_played = wins + draws + losses
    IF NEW.games_played != NEW.wins + NEW.draws + NEW.losses THEN
        RAISE EXCEPTION 'games_played должен равняться wins + draws + losses';
    END IF;
    
    -- Автоматически рассчитываем очки
    NEW.points = NEW.wins * 3 + NEW.draws;
    
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

-- Создаем новый триггер
```
CREATE TRIGGER standings_points_trigger
BEFORE INSERT OR UPDATE ON kr2_Standings
FOR EACH ROW
EXECUTE FUNCTION auto_calc_points();
```

-- 1. Успешный запрос (вставка корректных данных)
```
INSERT INTO kr2_Standings (team_id, games_played, wins, draws, losses, goals_for, goals_against) 
VALUES (100, 10, 5, 3, 2, 20, 15);
```

-- Проверяем: points должен быть 5*3 + 3 = 18
```
SELECT team_id, wins, draws, points FROM kr2_Standings WHERE team_id = 100;
```

-- 2. Запрос с ошибкой (триггер прерывает операцию)
```
INSERT INTO kr2_Standings (team_id, games_played, wins, draws, losses, goals_for, goals_against) 
VALUES (101, 10, -1, 3, 2, 20, 15);  -- wins = -1 
```
(недопустимо)

-- Ожидаемая ошибка: "Значения не могут быть отрицательными"

-- Пример 1: Обновление существующей записи
```
UPDATE kr2_Standings 
SET wins = 7, draws = 2 
WHERE team_id = 100;
```
-- Проверяем: points должен стать 7*3 + 2 = 23

-- Пример 2: Еще одна ошибка (несоответствие games_played)
```
INSERT INTO kr2_Standings (team_id, games_played, wins, draws, losses, goals_for, goals_against) 
VALUES (102, 10, 5, 3, 3, 20, 15);  -- 5+3+3=11, но games_played=10
``` 
-- Ожидаемая ошибка: "games_played должен равняться wins + draws + losses"


# 2
```
CREATE OR REPLACE PROCEDURE delete_old_scheduled_matches(
    p_before_date TIMESTAMP
)
LANGUAGE plpgsql
AS $$
DECLARE
    deleted_count INTEGER;
BEGIN
    -- Удаляем матчи
    DELETE FROM kr2_Matches
    WHERE status = 'scheduled'
      AND match_date < p_before_date;
    
    -- Получаем количество удаленных записей
    GET DIAGNOSTICS deleted_count = ROW_COUNT;
    
    -- Выводим информацию
    RAISE NOTICE 'Удалено % запланированных матчей до %', deleted_count, p_before_date;
END;
$$;
```

```
-- Вызов процедуры для удаления матчей, запланированных до 1 января 2024 года
CALL delete_old_scheduled_matches('2024-01-01 00:00:00');
```

```
-- Добавляем тестовые данные
INSERT INTO kr2_Matches (home_team_id, away_team_id, stadium_id, match_date, status) VALUES 
(1, 2, 1, '2023-12-01 20:00:00', 'scheduled'),
(3, 4, 3, '2023-12-15 18:00:00', 'scheduled'),
(5, 6, 5, '2024-04-10 19:00:00', 'scheduled'),
(2, 4, 2, '2023-11-20 20:00:00', 'completed'),
(1, 3, 1, '2024-05-01 20:00:00', 'scheduled');

-- Проверяем данные до удаления
SELECT 'ДО удаления:' as info, match_id, match_date, status FROM kr2_Matches ORDER BY match_date;

-- Вызываем процедуру
CALL delete_old_scheduled_matches('2024-01-01 00:00:00');

-- Проверяем данные после удаления
SELECT 'ПОСЛЕ удаления:' as info, match_id, match_date, status FROM kr2_Matches ORDER BY match_date;
```

# 3
```
CREATE OR REPLACE PROCEDURE update_team_goals_cursor(
    p_team_id INTEGER
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_team_name VARCHAR(40);
    v_goals_for INTEGER := 0;
    v_goals_against INTEGER := 0;
    v_home_score INTEGER;
    v_away_score INTEGER;
    v_is_home BOOLEAN;
    
    -- Курсор для завершенных матчей команды
    cur_matches CURSOR FOR
        SELECT 
            home_score, 
            away_score, 
            home_team_id = p_team_id as is_home
        FROM kr2_Matches
        WHERE status = 'completed'
          AND (home_team_id = p_team_id OR away_team_id = p_team_id);
BEGIN
    -- Проверяем, существует ли команда
    SELECT name INTO v_team_name 
    FROM kr2_Teams 
    WHERE team_id = p_team_id;
    
    IF NOT FOUND THEN
        RAISE EXCEPTION 'Команда с ID % не найдена', p_team_id;
    END IF;
    
    -- Открываем курсор и обрабатываем записи
    OPEN cur_matches;
    
    LOOP
        FETCH cur_matches INTO v_home_score, v_away_score, v_is_home;
        EXIT WHEN NOT FOUND;
        
        IF v_is_home THEN
            v_goals_for := v_goals_for + COALESCE(v_home_score, 0);
            v_goals_against := v_goals_against + COALESCE(v_away_score, 0);
        ELSE
            v_goals_for := v_goals_for + COALESCE(v_away_score, 0);
            v_goals_against := v_goals_against + COALESCE(v_home_score, 0);
        END IF;
    END LOOP;
    
    CLOSE cur_matches;
    
    -- Выводим результат
    RAISE NOTICE 'ID команды: %', p_team_id;
    RAISE NOTICE 'Название команды: %', v_team_name;
    RAISE NOTICE 'Забитые голы: %', v_goals_for;
    RAISE NOTICE 'Пропущенные голы: %', v_goals_against;
END;
$$;
```

```
-- Вызов процедуры для команды с ID = 1 (FC Barcelona)
CALL update_team_goals_cursor(1);
```

```

-- Добавим тестовые завершенные матчи для команды с ID = 1
INSERT INTO kr2_Matches (home_team_id, away_team_id, stadium_id, match_date, home_score, away_score, status) VALUES 
(1, 2, 1, '2024-01-15 20:00:00', 3, 1, 'completed'),  -- Домашний матч
(2, 1, 2, '2024-02-10 18:00:00', 2, 2, 'completed'),  -- Гостевой матч
(1, 3, 1, '2024-03-05 19:00:00', 1, 0, 'completed');  -- Домашний матч

-- Вызываем процедуру
CALL update_team_goals_cursor(1);

-- Пример вызова с несуществующей командой (будет ошибка)
CALL update_team_goals_cursor(999);
```
