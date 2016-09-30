# Такая разная асинхронность

Здравствуйте, меня зовут Дмитрий Карловский и я.. многозадачный человек. В смысле у меня много задач и мало времени, чтобы их все уже, наконец, закончить. Отчасти это и к лучшему - всегда есть чем заняться. С другой стороны - пока ты разрываешься между проектами, мир катится куда-то не туда и некому забраться на броневик и призвать толпу остановиться и немного подумать. А вопрос-то серьёзный - долгое время мир JS был погружён в *ад обратных звонков* и с ними не только не боролись - их боготворили. Потом он чуть менее чем полностью погряз в обещаниях. Сейчас к ним с разных сторон усиленно вставляют подпорки разной степени кривизны. А света в конце тоннеля всё не видать. Но обо всём по порядку...

# Теория многозадачности

Сперва определимся с терминами. В процессе работы, приложение выполняет различные задачи. Например, "скачать файл с удалённого сервера" или "обработать запрос пользователя".

Не редки ситуации, когда для выполнения одной задачи требуется выполнение дополнительных задач - "подзадач". Например, для обработки запроса пользователя, необходимо скачать файл с удалённого сервера.

Запустить подзадачу мы можем синхронно, и тогда текущая задача заблокируется в ожидании завершения подзадачи. А можем запустить асинхронно, и тогда текущая задача продолжит своё выполнение не дожидаясь завершения подзадачи.

Тем не менее, обычно для завершения выполнения задачи, пусть и не сразу, но требуется и завершение выполнения подзадачи с последующей обработкой её результатов. Блокировку одной задачи в ожидании сигналов от другой будем называть "синхронизацией". В общем случае, синхронизация одних и тех же задач может происходить и множество раз, по самой различной логике, но в дальнейшем мы будем рассматривать лишь простейший и самый распространённый вариант - синхронизацию по завершению подзадачи.

<cut />

В языках, поддерживающих многопоточность, обычно каждая задача запускается в отдельном "системном потоке" или (более правильно) "нити". Каждая нить может исполняться на отдельном ядре процессора, параллельно с другими нитями. Так как нитей может быть много, а число ядер весьма ограничено, то операционная система реализует механизм "вытесняющей многопоточности", когда любая нить, если она долго исполняется, может быть принудительно приостановлена, чтобы дать возможность поработать другим нитям.

Параллельная работа задач приводит к различным проблемам при работе с общей памятью, для решения которых приходится использовать нетривиальные механизмы синхронизации. Чтобы упростить работу программиста и повысить надёжность, производимого им программного обеспечения, некоторые языки полностью отказываются от многопоточности и запускают все задачи в одной единственной нити. Многозадачность в этом случае реализуется одним из следующих способов:

**Волокна** (fibers), также известные как "сопрограммы" (coroutines). По сути это те же нити, но реализующие "кооперативную многозадачность". Все волокна имеют свои стеки, но исполняются в рамках одной нити, а значит не могут исполняться параллельно. При этом решение о том, когда переключить нить на другое волокно, принимает само волокно.

**Цепочки задач**. Суть подхода в том, что вместо того, чтобы приостанавливать текущую задачу на время выполнения подзадачи, мы разбиваем задачу на много маленьких подзадач и говорим каждой, какую подзадачу нужно выполнить по завершении этой.

**Конечные автоматы** (state machine), также известные как "генераторы" (generators), "асинхронные функции" (async functions) и "полусопрограммы" (semicoroutines) и "сопрограммы без стека" (stackless coroutines). Фактически, это объекты, хранящие локальное состояние единственного метода, в начале которого находится ветвление с переходом к коду одного из шагов исходной задачи. По завершении шага управление возвращается вызвавшей функции. Повторный вызов асинхронной функции уже приводит к переходу к другому шагу.

# Реализации на NodeJS

В репозитории [nin-jin/async-js](https://github.com/nin-jin/async-js) в отдельных ветках собраны реализации простого приложения на разных моделях многозадачности. Суть приложения простая и состоит из 3 частей:

1. **Модель** (user.js). Загружает конфиг с диска и предоставляет метод для получения имени пользователя из этого конфига.
2. **Отображение** (greeter.js). Принимает модель пользователя и печатает, обращение к нему в консоль.
3. **Контроллер** (index.js). Печатает пользователю приветствие, а затем прощание. Попутно выводит время своей работы и логирует ошибку, если происходит исключительная ситуация, не давая процессу упасть.

Конфиг простой:
```
{
    "name" : "Anonymous"
}
```

## [Синхронный код](https://github.com/nin-jin/async-js/compare/sync?diff=unified&name=sync)

**user.js**
```
var fs = require( 'fs' )

var config

var getConfig = () => {
    if( config ) return config

    var configText = fs.readFileSync( 'config.json' )
    return config = JSON.parse( configText )
}

module.exports.getName = () => {
    return getConfig().name
}
```

**greeter.js**
```
module.exports.say = ( greeting , user ) => {
    console.log( greeting + ', ' + user.getName() + '!' )
}
```

**index.js**
```
var user = require( './user' )
var greeter = require( './greeter' )

try {

    console.time( 'time' )
    greeter.say( 'Hello' , user )
    greeter.say( 'Bye' , user )
    console.timeEnd( 'time' )

} catch( error ) {
    console.error( error )
}
```

Крайне простой и понятный. В нём легко разбираться и не менее легко вносить изменения. Но у него есть один существенный недостаток - пока выполняется эта задача никакая другая задача выполнена быть не может, даже если мы ждём загрузки файла с сетевого диска и ничего полезного не делаем. Если это скрипт одной задачи, как в примере выше, то ничего страшного, но если нам нужен веб-сервер, который должен обрабатывать множество запросов одновременно, то однозадачное решение нам не подходит.

## [Предопределённые цепочки](https://github.com/nin-jin/async-js/compare/sync...async-nodejs)

Многие синхронные методы в [NodeJS API](https://nodejs.org/en/docs/) имеют и свои асинхронные аналоги, где последним аргументом передаётся "продолжение" (continuation), то есть функция, которую следует вызвать после завершения асинхронной задачи.

**user.js**
```
var fs = require( 'fs' )

var config

var getConfig = done => {

    if( config ) return setImmediate( () => {
        return done( null , config )
    })

    fs.readFile( 'config.json' , ( error , configText ) => {
        if( error ) return done( error )

        try {
            config = JSON.parse( configText )
        } catch( error ) {
            return done( error )
        }

        return done( null , config )
    })

}

module.exports.getName = done => {

    getConfig( ( error , config ) => {
        if( error ) return done( error )

        try {
            var name = config.name
        } catch( error ) {
            return done( error )
        }

        return done( null , name )
    } )

}
```

**greeter.js**
```
module.exports.say = ( greeting , user , done ) => {

    user.getName( ( error , name ) => {
        if( error ) return done( error )

        console.log( greeting + ', ' + name + '!' )

        return done()
    })

}
```

**index.js**
```
var user = require( './user' )
var greeter = require( './greeter' )

var script = done => {
    console.time( 'time' )

    greeter.say( 'Hello' , user , error => {
        if( error ) return done( error )

        greeter.say( 'Bye' , user , error => {
            if( error ) return done( error )

            console.timeEnd( 'time' )

            done()
        } )

    } )

}

script( error => {
    if( !error ) return

    console.error( error )
} )
```

Как видно, код заметно усложнился. Нам пришлось все (даже синхронные) функции, переписать в цепочечном стиле. При этом, правильная обработка ошибок доставляет особую боль: если забыть где-то обработать ошибку, то приложение может упасть, а может не упасть, а может упасть, но не сразу, а чуть позже, вдалеке от места возникновения ошибки. А если оно и каким-то чудом не упадёт, то и ошибка никаким образом залогирована не будет. Написание кода в таком стиле требует от программиста чуткости и внимательности, поэтому большинство модулей в NPM - заряженные пистолеты, способные в любой момент подарить вам незабываемые часы в компании отладчика.

## [Постопредляемые цепочки](https://github.com/nin-jin/async-js/compare/sync...async-promises)

Реализуемые через "обещания" (promises), они берут на себя основную работу по прокидыванию ошибок. Единственное, что нужно помнить - в конце цепочки должен стоять обработчик ошибок, иначе приложение может завершиться по среди выполнения задачи, ничего при этом не сказав.

**user.js**
```
var fs = require( 'fs' )

var config

var getConfig = () => {
    return new Promise( ( resolve , reject ) => {
        if( config ) return resolve( config )

        fs.readFile( 'config.json' , ( error , configText ) => {
            if( error ) return reject( error )

            return resolve( config = JSON.parse( configText ) )
        } )
    } )
}

module.exports.getName = () => {
    return getConfig().then( config => {
        return config.name
    } )
}
```

**greeter.js**
```
module.exports.say = ( greeting , user ) => {
    return user.getName().then( name => {
        console.log( greeting + ', ' + name + '!' )
    } )
}
```

**index.js**
```
var user = require( './user' )
var greeter = require( './greeter' )

Promise.resolve()
.then( () => {
    console.time( 'time' )
    return greeter.say( 'Hello' , user )
} )
.then( () => {
    return greeter.say( 'Bye' , user )
} )
.then( () => {
    console.timeEnd( 'time' )
} )
.catch( error => {
    console.error( error )
} )
```

По сравнению с предопределёнными цепочками, код получился по проще, но всё так же разбит на множество мелких функций. Преимуществом данного подхода является то, что он будет  работать одинаково хорошо в любом окружении. Даже там, где обещаний нет изначально - их легко добавить не сложной библиотекой.

В целом, оба вида цепочек приводят к большому объёму визуального шума и усложнению написания нелинейных алгоритмов, использующих циклы, условные ветвления, локальные переменные и так далее.

## [Генераторы](https://github.com/nin-jin/async-js/compare/sync...async-generators-co)

Некоторые JS-движки поддерживают генераторы, которые довольно элегантно интегрируются с обещаниями, что позволяет реализовывать "приостанавливаемые функции" (awaitable).

**user.js**
```
var fs = require( 'fs' )
var co = require( 'co' )

var config

var getConfig = () => {
    if( config ) return config

    return config = new Promise( ( resolve , reject ) => {
        fs.readFile( 'config.json' , ( error , configText ) => {
            if( error ) return reject( error )
            resolve( JSON.parse( configText ) )
        } )
    } )
}

module.exports.getName = co.wrap( function* () {
    return ( yield getConfig() ).name
} )
```

**greeter.js**
```
var co = require( 'co' )

module.exports.say = co.wrap( function* ( greeting , user ) {
    console.log( greeting + ', ' + ( yield user.getName() ) + '!' )
} )
```

**index.js**
```
var co = require( 'co' )

var user = require( './user' )
var greeter = require( './greeter' )

co( function*() {
    console.time( 'time' )
    yield greeter.say( 'Hello' , user )
    yield greeter.say( 'Bye' , user )
    console.timeEnd( 'time' )
} ).catch( error => {
    console.error( error )
} )
```

Код получился почти столь же простым, что и синхронный, разве что нам пришлось все функции превратить в генераторы и завернуть в специальную обёртку, которая получив (yield) от генератора обещание, подписывается на его "резолв", после которого "продолжает" генератор с передачей ему полученного значения. Таким образом мы снова можем пользоваться условными ветвлениями, циклами и прочими идиомами управления потоком.

## [Асинхронные функции](https://github.com/nin-jin/async-js/compare/sync...async-await-babel)

Фактически это не более, чем синтаксический сахар для генераторов. Но сахар этот ещё мало где поддерживается, поэтому пока ещё приходится использовать babel для трансформации в код на генераторах.

**user.js**
```
var fs = require( 'fs' )

var config

var getConfig = () => {
    if( config ) return config

    return config = new Promise( ( resolve , reject ) => {
        fs.readFile( 'config.json' , ( error , configText ) => {
            if( error ) return reject( error )

            resolve( JSON.parse( configText ) )
        } )
    } )
}

module.exports.getName = async () => {
    return ( await getConfig() ).name
}
```

**greeter.js**
```
module.exports.say = async ( greeting , user ) => {
    console.log( greeting + ', ' + ( await user.getName() ) + '!' )
}
```

**index.js**
```
var user = require( './user' )
var greeter = require( './greeter' )

async function app() {
    console.time('time')
    await greeter.say('Hello', user)
    await greeter.say('Bye', user)
    console.timeEnd('time')
}

app().catch( error => {
    console.error( error )
} )
```

### [Волокна](https://github.com/nin-jin/async-js/compare/sync...async-fibers)

Несложное нативное расширение для NodeJS реализует полноценные волокна. Всё, что вам нужно - это запустить задачу в волокне и далее, на любом уровне вложенности вызовов функций вы можете приостановить волокно, передав управление другому. В примере далее используются так называемые "фьючеры" (futures), которые позволяют в любой момент синхронизовать одну задачу с другой.

**user.js**
```
var Future = require( 'fibers/future' )
var FS = Future.wrap( require( 'fs' ) )

var config

var getConfig = () => {
    if( config ) return config

    var configText = FS.readFileFuture( 'config.json' )
    return config = JSON.parse( configText.wait() )
}

module.exports.getName = () => {
    return getConfig().name
}
```

**greeter.js**

А его даже не потребовалось менять - он всё такой же синхронный.

**index.js**
```
var Future = require( 'fibers/future' )

var user = require( './user' )
var greeter = require( './greeter' )

Future.task( () => {

    try {

        console.time('time')
        greeter.say('Hello', user)
        greeter.say('Bye', user)
        console.timeEnd('time')

    } catch( error ) {
        console.error( error )
    }

} ).detach()
```

При использовании волокон, большая часть кода остаётся синхронной, но в случае необходимости ожидания, блокируется не вся нить, а лишь отдельное волокно. В результате получается как бы параллельное исполнение синхронных волокон.

# Производительность

Сравним время выполнения основной задачи в каждом варианте многозадачности на NodeJS v6.3.1:

1. Синхронный код: 4мс.
2. Предопределённые цепочки: 6мс.
3. Обещания: 7мс.
4. Генераторы: 7мс.
5. Асинхронные функции превращённые в генераторы через Babel: 22мс.
6. Волокна: 6мс.

Выводы:

1. Синхронный код существенно быстрее асинхронного.
2. Волокна практически не дают пенальти по производительности (только на запуск и переключение волокон).
3. Обещания и генераторы дают пенальти на вызов каждой функции. В примере у нас мало функций, поэтому просадка не большая.
4. Babel генерирует весьма паршивый код.

# Отладка

Давайте посмотрим как наши приложения отреагируют на исключительную ситуацию. Например, в конфиг вместо объекта поместим просто `null`. Загрузка и парсинг конфига пройдёт нормально, а вот метод `getName` должен упасть с ошибкой. Мы уже позаботились, чтобы приложение не упало, не проигнорировало ошибку, а залогировало стектрейс в консоль. Вот, что выведут наши реализации:

## Синхронный код

```
TypeError: Cannot read property 'name' of null
    at Object.module.exports.getName (./user.js:13:23)
    at Object.module.exports.say (./greeter.js:2:41)
    at Object.<anonymous> (./index.js:7:13)
    at Module._compile (module.js:541:32)
    at Object.Module._extensions..js (module.js:550:10)
    at Module.load (module.js:456:32)
    at tryModuleLoad (module.js:415:12)
    at Function.Module._load (module.js:407:3)
    at Function.Module.runMain (module.js:575:10)
    at startup (node.js:160:18)
```

Похоже стектрейс захватил изрядную долю внутренностей NodeJS, но главное, что интересующая нас последовательность вызовов `index.js:7 -> say@greeter.js:2 -> getName@user.js:13` присутствует, а значит мы сможем понять как приложение докатилось до этой ошибки.

## Предопределённые цепочки

```
TypeError: Cannot read property 'name' of null
    at error (./user.js:31:30)
    at fs.readFile.error (./user.js:20:16)
    at FSReqWrap.readFileAfterClose [as oncomplete] (fs.js:439:3)
```

Стектрейс начинается от прихода события о загрузке файла. Что было до этого мы уже не узнаем.

## Обещания

```
TypeError: Cannot read property 'name' of null
    at getConfig.then.config (./user.js:19:22)
```

Максимально минималистичный стектрейс.

## Генераторы

```
TypeError: Cannot read property 'name' of null
    at Object.<anonymous> (./user.js:18:33)
    at next (native)
    at onFulfilled (./node_modules/co/index.js:65:19)
```

Тут используются те же обещания со всеми вытекающими отсюда последствиями.

## Асинхронные функции

```
TypeError: Cannot read property 'name' of null
    at Object.<anonymous> (user.js:18:12)
    at undefined.next (native)
    at step (C:\proj\async-js\user.js:1:253)
    at C:\proj\async-js\user.js:1:430
```

Странно было бы ожидать тут чего-то другого.

## Волокна

```
TypeError: Cannot read property 'name' of null
    at Object.module.exports.getName (./user.js:14:23)
    at Object.module.exports.say (./greeter.js:2:41)
    at Future.task.error (./index.js:11:17)
    at ./node_modules/fibers/future.js:467:21
```

Всё, что надо и почти ничего лишнего.

В отладчике вы увидите ту же самую картину: вы сможете пройтись по стеку синхронного и волоконизированного кода, посмотреть значения локальных переменных, поставить точки останова и по шагам пройтись по исполнению вашего приложения. В то же время, код разбитый на цепочки функций, приправленный обещаниями или завёрнутый в генераторы - настоящий кошмар для отладчика. А если вы ещё и кривым транспилятором воспользовались, то храбрости разработчика, взявшегося за отладку этого кода, может позавидовать сам [Мустафа Хуссейн](http://domfactov.com/vnuk-saddama-huseyna-mustafa-huseyn-samyiy-hrabryiy-malchik.html).

# Что делать?

1. Не гнаться за модой, а использовать решения, позволяющие писать лаконичный, быстрый, удобный в отладке код.
2. Помогать *людям в солнцезащитных очках* искать путь к свету.
3. Пропагандировать всесторонний анализ проблематики, вместо проталкивания однобокого мнения.

Волокна, объективно, по сумме качеств, лучше остальных представленных тут решений. Единственный минус - это ни в коей мере не стандарт и в браузерах даже не планируется к реализации. Но это не столько минус волокон, сколько минус сообщества, которое проталкивает в стандарты обещания, генераторы, асинхронные функции, но совершенно игнорирует куда более простые и прямые решения.

# Ссылки

1. [github:nin-jin/async-js](https://github.com/nin-jin/async-js) - исходники примеров.
2. [wiki:coroutine](https://en.wikipedia.org/wiki/Coroutine) - подборка информации о сопрограммах как концепции.
3. [npm:node-fibers](https://github.com/laverdet/node-fibers) - модуль добавляющий волокна в NodeJS.
