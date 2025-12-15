CREATE OR REPLACE FUNCTION auto_calc_points()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Проверка на NULL
    IF NEW.wins IS NULL OR NEW.draws IS NULL THEN
        RAISE EXCEPTION 
            'Поля wins и draws не могут быть NULL';
    END IF;

    -- Проверка на отрицательные значения
    IF NEW.wins < 0 OR NEW.draws < 0 THEN
        RAISE EXCEPTION 
            'Поля wins и draws не могут быть отрицательными (wins=%, draws=%)',
            NEW.wins, NEW.draws;
    END IF;

    -- Автоматический пересчёт очков
    NEW.points := NEW.wins * 3 + NEW.draws;

    RETURN NEW;
END;
$$;

CREATE OR REPLACE PROCEDURE delete_old_scheduled_matches(
    IN p_before_date TIMESTAMP
)
LANGUAGE plpgsql
AS $$
BEGIN
    IF p_before_date IS NULL THEN
        RAISE EXCEPTION
            'Параметр p_before_date не может быть NULL';
    END IF;

    DELETE FROM kr2_Matches
    WHERE status = 'scheduled'
      AND match_date < p_before_date;
END;
$$;

CREATE OR REPLACE PROCEDURE update_team_goals_cursor(
    IN p_team_id INTEGER
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_team_name      VARCHAR(40);
    v_goals_for      INTEGER := 0;
    v_goals_against  INTEGER := 0;

    rec RECORD;

    cur_matches CURSOR FOR
        SELECT
            home_team_id,
            away_team_id,
            home_score,
            away_score
        FROM kr2_Matches
        WHERE status = 'completed'
          AND (home_team_id = p_team_id OR away_team_id = p_team_id);
BEGIN
    -- Проверка существования команды
    SELECT name
    INTO v_team_name
    FROM kr2_Teams
    WHERE team_id = p_team_id;

    IF NOT FOUND THEN
        RAISE EXCEPTION
            'Команда с team_id = % не найдена',
            p_team_id;
    END IF;

    -- Обход матчей курсором
    OPEN cur_matches;

    LOOP
        FETCH cur_matches INTO rec;
        EXIT WHEN NOT FOUND;

        IF rec.home_team_id = p_team_id THEN
            v_goals_for     := v_goals_for + rec.home_score;
            v_goals_against := v_goals_against + rec.away_score;
        ELSE
            v_goals_for     := v_goals_for + rec.away_score;
            v_goals_against := v_goals_against + rec.home_score;
        END IF;
    END LOOP;

    CLOSE cur_matches;

    -- Вывод результата
    RAISE NOTICE
        'Team ID: %, Team Name: %, Goals For: %, Goals Against: %',
        p_team_id, v_team_name, v_goals_for, v_goals_against;
END;
$$;
