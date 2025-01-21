# FP_LAB4

---

* Студенты: `Панин Иван Михайлович` `Голиков Денис ?`
* Группа: `P34082` `P34102`
* ИСУ: `369405`  `?`
* Язык: `Erlang`

--- 

# Игра на Erlang с клиент-серверной составляющей
В ходе работы было принято решение реализовать сервер для браузерной реализации игры 2048 с браузерным клиентом. Клиент написан на JavaScript. Серверная часть написана на Erlang с использованием cowboy HTTP server, библиотеки mochiweb для работы с HTTP серверами и библиотеки erlang-sqlite3 для взаимодействия с БД. Также в процессе разработки для логгирования использовалась библиотека Lager.

## Реализация:

### Описание программы:

#### Сервер:

- Точка входа в программу: [erl2048_app.erl](server/src/erl2048_app.erl)
- Работа с БД: [db.erl](server/src/db.erl)
- Игровая логика: [game.erl](server/src/game/game.erl)
- Функции для работы с игровым полем: [grid.erl](server/src/game/grid.erl)
- Функции для работы с клетками: [tile.erl](server/src/game/tile.erl)

### Ключевые элементы реализации:

**Запуск сервера и инициализация взаимодействия с БД**
````erlang
start(_Type, _Args) ->
    Dispatch = cowboy_router:compile([
        {'_', [
            {"/", cowboy_static, {file, "../client/index.html"}},
            {"/websocket", ws_handler, []},
            {"/static/[...]", cowboy_static, {dir, "../client/static"}}
        ]}
    ]),
    {ok, _} = db:start_link(),
    cowboy:start_http(http, 100, [{port, 8081}],
        [{env, [{dispatch, Dispatch}]}]).

start() ->
    application:ensure_all_started(erl2048).

stop(_State) ->
    {ok, _} = db:stop(),
    ok.
````
**Обработка данных, полученных через Websocket**
````erlang
websocket_handle({text, Msg}, Req, State) ->
    Message = mochijson2:decode(Msg, [{format, proplist}]),
    Action =  binary_to_list(proplists:get_value(<<"action">>, Message)),
    {NewState, Response} = case Action of
        "start" ->
            TmpState = game:init(State),
            {TmpState, TmpState};
        "move"  ->
            TmpState = game:move(list_to_atom(binary_to_list(proplists:get_value(<<"value">>, Message))), State),

            %Checking for won or lose
            {struct, JsonData} = TmpState,
            Over = proplists:get_value(over, JsonData),
            Won  = proplists:get_value(won, JsonData),
            KeepPlaying = proplists:get_value(keepPlaying, JsonData),

            Resp = if
                Over =:= true -> {struct, [{over, true}]};
                (Won =:= true) and (KeepPlaying =:= false) -> {struct, [{ask, true}]};
                true -> TmpState
            end,

            {TmpState, Resp};
        "newName" ->
            NewName = proplists:get_value(<<"value">>, Message),
            JsonData = element(2, State),

            User = proplists:get_value(user, JsonData),
            {struct,UserJsonData} = User,

            Id = proplists:get_value(id, UserJsonData),

            db:changeName(Id, NewName),

            TmpState = {struct, [
                    { user, { struct, [ { name, NewName },{ id, Id } ] } }
                    | proplists:delete(user, JsonData)
                ]},
            {
                TmpState,
                {struct, [{ user, { struct, [ { name, NewName },{ id, Id } ] } }]}
            };
        "keepPlaying" ->
            TmpState = {struct, [ { keepPlaying, true } | proplists:delete(keepPlaying, element(2, State)) ]},
            {TmpState, TmpState};
        _Else -> State
    end,
    {reply, {text, mochijson2:encode(Response)}, Req, NewState};
websocket_handle(_Data, Req, State) ->
    {ok, Req, State}.
````

**Движение клетки**
````erlang
moveTile(Cell, Vector, JsonData) ->

    Grid = proplists:get_value(grid, JsonData),
    Tile = grid:cellContent(Cell, Grid),

    case Tile =:= null of
        true -> JsonData;
        false ->
            { Farthest, Next } = findFarthestPosition(Cell, Vector, Grid),

            {struct, CurrJsonData} = Tile,
            CurrValue = proplists:get_value(value, CurrJsonData),

            NextTile = if
                Next =:= null -> null;
                true ->
                    grid:cellContent(Next, Grid)
            end,

            {NextValue, NextMerged} = if
                NextTile =:= null -> {null, null};
                true ->
                    NextJsonData = element(2, NextTile),
                    {proplists:get_value(value, NextJsonData), proplists:get_value(mergedFrom, NextJsonData)}
            end,

            if  CurrValue =:= NextValue,
                NextMerged =:= null
                ->
                    MergedValue = CurrValue * 2,
                    Merged = {
                        struct,
                        [
                            {value, MergedValue},
                            {mergedFrom, [Tile,NextTile]},
                            {previousPosition, null}
                        ]
                    },
                    NewGrid = grid:insertTile(Next, Merged, grid:removeTile(Cell, Grid)),

                    % Update the score
                    Score = proplists:get_value(score, JsonData) + MergedValue,

                    % The mighty 2048 tile
                    Won = if
                        MergedValue =:= 2048 -> true;
                        true -> false
                    end,

                    Removed = my_proplists:delete_several([score, won, grid], JsonData),

                    [
                        {grid,NewGrid},
                        {won,Won},
                        {score,Score} |
                        Removed
                    ];
                true ->
                    [
                        {
                            grid,
                            grid:moveTile(Cell, Farthest, proplists:get_value(grid, JsonData))
                        }
                        | proplists:delete(grid, JsonData)
                    ]
            end
    end.
````

**Движение поля**
````erlang
move(left, State) ->
    move(getVector(left), State);
move(right, State) -> 
    move(getVector(right), State);
move(up, State) -> 
    move(getVector(up), State);
move(down, State) -> 
    move(getVector(down), State);
move(Vector, State) ->
    {struct, JsonData} = State,

    case 
        proplists:get_value(over, JsonData) or (
            proplists:get_value(won, JsonData) and (not proplists:get_value(keepPlaying, JsonData))
        )
    of
        true -> State;
        _Else ->
            PreparedJsonData = updateBestScore(prepareTiles(JsonData)),

            { TraversalsX, TraversalsY } = buildTraversals(Vector),

            NewJsonData = process_travesals_y(
                TraversalsY,
                TraversalsX,
                Vector,
                PreparedJsonData
            ),

            NewGrid = proplists:get_value(grid, NewJsonData),
            Grid = proplists:get_value(grid, PreparedJsonData),

            if
                NewGrid =/= Grid -> %If changed - add new tile
                    
                    {struct, UserJsonData} = proplists:get_value(user, NewJsonData),

                    NewScore = proplists:get_value(score, NewJsonData),
                    Score = proplists:get_value(score, PreparedJsonData),

                    case NewScore > Score of true ->
                        db:insert(
                            proplists:get_value(score, NewJsonData),
                            proplists:get_value(id, UserJsonData)
                        );
                        _Else -> undefined
                    end,

                    Over = case movesAvailable(NewGrid) of
                        true -> false;
                        fale -> true % Game over!
                    end,
                    Removed = my_proplists:delete_several([grid, over], NewJsonData),
                    {struct,[{ grid, addRandomTile(NewGrid) }, { over, Over } | Removed ]};
                true -> %return state otherwise
                    {struct,PreparedJsonData}
            end
    end.
````

**Работа с БД**
````erlang
start_link() ->
    {ok, PID} = sqlite3:open(db, [{file, "db.sqlite3"}]),

    Tables = sqlite3:list_tables(db),

    case lists:member("scores", Tables) of false ->
        sqlite3:create_table(db, scores, [{id, integer, [{primary_key, [asc, autoincrement]}]}, {userid, integer}, {score, integer}])
    end,

    case lists:member("users", Tables) of false ->
        sqlite3:create_table(db, users, [{id, integer, [{primary_key, [asc, autoincrement]}]}, {name, text}])
    end,

    {ok, PID}.

stop() ->
    sqlite3:close(db).

select() ->
    Ret = sqlite3:sql_exec(db, "select users.name, scores.score from scores LEFT JOIN users ON (users.id = scores.userid) ORDER BY score desc;"),
    [{columns,_},{rows,Rows}] = Ret,
    formatScores(Rows).

insert(Score, Player) ->
    [{columns,_},{rows,Rows}] = sqlite3:sql_exec(db, "SELECT score FROM scores WHERE userid = ?", [{1,Player}]),
    DBScore = if
        length(Rows) > 0  -> element(1,hd(Rows));
        true -> 0
    end,

    if Score > DBScore ->
        sqlite3:delete(db, scores, {userid, Player}),
        sqlite3:write(db, scores, [{userid, Player}, {score, Score}]),
        sqlite3:sql_exec(db, "DELETE FROM scores WHERE id IN (SELECT id FROM scores ORDER BY score desc LIMIT 1 OFFSET 10)");
        true -> undefined
    end.

formatScores([]) ->
    [];
formatScores([{Name, Score} | Rows]) ->
    [{struct, [{name, Name},{score, Score}]} | formatScores(Rows)].

createUser(UserName) ->
    sqlite3:write(db, users, [{name, UserName}]).

changeName(Id, NewName) ->
    sqlite3:update(db, users, {id, Id}, [{name, NewName}]).
````

## Заключение
В ходе выполнения данной работы, нами были получены навыки создания веб-серверов на языке Erlang и работы с БД на языке Erlang.
