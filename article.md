### Тестируем aiohttp с помощью простого чата
__Оглавление__
* *Введение*
* *Структура*
* *Роуты*
* *Handlers, Request and Response*
* *Настройки конфигурации*
* *Middlewares*
* *Базы данных*
* *Шаблоны*
* *Сессии, авторизация*
* *Статика*
* *WebSocket*
* *Выгрузка на Heroku*

---
__Введение__

Прошлой осенью мне удалось побывать на нескольких python митапах в Киеве. 
На одном из них выступал Николай Новик и рассказывал он о новом асинхронном 
фреймворке [aiohttp](http://aiohttp.readthedocs.org/en/stable/), 
работающем на библиотеке для асинхронных вызовов [asyncio](https://docs.python.org/3/library/asyncio.html) в 3 версии интерпретатора питона.
Данный фреймворк заинтересовал меня тем, что он создавался core python разработчиками, 
в основном Андреем Светловым из Киева и позиционировался как концепт python фреймворка для веба. 
Сейчас имеется огромное количество разных фреймворков, в каждом из которых своя философия, 
синтаксис и реализация общих для веба шаблонов. Я надеюсь, что со временем, все это разнообразие 
будет на одной основе - aiohttp.

__Структура__

Чтобы протестировать по максимуму все возможности aiohttp, я попытался разработать простой чат
на вебсокетах.Основой aiohttp является бесконечный loop, в котором крутятся handlers. 
Handler - так называемая сорутина, объект, который не блокирует ввод/вывод(I/O). 
Данный тип объектов появился в python 3.4 в библиотеке asyncio. Пока не произойдут все 
вычисления в данном объекте, он как бы засыпает, а в это время интерпретатор может обрабатывать 
другие объекты. Чтобы было понятно, наведу пример. Зачастую все задержки сервера происходят, 
когда он ожидает ответа от базы данных и пока этот ответ не придёт и не обработается, 
другие объекты ждут своей очереди. В данном случае другие объекты будут обрабатываться, 
пока не придёт ответ от базы. Но для реализации этого нужен асинхронный драйвер. 
На данный момент для aiohttp реализованы [ асинхронные драйвера и обёртки ]( https://github.com/aio-libs/ ) для большинства популярных баз данных ( [ postgresql ]( https://github.com/aio-libs/aiopg ), [ mysql ]( https://github.com/aio-libs/aiomysql ), [ redis ](https://github.com/aio-libs/aioredis)) 
Для mongodb есть [ Motor ]( http://motor.readthedocs.org/en/stable/ ), который я буду использовать в своём чате.

Точкой входа для моего чата служит файл [**app.py**](https://github.com/Crandel/aiohttp/blob/master/app.py). В нем создаётся объект app

```python
import asyncio
from aiohttp import web

loop = asyncio.get_event_loop()

app = web.Application(loop=loop, middlewares=[
    session_middleware(EncryptedCookieStorage(SECRET_KEY)),
    authorize,
    db_handler,
])
```
Как вы видите, при инициализации в app передаётся loop, а также список middleware, о котором я расскажу попозже.

__Роуты__

В отличии от flask на который aiohttp очень похож, роуты добавляются в уже инициализированное приложение app.
```python
app.router.add_route('GET', '/{name}', handler)
```
Вот кстати [ объяснение ] (http://asvetlov.blogspot.com/2014/10/flask_20.html) Светлова почему именно так реализовано.

Я вынес заполнение путей(route) в отдельный файл [routes.py](https://github.com/Crandel/aiohttp/blob/master/routes.py)
```python
from chat.views import ChatList, WebSocket
from auth.views import Login, SignIn, SignOut

routes = [
    ('GET', '/',        ChatList,  'main'),
    ('GET', '/ws',      WebSocket, 'chat'),
    ('*',   '/login',   Login,     'login'),
    ('*',   '/signin',  SignIn,    'signin'),
    ('*',   '/signout', SignOut,   'signout'),
]
```
Первый элемент - http метод, далее расположен url, третьим в кортеже идёт объект handler, 
и напоследок - имя пути, чтобы удобно было его вызывать в коде.

Далее я импортирую список routes в app.py и заполняю пути простым циклом пути в приложение
```python
from routes import routes

for route in routes:
        app.router.add_route(route[0], route[1], route[2], name=route[3])
```
Все просто и логично

__Handlers, Request and Response__

Я решил обработку запросов сделать по примеру Django фреймворка. В папке [auth](https://github.com/Crandel/aiohttp/tree/master/auth) находиться все, 
что касается пользователей, авторизации, обработка создания пользователя и его входа.
А в папке [chat](https://github.com/Crandel/aiohttp/tree/master/chat) находиться логика работы чата соответственно.
В aiohttp можно реализовать [handler](http://aiohttp.readthedocs.org/en/stable/web.html#handler) в качестве как функции, так и класса.
Я выбрал реализацию через класс
```python
class Login(web.View):

    async def get(self):
        session = await get_session(self.request)
        if session.get('user'):
            url = request.app.router['main'].url()
            raise web.HTTPFound(url)
        return b'Please enter login or email'
```
Про сессии я расскажу ниже, а все остальное думаю понятно и так. Хочу заметить,
что переадресация происходит либо возвратом(return) либо выбросом исключения в виде объекта
web.HTTPFound(), которому передаётся путь параметром.
Http методы в классе реализуются через асинхронные функции get, post и тд.
Есть некоторые особенности, если нужно работать с параметрами запроса.
```python
data = await self.request.post()
```

__Настройки конфигурации__

Все настройки я храню в файле [settings.py](https://github.com/Crandel/aiohttp/blob/master/settings.py).
Для хранения секретных данных я использую [envparse](https://github.com/rconradharris/envparse).
Данная утилита позволяет читать данные из переменных окружения а также парсить специальный файл, где эти переменные хранятся.
```python
if isfile('.env'):
    env.read_envfile('.env')
```
Во первых, мне это было необходимо для поднятия проекта на Heroku, а во вторых, это оказалось ещё и очень удобно,
я сначала использовал локальную базу, а потом тестировал на удалённой,
и переключение состояло из изменения всего одной строки в файле .env

__Middlewares__

При инициализации приложения можно задавать middleware. Я вынес их в отдельный [файл](https://github.com/Crandel/aiohttp/blob/master/middlewares.py).
Реализация стандартная - функция декоратор, в которой можно делать проверки или любые другие действия с запросом
*Пример проверки на авторизацию*
```python
async def authorize(app, handler):
    async def middleware(request):
        def check_path(path):
            result = True
            for r in ['/login', '/static/', '/signin', '/signout', '/_debugtoolbar/']:
                if path.startswith(r):
                    result = False
            return result

        session = await get_session(request)
        if session.get("user"):
            return await handler(request)
        elif check_path(request.path):
            url = request.app.router['login'].url()
            raise web.HTTPFound(url)
            return handler(request)
        else:
            return await handler(request)

    return middleware
```
Также я сделал middleware для подключения базы данных
```python
async def db_handler(app, handler):
    async def middleware(request):
        if request.path.startswith('/static/') or request.path.startswith('/_debugtoolbar'):
            response = await handler(request)
            return response

        request.db = app.db
        response = await handler(request)
        return response
    return middleware
```
Детали подключения ниже по тексту.

__Базы данных__

Для чата я использовал Mongodb и асинхронный драйвер Motor.
Подключение к базе происходит при инициализации приложения
```python
app.client = ma.AsyncIOMotorClient(MONGO_HOST)
app.db = app.client[MONGO_DB_NAME]
```
А закрытие соединения происходит в специальной функции shutdown
```python
async def shutdown(server, app, handler):

    server.close()
    await server.wait_closed()
    app.client.close()  # database connection close
    await app.shutdown()
    await handler.finish_connections(10.0)
    await app.cleanup()
```
Хочу заметить, что в случае асинхронного сервера нужно корректно завершить все параллельные задачи

Немного подробнее про создание event loop
```python
loop = asyncio.get_event_loop()
serv_generator, handler, app = loop.run_until_complete(init(loop))
serv = loop.run_until_complete(serv_generator)
print('start server', serv.sockets[0].getsockname())
try:
    loop.run_forever()
except KeyboardInterrupt:
    print(' Stop server begin')
finally:
    loop.run_until_complete(shutdown(serv, app, handler))
    loop.close()
print('Stop server end')
```
Сама петля создаётся из asyncio
```python
serv_generator, handler, app = loop.run_until_complete(init(loop))
```
Метод run_until_complete добавляет корутины в петлю. В данном случае он добавляет функцию [инициализации](https://github.com/Crandel/aiohttp/blob/master/app.py#L31) приложения.
```python
try:
    loop.run_forever()
except KeyboardInterrupt:
    print(' Stop server begin')
finally:
    loop.run_until_complete(shutdown(serv, app, handler))
    loop.close()
```
Собственно сама реализация бесконечного цикла, который прерывается в случае исключения. Перед закрытием вызывается функция shutdown, которая тушит все соединения и корректно останавливает сервер.

Теперь нам надо разобраться, как делать запросы, извлекать и изменять данные
```python
class Message():

    def __init__(self, db, **kwargs):
        self.collection = db[MESSAGE_COLLECTION]

    async def save(self, user, msg, **kw):
        result = await self.collection.insert({'user': user, 'msg': msg, 'time': datetime.now()})
        return result

    async def get_messages(self):
        messages = self.collection.find().sort([('time', 1)])
        return await messages.to_list(length=None)
```
Хотя у меня не задействована ОРМ, запросы к базе мне удобнее делать в отдельных классах.
В папке chat я создал файл [models.py](https://github.com/Crandel/aiohttp/blob/master/chat/models.py), где находится класс Message.
В методе get_messages создаётся запрос, который достаёт все сохранённые сообщения, отсортированные по времени.
В методе save создаётся запрос на сохранение сообщения в базу.

__Шаблоны__
Для aiohttp написано несколько асинхронных оберток для популярных шаблонизаторов, в частности [aiohttp_jinja2](https://github.com/aio-libs/aiohttp_jinja2) и [aiohttp_mako](https://github.com/aio-libs/aiohttp_mako). Для своего чата я использую jinja2
```python
aiohttp_jinja2.setup(app, loader=jinja2.FileSystemLoader('templates'))
```
Вот так поддержка шаблонов инициализируется в приложении.
FileSystemLoader('templates') указывает jinja2 что наши шаблоны лежать в папке [templates](https://github.com/Crandel/aiohttp/tree/master/templates)
```python
class ChatList(web.View):
    @aiohttp_jinja2.template('chat/index.html')
    async def get(self):
        message = Message(self.request.db)
        messages = await message.get_messages()
        return {'messages': messages}
```
Через декоратор мы указываем, какой [шаблон](https://github.com/Crandel/aiohttp/blob/master/templates/chat/index.html) будем использовать в вьюхе, а для заполнения контекста, возвращаем словарь с переменными, с которыми потом работаем в шаблоне

__Сессии, авторизация__
Для работы с сессиями есть библиотека [aiohttp_session](https://github.com/aio-libs/aiohttp_session).
Есть возможность хранить сессии в редисе или в куках в зашифрованном виде, используя cryptography
Способ хранения указывается ещё при установке библиотеки
```python
aiohttp_session[secure]
```
Для инициализации сессии, добавляем её в middleware.
```python
session_middleware(EncryptedCookieStorage(SECRET_KEY)),
```
Чтобы достать или положить значения в сессию, нужно сначала извлечь её из запроса
```python
session = await get_session(request)
```

Для авторизации пользователя, я ложу в сессию его id, а потом в middleware проверяю его наличие. Конечно для безопасности нужно больше проверок, но для тестирования концепции хватит и этого.

__Статика__

Статика подключается отдельным роутом при инициализации приложения.
```python
app.router.add_static('/static', 'static', name='static')
```
Чтобы задействовать её в шаблоне, нужно достать её из app
```python
<script src="{{ app.router.static.url(filename='js/main.js') }}"></script>
```
Все просто, ничего сложного нету.

__WebSocket__

Наконец-то мы добрались до самой вкусной части aiohttp)
Реализация сокетов очень проста
В javascript я добавил минимально необходимый функционал для работы сокета
```javascript
try{
    var sock = new WebSocket('ws://' + window.location.host + '/ws');
}
catch(err){
    var sock = new WebSocket('wss://' + window.location.host + '/ws');
}

// show message in div#subscribe
function showMessage(message) {
    var messageElem = $('#subscribe'),
        height = 0,
        date = new Date();
        options = {hour12: false};
    messageElem.append($('<p>').html('[' + date.toLocaleTimeString('en-US', options) + '] ' + message + '\n'));
    messageElem.find('p').each(function(i, value){
        height += parseInt($(this).height());
    });

    messageElem.animate({scrollTop: height});
}

function sendMessage(){
    var msg = $('#message');
    sock.send(msg.val());
    msg.val('').focus();
}

sock.onopen = function(){
    showMessage('Connection to server started')
}

// send message from form
$('#submit').click(function() {
    sendMessage();
});

$('#message').keyup(function(e){
    if(e.keyCode == 13){
        sendMessage();
    }
});

// income message handler
sock.onmessage = function(event) {
  showMessage(event.data);
};

$('#signout').click(function(){
    window.location.href = "signout"
});

sock.onclose = function(event){
    if(event.wasClean){
        showMessage('Clean connection end')
    }else{
        showMessage('Connection broken')
    }
};

sock.onerror = function(error){
    showMessage(error);
}
```
Для реализации сервера я использую вьюху WebSocket
```python
class WebSocket(web.View):
    async def get(self):
        ws = web.WebSocketResponse()
        await ws.prepare(self.request)

        session = await get_session(self.request)
        user = User(self.request.db, {'id': session.get('user')})
        login = await user.get_login()

        for _ws in self.request.app['websockets']:
            _ws.send_str('%s joined' % login)
        self.request.app['websockets'].append(ws)

        async for msg in ws:
            if msg.tp == MsgType.text:
                if msg.data == 'close':
                    await ws.close()
                else:
                    message = Message(self.request.db)
                    result = await message.save(user=login, msg=msg.data)
                    print(result)
                    for _ws in self.request.app['websockets']:
                        _ws.send_str('(%s) %s' % (login, msg.data))
            elif msg.tp == MsgType.error:
                print('ws connection closed with exception %s' % ws.exception())

        self.request.app['websockets'].remove(ws)
        for _ws in self.request.app['websockets']:
            _ws.send_str('%s disconected' % login)
        print('websocket connection closed')

        return ws
```
Сам сокет создаётся используя функцию WebSocketResponse().
Обязательно перед использованием его нужно "приготовить".
Список открытых сокетов у меня хранится в приложении(чтобы при закрытии сервера их можно было корректно закрыть).
При подключении нового пользователя, все участники получают уведомление о том что новый участник присоединился к чату.
Далее мы ожидаем сообщения от пользователя. Если оно валидно, мы сохраняем его в базе данных и отсылаем другим участникам чата.
Когда сокет закрывается, мы удаляем его из списка сокетов и оповещаем чат, что его покинул один из участников.
Очень простая реализация, визуально в синхронним стиле, без кучи колбеков, как в торнадо к примеру. Бери и пользуйся).

__Выгрузка на Heroku__

Тестовый [чат](https://secure-escarpment-46948.herokuapp.com/) я выложил на Heroku, для наглядной демонстрации.
При деплое возникло несколько проблем, в частности для использования их внутренней базы mongodb нужно было заполнить кучу информации, что делать мне было лень, поэтому я воспользовался услугами [MongoLab](https://mlab.com/) и создал базу там.
Далее были проблемы с установкой самого приложения. Для установки cryptography нужно было явно указывать его в requirements.txt.
Также для указания версии python нужно создавать в корне проекта файл [runtime.txt](https://github.com/Crandel/aiohttp/blob/master/runtime.txt)

__Выводы__

В целом создание чата , изучение aiohttp, разбор работы сокетов и некоторых других технологий, с которыми я до этого не работал,заняло у меня где-то около 3 недель работы по вечерам и редко на выходных. 
Документация в aiohttp довольно неплохая, много асинхронных драйверов и обёрток уже готовы для тестирования.
Возможно для продакшена пока не все готово, но развитие идёт очень активно(за 3 недели aiohttp обновилась с версии 0.19 до 0.21).
Если нужно добавить в проект сокеты, то этот вариант отлично подойдёт, чтобы не тащить тяжёлую tornado в зависимости.

__Ссылки__
[Github](https://github.com/Crandel/aiohttp)
[Aiohttp](http://aiohttp.readthedocs.org/en/stable/)