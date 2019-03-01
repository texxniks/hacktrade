HackTrade
=============
Фреймворк для написания торговых роботов. *Первый драфт документации. Присылайте ошибки.*

Краткое руководство
-------------------
В данном руководстве рассказано из каких частей состоит торговый робот. Руководство расчитано на пользователя с начальными знаниями программирования или языка **Lua**.

Итак, минимальный робот в **HackTrade** состоит из 3-х элементов:

1. Исполнения основного модуля
2. Определения основной функции
3. Вызова функции **Trade**, для осуществления торговых действий

```lua
dofile("hacktrade.lua")

function Robot()
  Trade()
end
```

Фреймворк состоит из одного файла [hacktrade.lua](hacktrade.lua). Для инициализации фреймворка вы должны в первую очередь исполнить файл с его кодом. Для этого используется стандартная функция **dofile** в которой указывается путь к файлу (в приведённом примере фреймворк находится в той же папке, что и робот).

Следующий шаг - объявление функции **Robot**. Эта функция должна содержать код робота, при этом она выполнена в формате сопрограммы и может быть явно прервана на любом этапе выполнения. Это позволяет фреймворку заходить в функцию и продолжать её выполнение, тем самым вы не должны явно вставлять код обработки заявок и получения данных, фреймворк это сделает за вас, только передайте ему управление. Такой подход к написанию робота позволяет вам без лишних управляющих структур запрограммировать конечный автомат. Это даёт возможность создавать предсказуемых и наджных роботов.

Чтобы передать управление фреймворку и обработать заявки объявлена специальная функция **Trade**. Когда вы вызываете эту функцию, выполнение функции **Robot** прерывается и фреймворк получает управления для совершения полезной работы: пересчёт заявок, выставление и снятие заявок. Когда фреймворк закончит полезную работу, то он снова вызовет функцию **Robot** с того же места, где последний раз вызывалась функция **Trade**.

Когда в функции **Robot** исполнятся все строки, робот завершится. Поэтому робот, приведённый выше, завершиться практически сразу после запуска. Чтобы ваш робот торговал продолжительное время, вам необходимо создать бесконечный цикл, в котором вызывается функция **Trade**:

```lua
dofile("hacktrade.lua")

function Robot()
  while true do
    Trade()
  end
end
```

Теперь ваш робот будет работать, пока вы его явно не остановите.

Рыночные данные
---------------

Для получения рыночных данных доступны два типа объектов: **MarketData** и **Indicator**. Первый позволяет получить данные о последней сделке и стакане, а второй информацию с графиков. Добавим объект **MarketData** к нашему роботу:


```lua
dofile("hacktrade.lua")

function Robot()

  feed = MarketData{
    market = "MCODE",
    ticker = "TCKR",
  }

  while true do
    Trade()
  end
end
```

Для создания экземпляра объекта необходимо передать параметры: *код рынка* и *тикер*. В результате создаётся объект и сохраняется в переменную **feed**. Экземпляр будет использоваться для получения цены последней сделки.

Чтобы получить различные параметры торгового инструмента, обратитесь к атрибутам объекта **feed**;

```lua
feed.last             -- Цена последней сделки
feed.quantity         -- Количество в лотах
feed.high             -- Максимальная цена за торговую сессию
feed.bid              -- Лучшая цена спроса
feed.offer            -- Лучшая цена предложения
feed.sec_scale        -- Точность цены
feed.sec_price_step   -- Минимальный шаг цены
```

Доступны и другие параметры, названия которых вы можете посмотреть в документации к терминалу **QUIK**.

Умные заявки
------------

В даннмом разделе представлена реализация stateful-заявок, которые помнят своё внутреннее состояние и могут динамически принимать изменения параметров. Этот подход сильно отличается от традиционных лимитных заявок.

На данный функционал вдохновила система **QATCH**. Заявка в фреймворке **HackTrade** представляет собой динамическую лимитную заявку с изменяемой ценой. При этом также может меняться количество и даже направление. Главное, то нужно запомнить, "умная заявка" будет пытаться набирать указанное количество лотов по заданной цене. Даже если вы снимите заявку торговой систем, **SmartOrder** породит новую и продолжить добирать заданное ранее количество (представьте, что **SmartOrder** - это заявка, которая гарантированно набирает зданный объём).

Для большинства роботов на хватит одной умной заявки, но вы можете создавать их больше, например для арбитражной стратегии. где для каждого инструмента должна быть своя умная заявка, либо для маркет-мейкера, когда в рынке стоят сразу несколько заявок. Давайте создадим одну такую заявку:

```lua
dofile("hacktrade.lua")

function Robot()

  feed = MarketData{
    market = "MCODE",
    ticker = "TCKR",
  }

  order = SmartOrder{
    market = "MCODE",
    ticker = "TCKR",
    account = "ACNT1234567890",
    client = "775533",
  }

  while true do
    Trade()
  end
end
```

Для создания заявки требуется указать: рынок, тикер с которыми будет работать заявка, а также номер счёта и код клиента, чтобы определить, от имени какого пользователя и по какому счёту будет торговать система (если у вас много счетов, можете создать одного универсального работа, либо склонировать его и адаптировать под разные счета и инструменты).

Чтобы открыть позицию требуемого размера вам нужно вызвать метод **update** указав цену и желаемое количество:

```lua
order:update(123.5, 15)
```

В приведённом примере заявка будет покупать *15* лотов по *123.5*. Допустим, заявка заполнена на *9* лотов, но рынок сильно ушёл не в вашу пользу. Не проблема, вы можете исполнить следующий код:

```lua
order:update(131.0, 15)
```

Теперь "умная заявка" снимет лимитную заявку на *15* лотов по цене *123.5* (которая заполнена на *9* лотов) и выставит новую по цене *131* и с количеством *6*; фактически заявка докупит лоты по новой цене.

Но если вы увеличите или уменьшите количество (а не только цену), то заявка будет докупать или ликвидировать часть позиции. Например, предыдующая заявка была исполнена и вы хотите уменьшить позицию:

```lua
order:update(129.5, 10)
```

В таком случае выставится лимитная заявка на продажу (!) *5* лотов по цене *129.5*. Фактически, заявка приводит вашу позицию к желаемому значению по указанной цене.

Были рассмотрены длинные позиции. Чтобы указать заявке занять короткую позицию передайте отрицательное количество:

```lua
order:update(129.5, -10)
```

Если бы предпоследнее указание было бы исполнено и у нас было *10* лотов (лонг), то последнее указание бы инициировало продажу 20 (!) лотов. То есть заявка бы заяняла позицию шорт на 10 лотов, при этом цена сделки не изменилась с предыдущего шага.

Если захотите открыть длинную позицию, просто передайте полоительное значение. Даже если короткая позиция открыта не полностью, заявка это учтёт.

Перейдём к нашему роботу: конечно же, вы можете получать цену из **feed**! Пример:


```lua
dofile("hacktrade.lua")

function Robot()

  feed = MarketData{
    market = "MCODE",
    ticker = "TCKR",
  }
  
  order = SmartOrder{
    market = "MCODE",
    ticker = "TCKR",
    account = "ACNT1234567890",
    client = "775533",
  }

  while true do
    order:update(feed.last, 1)
    Trade()
  end
end
```

Робот просто откроет лонг в *1* лот по цене последней сделки. **Внмание!** Так как у нас обновление заявки происходит в цикле, периодически передавая управление фреймворку, то при изменении цены последней сделки наш робот будет подстраивать цену лимитированной заявки, пока не получит долгожданный лот. Фактически, робот догоняет цену!

Теперь нужно сделать логику более полезной, привязав принятие решений об открытии позиций к торговым индикаторам. В дополнение скажу, что на умных заявках не сложно реализовать ступенчатые заявки, такие как айсберг (очень простой пример):

```lua
while true do
  if order.position == 0 then
    order:update(feed.last, 10)
  elseif order.position == 10 then
    order:update(feed.last, 20)
  end
  Trade()
end
```

Получение данных индикаторов
----------------------------

Даваайте дополним робота техническим индикатором (скользящей средней). Для этого воспользуемся классом **Indicator** (формально в **Lua** нет классов, в фреймворк добавлено немного магии с метатаблицами):

```lua
  ind = Indicator{tag="MYIND"}
```

Для создания индикатора требуется указать тег - идентификатор, который можно привязать к графику в терминале **QUIK** (как это сделать, посмотрите в документации). Теперь вы можете получать доступ к значениям индикатора. существует много способов, так как индикаторы иногда состоят из множества линий. Также раелизована обратная индексация, когда значения можно получать с конца не зная количество значений (свечек). Примеры:

```lua
ind[1]           -- Таблица со всеми параметрами первой свечки
ind[-1]          -- Таблица со всеми параметрами последней свечки
ind.values[-1]   -- Последнее значение цены закрытия (values это синоним close)
ind.closes[-2]   -- Предпоследнее значение цены закрытия
ind.closes_0[-2] -- Нулевая линия, предпоследнее значение цены закрытия
ind.opens_1[-5]  -- Первая линия, цена открытия 5-й свечки с конца
```

Обратите внимание, что вам вернётся столько значений, сколько задано в параметрах графика. Не добавляйте на график слишком много значений, это может замедлить работу программы.

Давайте дополним нашего робота реакцией на положение цены последней сделки относительно индикатора:

```lua
dofile("hacktrade.lua")

function Robot()

  feed = MarketData{
    market = "MCODE",
    ticker = "TCKR",
  }
  
  order = SmartOrder{
    market = "MCODE",
    ticker = "TCKR",
    account = "ACNT1234567890",
    client = "775533",
  }

  ind = Indicator{tag="MYIND"}

  while true do
    if feed.last > ind.closes[-2] then
      order:update(feed.last, 1)
    elseif feed.last < ind.closes[-2] then
      order:update(feed.last, -1)
    end
    Trade()
  end
end
```

Чтобы закрыть позицию вы можете, при определённых условиях, выполнить код:

```lua
order:update(feed.last, 0)
```

Робот готов. Это классический реверс по скользящей средней. Вы можете менять значение графика в реальном времени. Робот подстроится!

В качестве дополнения давайте посмотрим робота, который покупает на дешевле цены последней сделки и продаёт дороже (приведён только основной цикл):

```lua
while true do
  if feed.last > ind.closes[-2] then
    order:update(feed.last, 1)
  elseif feed.last < ind.closes[-2] then
    order:update(feed.last, -1)
  end
  Trade()
end
```

Разработчикам
-------------
При разработке рекомендуется использовать `lua5.1`, т.к. в quik используется [5.1.5](https://forum.quik.ru/forum10/topic3816/)

Код покрыт тестами, использующими [busted](https://olivinelabs.com/busted/), для которого есть хорошая документация и подробно описан процесс [установки](https://olivinelabs.com/busted/#usage). Запускаются они следующим образом:

```bash
$ busted
```
Можно также удобно просмотреть список всех проверяемых сценариев без фактического запуска
```bash
$ busted -l
```


Участие в проекте
-----------------
Уже возможно, присылайте патчи. Просьба весь изменяемый функционал покрывать автотестами.
Позже опишу более простой процесс с Pull Request.
