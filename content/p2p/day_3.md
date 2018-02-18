---
title: "روز سوم : افزایش سرعت کد زدن در ارلنگ"
draft: false
---
امروز سومین روز من در [چالش اسفند ماه](/blog/p2p) بود. در [پروژه‌ی ایکس](/blog/x-project)ی که برای خودم تعریف کردم، من هر ماه یک چلنجی رو برای خودم انتخاب می‌کنم. 

خب امروز تازه ساعت ۱۲ شب کار کردن روی پروژه رو شروع کردم و الان که دارم این پست رو می‌نویسم ساعت ۵ صبح هستش!
در ابتدا نشستم یه مقدار از این کتاب [Learn You Some Erlang](http://learnyousomeerlang.com) رو خوندم. خیلی بامزه و خوب توضیح داده و از روی [یک کتاب Haskell](http://learnyouahaskell.com/) اسکی رفته سبک نوشتن‌ش رو. چون از روی اون کتاب قبلی‌ه یه سری چیزا خونده بودم، خیلی مطالبش تکراری بود و تقریبا می‌شه گفت صرفا وقتم رو تلف کردم :دی البته یه سری چیزای ریز مثلا استفاده از `record`ها رو یاد گرفتم، کانسپت [Tail Recursion](https://en.wikipedia.org/wiki/Tail_call) ^[
در زبان‌های برنامه‌نویسی تابعی، به دلیل اینکه همه چیز با فراخونی تابع اتفاق میفته و خیلی چیزها به صورت بازگشتی هستند، باید به این موضوع توجه کرد چرا که performance رو خیلی تحت تاثیر قرار می‌گیره. یک تابع recursive عادی همینطوری stack trace رو پر می‌کنه اما یک تابع tail recursive مثلا یه حلقه‌ی loop عمل می‌کنه و یه حافظه‌ی ثابت استفاده میکنه. باید حواستون باشه که توابع بازگشتی‌تون رو تا جاییکه می‌شه tail recursive بنویسید
] برام یادآوری شد و اینکه با List Comprehensions که خیلی چیز جالبی بود آشنا شدم. یه سینتکسی هست که می‌شه باهاش مثلا یه مجموعه رو ساخت به این شکل :
{{< highlight erlang >}}
[X+Y || X <- [1,2, 3], Y <- [4, 5, 6], X rem 2 =:= 1, Y rem 2 =:= 0 ].
{{</highlight>}}

این چیزی که نوشتم میاد Xهای فرد رو با Yهای زوج جمع می‌کنه!

بعد از اینکه یکی دو ساعت وقتم رو با خوندن کتاب‌ه تلف کردم (به خاطر مطالب تقریبا تکراری)، نشستم یه سری تمیزکاری بکنم روی کدی که دیروز زده بود. اولا اینکه می‌خواستم کد کلاینت و سرور رو از هم جدا کنم که ماژولار تر بشه. ثانیا می‌خواستم اطلاعات کلاینت‌ها رو به صورت `record` نگه دارم تو سرور. ثالثا می‌خواستم یه سری failure اینا هندل کنم که دیگه به این آخری نرسیدم.

الان کد آخری که بهش رسیدم این هستش:

این کد سرور هستش:

{{< highlight erlang >}}
-record(server_store,
        {clients = []}).

-record(client,
        {pid, username}).

login(PID, Username, Store) ->
    Clients = Store#server_store.clients,
    io:fwrite("Looking for ~p in ~p", [PID, Clients]),
    case lists:keymember(PID, #client.pid, Clients) of
        true ->
            PID ! {error, "Already logged in"},
            Store;
        false ->
            case lists:keymember(Username, #client.username, Clients) of
                true ->
                    PID ! {error, "Username already exists"},
                    Store;
                false ->
                    PID ! {logged_in},
                    Clients = Store#server_store.clients,
                    NewClients = [#client{pid = PID, username = Username} | Clients],
                    Store#server_store{clients = NewClients}
            end
    end.

logout(PID, Store) ->
    Clients = Store#server_store.clients,
    case lists:keyfind(PID, #client.pid, Clients) of
        false ->
            PID ! {error, "You are not logged in yet!"},
            Store;
        Client ->
            Client#client.pid ! {logged_out},
            Store#server_store{clients = lists:delete(Client, Clients)}
    end.

ensure_login(PID, Store) ->
    Clients = Store#server_store.clients,
    case lists:keyfind(PID, #client.pid, Clients) of
        false -> false;
        Client -> Client#client.username
    end.

send_message(From_PID, To_Username, Message, Store) ->
    Clients = Store#server_store.clients,
    case ensure_login(From_PID, Store) of
        false -> From_PID ! {error, "You are not logged in!"};
        From_Username ->
            case lists:keyfind(To_Username, #client.username, Clients) of
                false -> From_PID ! {error, "Target user not found!"};
                ToClient ->
                    ToClient#client.pid ! {message_received, From_Username, Message},
                    From_PID ! {message_sent}
            end
    end.

start_server() ->
    register(server, self()),
    server(#server_store{clients = []}).


server(Store) ->
    receive
        {login, PID, Username} ->
            io:fwrite("Shit!~n", []),
            server(login(PID, Username, Store));
        {logout, PID} ->
            server(logout(PID, Store));
        {send_message, PID, To, Message} ->
            send_message(PID, To, Message, Store),
            server(Store)
    end.
{{</highlight>}}


و این هم کد کلاینت هستش :

{{< highlight erlang >}}
-module(client).
-export([start_client/2]).
-export([start_receiver/1]).

-record(shell,
        {login, logout, send_message}).

start_receiver(Shell_PID) ->
    receive
        Message ->
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
                    Shell_PID ! ErrorMessage
            end,
            io:fwrite("[Client] received: ~p~n", [Message]),
            start_receiver(Shell_PID)
    end.

get_shell(ServerInfo, Receiver_PID) ->
    #shell{
       login = fun(Username) ->
                ServerInfo ! {login, Receiver_PID, Username} end,

       logout = fun() ->
                ServerInfo ! {logout, Receiver_PID} end,

       send_message = fun(Username, Message) ->
                ServerInfo ! {send_message, Receiver_PID, Username, Message} end
      }.

start_client(Server_Registered_Name, Server_Node_Address) ->
    ServerInfo = {Server_Registered_Name, Server_Node_Address},
    Receiver_PID = spawn(client, start_receiver, [self()]),
    get_shell(ServerInfo, Receiver_PID).
{{</highlight>}}
تغییرات اساسی که کرده یکی استفاده از recordها بود که داستان‌های خودش رو داشت. یکی هم اینکه یه خلاقیتی زدم این وسط که شاید راه‌های بهتری هم داشته باشه. دفعه‌ی قبلی اطلاعات سرور در کلاینت hardcode شده بود. می‌خواستم به صورت ورودی بگیرمش داخل تابع `start_client`. مشکلی که وجود داشت این بود که برای اینکه داخل توابعی نظیر `login` اینا بهش دسترسی پیدا کنیم، باید به عنوان آرگومان می‌گرفتیمش و این باعث می‌شد داخل shell همیشه مجبور باشیم اطلاعات سرور رو هی بدیم. یه ایده‌ی خیلی خفنی که زدم :))) (حتی ممکنه ایده‌ی احمقانه‌ای باشه چرا که الان ساعت ۵ صبحه :دی) این بود که تابع ‍`start_client` یه رکورد به نام `shell` برگردونه که داخلش لیستی از توابع `login` و ... تعریف شده. حالا چطوری بدون آرگومان صداشون کنیم؟ ایده این بود که این توابع رو به صورت `Anonmyous` تعریف کنیم و از بیرون (ینی داخل تابع `start_client`) اطلاعات سرور رو بهش bind کنیم! که خب خودم بسی حال کردم با این حرکت :دی

مشکلاتی که خوردم به خاطر اینکه بعضا ارورهای Erlang برام نامفهوم بود طول می‌کشید تا حل‌شون کنم و متاسفانه ریسورس‌های رفع مشکلات Erlang تو اینترنت خیلی کم ه و بیشتر از اینکه Stackoverflow باشه تو mailing listشون ه:

۱. حواسم نبود که وقتی یه تابعی رو می‌خوای `spawn` کنی باید قبلش `export` شده باشه!

۲. حواسم نبود که Erlang یک زبان Dynamically Typed هستش و ماکسیمم کاری که برای آدم می‌کنه Pattern Matching هستش. بعد این کجا من رو اذیت کرد؟ وقتی که کد رو عوض کردم که از رکورد `Store` استفاده کنه، یه سری جاها رو فراموش کردم که `Store` پاس بدم و به جاش یه Tuple دیگه که تعداد المان‌هاش هم متفاوت بود پاس می‌دادم.
^[در واقع Erlang هر Tupleی رو از من قبول می‌کرد و به صورت run-time دچار مشکل می‌شد. اولش یه مقدار به تناقض رسیدم که مگه Erlang شعارش این نبود که ما کارمون reliability و availability و این‌ها هستش؟ پس چرا اینقدر راحت run-time ارور خورد؟ داخل همین کتاب Learn you some Erlang توضیح داده که Erlang به جای اینکه بیاد روی Static Type Checking مانور بده (که باعث می‌شه safety بره بالاتر)، روی failure recovery مانور داده. اصن فرض Erlang اینه که برنامه‌های مرتب کرش می‌کنیم و به جای اینکه بیایم جلوی کرش کردن‌شون رو بگیریم، بیایم جلوی منتشر شدن خطا رو بگیریم (از طریق پایین آوردن پراسس و بالا آوردن یه پراسس جدید). که خب کلا جالب بود فلسفه‌ش.
]

اینم از کارهای امروز. کارهایی که برای فردا بکنم یکی اینه که یه نگاه بزنم تو اینترنت ببینم Naming Conventionها تو ارلنگ چطوری هستش. یکی اینکه بشینم OTP رو از روی این کتاب‌ه بخونم ببینم چه فازیه و اینکه یه ذره هم راجع به پروتکل‌های P2P بخونم (اینا تسک‌هایی بود که امروز می‌خواستم انجامشون بدم که نرسیدم)

