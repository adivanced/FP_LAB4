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
