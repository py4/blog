---
title: "روز دوم - یک سیستم چت خیلی خیلی ساده"
draft: false
---
چالش اسفند: پیام‌رسان peer-to-peer - روز دوم - یک چت سیستم خیلی خیلی ساده!
===
امروز دومین روز از [این چالش](/blog/p2p) که جزو [پروژه‌ی ایکس من](/blog/x-project/) هستش بود. امروز [اون کتابه](http://erlang.org/download/getting_started-5.4.pdf)رو تموم کردم. الان یه sense اولیه‌ای نسبت به Erlang گرفتم هر چند که چیزهای خیلی خیلی خیلی زیادی هست که هنوز نمی‌دونم. امروز سعی کردم بدون خوندن اون کد چت ی که داخل کتاب زده، یه chat system ساده بنویسم. تلاش جالبی بود. در درجه‌ی اول مشکلم با سینتکس بود که به شدت من رو کند کرده بود چرا که به سینتکس‌ش عادت نداشتم. این هم در شرایطی که می‌شه گفت من تقریبا هیچ‌وقت functional کد نزدم. در درجه‌ی بعدی مشکل این بود که تو ذهنم procedural فکر می‌کردم و موقع پیاده‌سازی‌ش تو زبان functional دچار مشکل می‌شدم. به خصوص اینکه هی فراموش می‌کردم برای اینکه یه جوری این مقدار رو به عنوان storage داشته باشیم (قبلا مثلا instance member یه کلاسی بود تو زبان‌های دیگه)، باید به عنوان آرگومان‌های تابع بیاریمش. مشکل بعدی این بود که هی موقع کد زدن داشتم فکر می‌کردم internal implementation این به چه شکلی هستش که الان مثلا این spawnش چطوری پیاده‌سازی شده و این‌ها. شاید بهتر باشه حالا که یه sense اولیه‌ای راجع به زبانش گرفتم، بشینم یه داکیومنت بخونم راجع به design principleهایی که داخل پیاده‌سازی erlang به کار رفته. چون الانش خیلی اذیتم که نمیدونم این کدی که دارم می‌زنم دقیقا به چه شکل داره پیاده‌سازی می‌شه.

این فعلا اولین کدی هستش که نوشتم (فعلا فقط سعی کردم کار کنه و هیچ تمیزکاری و optimizationی رو در نظر نگرفتم):

{{< highlight erlang >}}
-module(messaging).
-export([start_server/0]).
-export([login/2]).
-export([logout/1]).
-export([client/0]).
-export([start_client/0]).
-export([send_message/3]).

start_server() -> 
    register(messaging_server, self()),
    server([]).

server_login(From, Username, Clients) ->
    case lists:keymember(From, 1, Clients) of
        true ->
            From ! {error, "Already logged in"},
            Clients;
        false -> 
            case lists:keymember(Username, 2, Clients) of
                    true ->
                        From ! {error, "Username already exists"},
                        Clients;
                    false ->
                        From ! {logged_in},
                        [{From, Username} | Clients]
            end
    end.
    
server_logout(From, Clients) ->
    case lists:keyfind(From, 1, Clients) of
        {From, Username} ->
            From ! {logged_out},
            lists:delete({From, Username}, Clients);
        false ->
            From ! {error, "You are not logged in yet!"},
            Clients
    end.

ensure_login(From, Clients) ->
    case lists:keyfind(From, 1, Clients) of
        false -> false;
        {From, Username} -> Username
    end.

server_send_message(From_PID, To_Username, Message, Clients) ->
    case ensure_login(From_PID, Clients) of
        false -> From_PID ! {error, "You are not logged in!"};
        From_Username ->
            case lists:keyfind(To_Username, 2, Clients) of
                false -> From_PID ! {error, "Target user not found!"};
                {To_PID, To_Username} ->
                    To_PID ! {message_received, From_Username, Message},
                    From_PID ! {message_sent}
            end
    end.
                    

server(Clients) ->
    receive
        {login, From, Username} -> 
            server(server_login(From, Username, Clients));
        {logout, From} ->
            server(server_logout(From, Clients));
        {send_message, From_PID, To_Username, Message} ->
            server_send_message(From_PID, To_Username, Message, Clients),
            server(Clients)
    end.


start_client() -> 
    spawn(messaging, client, []).

client() ->
    receive
        Message ->
            io:fwrite("fuck~n",[]),
            case Message of 
                {message_received, From, ChatMessage} ->
                    io:fwrite("~p Sent you: ~p~n", [From, ChatMessage]);
                {message_sent} ->
                    ok;
                {logged_in} ->
                    ok;
                {logged_out} ->
                    ok;
                {error, ErrorMessage} ->
                    io:fwrite("~p~n", [ErrorMessage])
            end,
            client()
    end.
        

login(Client, Username) ->
    {messaging_server, 'server@localhost'} ! {login, Client, Username}.

logout(Client) ->
    {messaging_server, 'server@localhost'} ! {logout, Client}.

send_message(Client, Username, Message) ->
    {messaging_server, 'server@localhost'} ! {send_message, Client, Username, Message}.
{{< / highlight >}}

یه مقدار چون راجع به internal قضیه‌ی concurrency ابهام داشتم، یه چیزایی سرچ کردم. به [این hitchhiker guide](http://learnyousomeerlang.com/the-hitchhikers-guide-to-concurrency) رسیدم که چیز خوبی بود و ظاهرا یه کتاب کامله که می‌تونم انلاین بخونمش. بامزه توضیح داده بود. البته این رو روی گوشی پشت چراغ‌های قرمز تهران خوندم و باید فردا دقیق‌تر بخونمش.

این‌ها برای امروز بود. برای فردا کاری که باید بکنم اینه که failure handling به قضیه اضافه کنم. بعد احتمالا یه تلاش بزنم که همین رو p2p ش کنم و دیگه پای یه سرور وسط نباشه فقط بحث node discovery میمونه که حالا باید فکر کنم چیکارش کنم. بعدش احتمالا احتیاج دارم یه کتاب کامل‌تر از Erlang بخونم که OTP اینا رو توضیح داده باشه (هر چند که الان چیز خاصی راجع به Erlang OTP نمی‌دونمش). ولی خب در درجه‌ی اول بعد از اون failure handling احتمالا بشینم همین کتاب یارو hitchhiker guideه رو بخونم.


