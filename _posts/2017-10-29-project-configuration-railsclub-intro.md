---
layout: post
title:  "Project configuration. Part 1. Railsclub intro."
date:   2017-10-29
desc: "Intro into project configuration thread"
keywords: "ruby,ruby on rails,configuration,monilith,microservice"
# categories: [conference]
tags: [configuration,conference,railsclub 2017]
icon: icon-microscope
---

Данная статья является изначально была транскриптом к докладу на конференции Railsclub 2017. По итогам конференции в текст были внесены незначительные исправления.

В процессе чтения вы поймете, почему нет смысла сразу описывать все детали в мельчайших подробностях. Я не снимаю с себя ответственности в вопросе предоставления интересующей информации. Вы можете написать в комментариях, какие вопросы у вас возникли и что стоит подробнее изложить. Это послужит триггером для детальных публикаций.

Поехали.

Сегодня я хочу поговорить по поводу темы, которая может показаться окружающим старой, избитой и не интересной. Но все, на самом деле не так. Как говорится, все новое, это хорошо забытое старое, особенное если старое **должно** быть обновлено. Я думаю, что вы слышали о том, что все доклады делятся на 2 типа: похвастаться или исповедаться. Сегодня я буду делать второе (наверное).

В процессе подготовки к докладу, я думал как лучше начать повествование... Придумал слезливую историю про свой опыт борьбы с воздушными замками, нашел пугающие картинки... А потом зашел на википедию и... нашел ответ на вопрос, который хочу раскрыть.

> Конфигурация программного обеспечения — совокупность настроек программы, задаваемая пользователем, **а также процесс изменения этих настроек в соответствии с нуждами пользователя**.

Проблема в том, что большинство смотрит только на первую часть этого определения, и (очень часто) не берут во внимание вторую часть. Конечно, проблема не только в этом, ключевые аспекты мы рассмотрим, но именно этот вопрос сподвиг меня пересмотреть свои взгляды на тему конфигурации проектов и выработать новый подход. И вопрос не только в том, как обновлять конфигурацию в прямом смысле этого слова. Нет, я не буду говорить о том, как нужно нажимать на клавиши, для того, чтобы породить код. Я хочу затронуть проблему внесения изменений в конфигурацию, доставку этих изменений до продакшена и работой в разных окружениях с одним и том же кодом. Вот об этом я и поговорю с вами сегодня. И да, разговор будет в контексте ruby, конечно (но можно транслировать на другие языки и технологии) :)

Начну я с небольшого лирического отступления. Вот тут многие говорят, что Ruby умер, что он не развивается, что мир тлен и надо валить. Так получилось, что за последние 3 года я увидел ряд совершенно разных проектов. Разных не только в плане предметной области, но и в плане архитектуры. Это были и монолиты, и полу-монолиты, и полу-полу-монолиты, и проекты на микросервисной архитектуре. И проекты на MRI, и на JRuby, и примеси на других языках. Зоопарк, одним словом.
Так вот, в каждом из этих проектов применялись разные подходы к конфигурации этих проектов. Подходы были разные, проблемы тоже были разные. Но все эти _проблемы_ сводились к одному и тому же слову: **конфигурация**.
Так получилось, что при участии в одном проекте у меня просто начало, как говорится, пригорать. Компетентность и профессионализм парней в команде не вызывали никаких вопросов, но вот подходы и определенные решения... я долго не мог понять, почему принимались именно такие решения, но видел что они ведут в никуда. А потом понял. Проблема была на поверхности: **К чему привыкли - то и исповедуем**. Поэтому вопрос **Привычки** рассмотрю в первую очередь.
Когда мы говорили с организаторами конференции Railsclub по поводу этого доклада, я им отправил вот это [видео](https://www.youtube.com/watch?v=FeCXooh8AXE). С тех пор прошло достаточно количество времени, чтобы я смог подобрать политкорректные слова, систематизировать свою ненависть и спокойно все изложить.

# Сonvention over Сonfiguration
Самое смешное в этой истории то, что экосистема rails построена на соглашениях, а именно вопрос конфигурации не подчиняется этому правилу и в этом вопросе кто во что горазд. Да, это просто такой вот парадокс. У нас будут соглашения, чтобы не было тонны конфигураций, но от конфигурации все равно не уйти.

Если кто не знает, Сonvention over Сonfiguration — это принцип построения фреймворков и библиотек, призванный сократить количество требуемой конфигурации без потери гибкости. Обычно переводится как «соглашения по конфигурации». В строгой форме этот принцип можно выразить так: аспект программной системы нуждается в конфигурации тогда и только тогда, когда этот аспект НЕ удовлетворяет некоторой спецификации. Принцип работает когда речь идет о маппинге классов на какие-либо ресурсы (таблицы базы данных, события, ресурсы файловой системы). Согласно принципу, если класс соответствует соглашению наименования, тогда он не нуждается в дополнительной конфигурации. В этом контексте название принципа можно перевести как «Соглашение НАД конфигурацией», такой перевод указывает на первостепенность соглашения, а не конфигурации.

Классический пример CoC принципа — Hibernate. В Hibernate правила объектно-реляционного меппинга можно описывать с помощью XML-файлов:

```xml
<class name="Tag" table="tag">
    <property name="Name" column="name"/>
    <property name="Value" column="value"/>
    <property name="Value2" column="value2"/>
    <property name="Value3" column="value3"/>
</class>
```

Как видно, здесь имеются повторения свойств класса и колонок таблицы. Если ввести соглашение о том, что по умолчанию колонки таблицы должны назаваться также как свойство, то можно опустить часть конфигурации:

```xml
<class name="Tag">
    <property name="Name"/>
    <property name="Value"/>
    <property name="Value2"/>
    <property name="Value3"/>
</class>
```

Однако, если какое-то свойство потребуется сохранить в колонку с отличным именем, это придется указать явно. В этом и состоит суть принципа.

Наибольшее применение принцип CoC находит в среде Ruby on Rails. Это в принципе понятно, если учесть, что RoR ориентирована на быструю разработку, а CoC позволяет свести конфигурацию к минимуму.

И так-то это правильно, но! Но это ни в коем виде не снимает ответственности с вопросов конфигурации. Не правильно считать, что все проблемы за вас уже решены другими людьми и если что-то в вашей работе выходит за пределы гайдов, то можно "немножко поговнокодить".

# Монолит - не монолит

Хотя, я не с того начал. Сначала стоит спросить, кто из вас работает над проектом-монолитом?
Вот этим людям, скорее всего, статья  в первой его части не интересной, ибо все намного проще (нет, не просто, проблемы есть), когда у вас приложение - монолит. Что мы имеем: все изменения хранятся в одном месте, их намного проще синхронизировать. Очень часто, с монолитами никаких _серьезных_ проблем не возникает. А если и возникают какие-то конфликты в плане конфигурирования - они не причиняют много боли. Немного абстрактных костылей, немного террора в сторону DevOps и все, проблемы, в большинстве своем, решены. Но вот когда же оно начинает разделяться на маленькие сервисы (советую с этим не торопиться, если не знаете во что ввязываетесь) - тут возникает много вопросов касательно того, как поддерживать консистентность, и, в том числе, в конфигурации. Ведь все прекрасно понимают, что чем больше независимых компонентов в системе, тем сложнее их синхронизировать. А еще подключается правило "работает - не трогай", о котором мы поговорим позже :).

Поэтому у меня сегодня несколько целей:
1. Поворчать по поводу того, как люди стреляют себе в ногу и считают это нормальным.
2. Посеять нотку сомнения среди части людей по поводу того, что все хорошо и ничего не нужно делать / менять.
3. Осветить проблему микросервисов на руби (да и не только на руби, но у нас же Railsclub, поэтому руби)

# Существующие подходы

А для этого нужно посмотреть на ретроспективу, без этого никак. Все помнят, как эволюционировали подходы к конфигурации? Кстати, все заметили, что я не спросил еще ни у кого, что по их мнению является _конфигурацией_?

##  Constants

Самый простой, быстрый и, в некоторых случаях, незаменимый вариант. Достаточно вспомнить как описывается версия ruby gem'а.

```ruby
# lib/persey/version.rb
module Persey
  VERSION = "0.0.11"
end

# persey.gemspec
# ...
Gem::Specification.new do |spec|
  # ...
  spec.version = Persey::VERSION
  # ...
end
```

Подход, с использованием констант для в роли инструмента для конфигурации я вижу очень часто, особенно в различных библиотеках.

```ruby
# https://github.com/rails/rails/blob/master/actioncable/lib/action_cable/gem_version.rb
module ActionCable
  def self.gem_version
    Gem::Version.new VERSION::STRING
  end

  module VERSION
    MAJOR = 5
    MINOR = 2
    TINY  = 0
    PRE   = "alpha"

    STRING = [MAJOR, MINOR, TINY, PRE].compact.join(".")
  end
end
```

Или вот:

```ruby
# https://github.com/rails/rails/blob/master/actioncable/lib/action_cable.rb
module ActionCable
  extend ActiveSupport::Autoload

  INTERNAL = {
    message_types: {
      welcome: "welcome".freeze,
      ping: "ping".freeze,
      confirmation: "confirm_subscription".freeze,
      rejection: "reject_subscription".freeze
    },
    default_mount_path: "/cable".freeze,
    protocols: [
      "actioncable-v1-json".freeze,
      "actioncable-unsupported".freeze
    ].freeze
  }
  # ...
end

# https://github.com/rails/rails/blob/master/actioncable/lib/action_cable/connection/web_socket.rb

module ActionCable
  module Connection
    # Wrap the real socket to minimize the externally-presented API
    class WebSocket # :nodoc:
      def initialize(env,
                     event_target,
                     event_loop,
                     protocols: ActionCable::INTERNAL[:protocols]
                    )
         # ...
      end
      # ...
    end
  end
end
```

Проекты бо'льшего масштаба исключением не являются (к сожалению). И если, в примерах выше, это хоть как-то это можно назвать константами, то в реальности разработчики настолько увлекаются, что записывают в константы все что угодно.

Как-то я был на собеседовании в одной компании. В ходе собеседования мы затронули тему "А как ты думаешь, можно ли использовать константы для хранения конфигурации?". Я тогда однозначно ответил что нет, что это плохо. Спойлер (именно из-за этого мою кандидатуру и отвергли, за что им большое спасибо).

Какие доводы были приведены в пользу использования констант?
1. К константам можно получить доступ отовсюду (если файл с константами загружен)
2. Константы позволяют повысить читаемость кода. Магические числа и строки пропадают.
3. Легко ищутся по коду (ты ищешь название константы, а не магическое число)
4. Нельзя модифицировать в runtime.

Ну в принципе, все так и есть (почти). Но вот контраргументы тоже есть.
1. Никто не держит константы в одном месте. Они всегда размазываются по коду. Для того, чтобы переиспользовать ту или иную константу, нужно знать, где она лежит (в каком файле/модуле/классе). А это никто не хочет запоминать (более того, это никому не нужно). В итоге, все приводит к тому, что разработчики начинают дублировать эти конфигурационные параметры. Как только нужно поменять ту или иную магическую строчку, начинается игра в "угадай, как коллега назвал константу", для того, чтобы найти её по коду. Лечится ли эта проблема? Да, побольше дисциплины, концентрации и внимания. Только вот зачем все это?
2. Грань между константным значением и параметром конфигурации начинает теряться. В константы начинают пихать все, что придет в голову.

```ruby
RETRIES_COUNT = 10
API_HOST = 'https://your.domain.com'
```

3.  Разные окружения превращают жизнь в ад. В тестовой среде нужны одни значения, на стейжинге другие, в продакшене третьи и тремя-четырьмя окружениями проект может не ограничиться.

```ruby
RETRIES_COUNT = case Rails.env
                when 'test'
                  2
                when 'development'
                  5
                when 'staging'
                  10
                else
                  50
                end

API_HOST = case Rails.env
                when 'test'
                  'https://fake.domain.com'
                when 'development'
                  'https://loalhost:3000'
                when 'staging'
                  'https://your-staging.domain.com'
                else
                  'https://your-production.domain.com'
                end
```

4. Не изменяются? Ну да, конечно.

```ruby
class SomeClass
  CONSTANT_STRING = "string"
  CONSTANT_NUMBER = 1
  CONSTANT_ARRAY = [1, 2, 3]
  CONSTANT_HASH = { a: :b }
  CONSTANT_HELL = {
    a: CONSTANT_HASH,
    b: CONSTANT_ARRAY,
    c: {
      d: :e,
      f: [1, 2, 3]
    }
  }
end

[1] pry(main)> SomeClass::CONSTANT_STRING
=> "string"
[2] pry(main)> SomeClass::CONSTANT_STRING << 'smth'
=> "stringsmth"
[3] pry(main)> SomeClass::CONSTANT_STRING
=> "stringsmth"
```

Всем известно, что  тут все просто, используем .freeze и тогда нельзя будет менять никакие значения.

```ruby
class SomeClass
  CONSTANT_STRING = "string".freeze
  CONSTANT_NUMBER = 1.freeze
  CONSTANT_ARRAY = [1, 2, 3].freeze
  CONSTANT_HASH = { a: :b }.freeze
  CONSTANT_HELL = {
    a: CONSTANT_HASH,
    b: CONSTANT_ARRAY,
    c: {
      d: :e,
      f: [1, 2, 3]
    }
  }.freeze
end

[1] pry(main)> SomeClass::CONSTANT_STRING
=> "string"
[2] pry(main)> SomeClass::CONSTANT_STRING << 'smth'
RuntimeError: can't modify frozen String

[3] pry(main)> SomeClass::CONSTANT_HELL[:a] = 123
RuntimeError: can't modify frozen Hash

[4] pry(main)> SomeClass::CONSTANT_HELL
=> {:a=>{:a=>:b}, :b=>[1, 2, 3], :c=>{:d=>:e, :f=>[1, 2, 3]}}
[5] pry(main)> SomeClass::CONSTANT_HELL[:c][:d] = 123
=> 123
[6] pry(main)> SomeClass::CONSTANT_HELL
=> {:a=>{:a=>:b}, :b=>[1, 2, 3], :c=>{:d=>123, :f=>[1, 2, 3]}}
```

Только вот freeze не "замораживает" вложенные объекты, и на такую ситуацию можно очень легко нарваться.

Но нужно отдать должное: переопределить константу не получится. Это единственное, пожалуй, чем они хороши.

Если кому-то интересно спросить про то, как это работает внутри ruby машины - поговорим с вами от этом в кулуарах, поэтому пойдем дальше.

Да, если кто-то хочет сказать, что можно при этом использовать переменные окружения - да, можно. Но это также не работает :)

## Config file
Второй по популярности подход - складывать все параметры в YAML файлы, которые потом будут считаны и полученный хеш будет использоваться как объект хранения конфигурации.

Плюсы у этого подхода, несомненно есть. Как и минусы, в прочем...

### Плюсы:
1. Наконец-то конфигурация стала аккумулироваться в одном месте
2. YAML файлы позволяют изящно описать конфигурацию, разделить по окружениям, убрать дублирование

```yaml
common: &common
  adapter: postgresql
  host:     <%= ENV.fetch('DB_HOST') %>
  database: <%= ENV.fetch('DB_NAME') %>
  username: <%= ENV.fetch('DB_USERNAME') %>
  password: <%= ENV.fetch('DB_PASSWORD') %>
  encoding: unicode
  pool: <%= ENV.fetch('DB_POOL') %>

development:
  <<: *common

test:
  <<: *common

staging:
  <<: *common
  port: 5432
```

3. Можно использовать переменные окружения (вариант прогона считанного файла через ERB препроцессор используется повсеместно)
4. Разделение конфигов по предметным областям - это очень хорошо. Люблю, когда все лежит по полочкам.

### Минусы:
1. Вы зависите от внутренней реализации того или иного гема (особенно в части использование переменных окружения). Например, конфиг базы данных проходит обработку ERB препроцессором. А что случилось с `cable.yml`? За что его обделили?
2. Вы не можете переприменять один и тот же параметр конфигурирования между разными файлами, так как они автономны. Исключение конечно можно сходу сказать - шарить эти данные через переменные окружения. Кто из вас это делал в большом проекте, где таких переменных окружения довольно много.

```bash
  $ grep -R ENV config app lib | wc -l
      458
```

3. Вы не можете переприменять части конфига в самом конфиге (ну на самом деле это не правда, имеется ввиду то, что изящно это сделать не получится)
4. Сложно поддерживать изменения, если их структура файлов изменяется
5. И самая большая проблема - сложно следить за изменениями в этих конфигурационных файлах.

С последним пунктом я познакомился довольно давно, когда контрибьютил в Gitlab. Там было 3 основных проблемы:
1. Синхронизация изменений файлов (исходные файлы поставлялись в виде `config.yml.example`), который нужно было скопировать и переписать. Как только в `config.yml.example` появляются новые параметры, которые вы, в силу какие-то обстоятельств, не заметили, у вас могут начаться проблемы.
2. Поддержка изменений в самих файлах (поддержка обратной совместимости)
3. Дублирование конфигурационных параметров (был gitlab-shell и сам gitlab. Обе части должны были знать о том, по какому пути лежат репозитории. Если приходилось менять один параметр - нужно было помнить про другой. Но Погодите, это же один продукт, что за фигня?)

Как решились эти проблемы? Да никак. Ну то есть мы у себя решили их ;) но `Undev` больше нет и никто уже про это не вспоминает. Я, по крайней мере, надеюсь на это.

Что ж, на текущий момент этот подход лидирует в сообществе rails (и не только) разработчиков.

## DSL

Про этот вариант наверное стоило сказать раньше, так как он появился и прижился раньше, чем вариант с файлами конфигурации. Но я его все же поставил следующим пунктом, так как он все же может использоваться в более гибком варианте.

Я думаю, что все знают про

```ruby
Rails.application.configure do
# ...
end
```

Так вот этот вариант как раз из этой оперы.

Плюсы:
1. Конфиг читается очень хорошо
2. Конфиг == код. Возможностей для описания конфигурации становится больше
3. Можно тестировать (!!!). Да, конфиг можно протестировать и быть уверенным что с ним все хорошо. Только вот никто это не делает. Тут, конечно, можно похоливарить, о том что нужно тестировать, а что нет.
4. Все плюсы из **Config object**

Мне в частности, еще нравится вариант `Dry-configurable` (я вообще поклонник Dry)

```ruby
class App
  extend Dry::Configurable

  # Pass a block for nested configuration (works to any depth)
  setting :database do
    # Can pass a default value
    setting :dsn, 'sqlite:memory'
  end
  # Defaults to nil if no default value is given
  setting :adapter
  # Pre-process values
  setting(:path, 'test') { |value| Pathname(value) }
  # Passing the reader option as true will create attr_reader method for the class
  setting :pool, 5, reader: true
  # Passing the reader attributes works with nested configuration
  setting :uploader, reader: true do
    setting :bucket, 'dev'
  end
end

App.config.database.dsn
# => "sqlite:memory"

App.configure do |config|
  config.database.dsn = 'jdbc:sqlite:memory'
end

App.config.database.dsn
# => "jdbc:sqlite:memory"
App.config.adapter
# => nil
App.config.path
# => #<Pathname:test>
App.pool
# => 5
App.uploader.bucket
# => 'dev'
```


## Singleton

Также довольно крепко прижился подобный вариант хранения конфигурации и в рубишном мире (в том числе и в самом коде Rails).

Суть в этом подходе сводится к следующему: данные конфигурации считываются из какого-то источника (источников) и сохраняются в одном объекте.

Плюсы и минусы зависят от того, откуда и как берутся эти данные.

## DB

Я всегда вспоминаю времена работы в web студии, когда клепал стайтики на CMS'ках, когда вижу подобный подход. Когда есть табличка в базе, там в виде key => value хранятся параметры конфигурирование и разработчик из кода бегает в эту базу для того, чтобы считывать те или иные значения. На самом деле, отвратительно плохого в этом ничего нет. Это один из способов расшарить конфигурацию, которая может поменяться в любой момент. И не важно, где она будет (PostgreSQL, Redis, etc)

Проблемы:
- Очень хрупкая система (живет только в том случае, когда невозможно удалить ключ)
- Нагрузка на базу данных может быть очень большой
- Паразитный траффик
- Просадки в скорости работы приложения

Плюсы:
- Конфигурацию отдельных компонентов системы можно менять на лету
- Шаринг конфигурации между разными сервисами из коробки

# Проблемы, на которые большинство не обращает внимания

## Много конфигов

Очевидно, что чем сложнее приложение, тем больше библиотек используется. И зачастую, каждая библиотека просит свой собственный конфигурационный файл. Если говорить в контексте Rails, это приводит к тому, что 2 директории начинают распухать от количества подобных файлов:

`config`
`config/initializers`

Проблема тут, как понимаете, не в количестве файлов, а в количестве переменных/модулей/классов, которые хранят значения конфигурации.
Так или иначе, рост количества таких сущностей приводит к тому, что появляются дополнительные абстракции, для того, чтобы соблюсти принципы DRY, что приводит к большой связанности кода и меньшему пониманию, что происходит. И начинается проявляться дополнительная проблема, как обеспечить консистентность конфигурации. Проблема поддержки. Однако, эта проблема легко перекладывается на плечи других людей (например DevOps, мы же говорили о том, что можно все эти проблемы решить при помощи переменных окружения, хотя и это тоже бред, как минимум потому что в конце концов невозможно придумывать и совмещать большое количество переменных)

## Разные форматы файлов

С подобной ситуацией сталкивается не так много команд и проектов, однако такое бывает и стоит про это сказать. Как много людей используют в своих решениях не только ruby библиотеки а конкретный софт, который имеет вполне конкретную конфигурацию?
Что, если есть небольшие сервисы, которые имеют свои конфиги? Вам же при этом (не всегда) нужно учитывать параметры из этих конфигов. Копипастить? Прокидывать через переменные окружения? А что делать, если сервис запускается супервизиром при старте системы? Жонглировать параметрами окружения? А если сервис не может считать эти параметры окружения (умеет читать только из файла)? Переписывать сервис только для того, чтобы он смог это сделать? А если вы не знаете язык программирования или ПО проприетарное?
Все эти вопросы в итоге сводятся к тому, что нужно просто продублировать важные параметры. Просто продублировать, вместо того, чтобы переиспользовать. Да, я не спорю, этот вопрос также можно решить при помощи DevOps'ов. Да и то, не всегда. В общем, иногда возникает ситуация, когда нужно просто считать еще один конфиг и иметь к нему доступ из кода приложения.

## Динамическое конфигурирование & использование конфигурации в конфигурации

Как часто вам нужно из нескольких параметров конфигурации собрать 1?
Далеко ходить не нужно - строка подключения к базе данных.
Вспомним про конфиг базы данных.

```yaml
common: &common
  adapter: postgresql
  host:     <%= ENV.fetch('DB_HOST') %>
  database: <%= ENV.fetch('DB_NAME') %>
  username: <%= ENV.fetch('DB_USERNAME') %>
  password: <%= ENV.fetch('DB_PASSWORD') %>
  encoding: unicode
  pool: <%= ENV.fetch('DB_POOL') %>

development:
  <<: *common

test:
  <<: *common

staging:
  <<: *common
  port: 5432
```

И вам в приложении нужно использовать не только данные в поэтом формате, но и строку подключения одной строкой. Что в таком случае делать?

```yaml
common: &common
  adapter: postgresql
  host:     <%= ENV.fetch('DB_HOST') %>
  database: <%= ENV.fetch('DB_NAME') %>
  username: <%= ENV.fetch('DB_NAME') %>
  password: <%= ENV.fetch('DB_PASSWORD') %>
  encoding: unicode
  pool: <%= ENV.fetch('DB_POOL') %>
  dsn: postgres://<%= ENV.fetch('DB_NAME') %>:<%= ENV.fetch('DB_PASSWORD') %>@<%= ENV.fetch('DB_HOST') %>:<%= ENV.fetch('DB_PORT') %>/<%= ENV.fetch('DB_NAME') %>?pool=<%= ENV.fetch('DB_POOL') %>
```

Никто не видит в этом проблему? На самом деле проблемы нет, есть досадная неприятность в виде необходимости контролировать 2 места использования переменной окружения вместо одного.

Но вот если наоборот, из 1 параметра вычленить несколько аттрибутов? У вас на входе `dsn` а нужно разбить на отдельные параметры (например строка подключения к AWS RDS DB)?

```ruby
common: &common
  dsn:      <%= ENV.fetch(‘DB_DSN’) %>
  adapter:  ???
  host:     ???
  database: ???
  username: ???
  password: ???
  encoding: unicode
  pool: ???
```

Или дополнить 1 параметр дополнительным данными (например несколько баз данных у редиса)? Вот тут геморроя побольше прибавляется.

Смысла забивать область переменных окружений по каждому чиху я особо не вижу. Достаточно затребовать поддержки подобного поведения/фич у той части проекта, которая отвечает за конфигурацию.

## Комбинирование разных типов конфигурации

А что делать, когда у вас есть конфигурация, описанная при помощи DSL, конфигурация, которая лежит в базе данных, файлики с конфигами, конфиги, переданные через переменные окружения? Перевести все на один формат? Например использовать только YAML файлы или только DSL? Вообще-то это правильная мысль и я бы выбрал второй вариант. Как минимум в этом случае вы можете использовать `lambda` для того, чтобы включить один конфиг во второй.

## Переиспользование конфигурации в родительских проектах (config registry)

Это также интересный момент. Давайте представим ситуацию: вы пишете небольшое приложение, которые берет данные из API, что-то с ними делает, результат записывает файл и загружает его в AWS S3 бакет. В этой абстрактной задачке можно выделить 3 компонента:
1. Обращение к API
2. Выполнение операции
3. Загрузка файла

Очевидно, что каждая из этих частей (как минимум 2) будут требовать credentials. И скорее всего, они будут лежать в 2 файликах рядышком и первая и третья компонента прекрасно их считает и все будет работать. Теперь вопрос: какова вероятность того, что вам потребуется во второй компоненте учитывать конфигурационнные параметры из 1 и 3?

Тут возникают следующие вопросы:
1. Вы используете стороннее решение?
2. Обращаться к объекту конфигурации?
3. Если конфигурация инкапсулирована и публичного доступа к ней нет?

Конечно, это же руби, вы можете вызывать приватные методы и дотянуться до самих пикантных мест, рискуя наткнуться на то, что ваш код сломается, когда изменится внутренний интерфейс библиотеки.

Или проще объединить эти 2 конфига (сделать proxy интерфейс к ним).

А теперь представим, что у нас есть некое описание моделей, с которыми может работать ваши сервисы, и вы вычленили их в отдельный гем. Так или иначе, этот гем будет иметь конфигурацию, и вам может понадобиться доступ к параметрам каких-то ключей из вашего сервиса. Как поступать в таком случае? Дублировать конфиги? Или переиспользовать объект конфигурации из родительского класса?

А если вы при этом подключили еще 1 библиотеку, и там есть свой конфиг? Обращаться напрямую к этому конфигу или тоже, пробросить доступ к нему через ваш конфиг? Если выбираете первый вариант, вам не кажется что это похоже на историю с размазывание констант? И Как быть с правилом одного уровня абстракции? И как гарантировать, что ваш интерфейс конфигурации будет неизменным (подмена бекенда не призывает к изменению кода)?

## Перезагрузка конфигов

### Перезапуск приложения vs перезагрузка конфигурации

Вернемся ненадолго к монолитам. Когда у вас приложение выполнено в виде монолита у него есть как плюсы, так и минусы в плане возможности использовать такой подход к обновлению конфигурации. Стоит начать с того, что практического смысла в перезагрузке конфигурации без перезапуска приложения особо то и нет. Как минимум потому, что в самом проекте изменения (обычно) происходят довольно быстро и вероятность того, что изменение конфигурации может придти с изменением кода очень велика. Да и изменение конфигурации в этом случае приравнивается к изменению кода, ввиду того, что связность кода очень велика.

Ситуация немного меняется, когда монолит запускается не совсем как монолит. Несмотря на то, что код и лежит в одном месте, но запускаются все эти компоненты отдельно. Например, отдельно API (да и сам API может также разделяться, например по версиям), Отдельно web часть лично кабинета, отдельно публичные контроллеры, отдельно фоновые задачи `critical` уровня и так далее.

И все кардинально изменяется, когда вместо 1-го большого приложения у вас становится много маленьких (или как люди любят это называть - микросервисы).

В последних 2-х случаях законным становится вопрос:
**Имеет ли практический смысл перезапуск всех сервисов, если код в них не поменялся, но произошли изменения в конфигурации?**

Конечно, этот вопрос имеет еще много других параметров:
- Какова стоимость перезапуска сервиса?
- Нужно ли перезапускать сервис?

Очевидно, что чем меньше сервис, тем легковеснее его перезапуск. И даунтайм здесь может быть довольно мал. Но что из себя представляет этот перезапуск? **Graceful restart** или **SIGKIL** + **SIGEXIT**? Если у вас второй вариант - перезапуск всех сервисов при необходимости перезагрузить конфигурацию становится опасной затеей. Как минимум потому что какой-то сервис может
1. Выполнять тяжелые операции
2. Содержать кеш, который после перезапуска нужно прогревать
3. Преждевременная остановка работы может повлиять на консистентность данных
4. Очень частые перезапуски могут блокировать flow

Уже этих пунктов достаточно для того, чтобы усомниться в полезности перезапускать все сервисы при изменении в одном.

Желание держать все сервисы запущенными "на максимально свежем коде" присущи многим разработчикам. Иногда до состояния фанатизма. Хм. А все из вас обновляют гемы каждый день (для того, чтобы гарантировать, что используется самая свежая версия гема)? А почему тогда нужно перезапускать сервис, если изменений в нем никаких не произошло (или изменения в соседнем сервисе не затрагивают изменения в каком-то конкретном)?

Таким образом для меня приоритетнее вопрос: **А нужно ли его перезапускать?** и **Затрагивают ли изменения конфигурации/кода работу самого приложения?**

Давайте поподробнее разберем второй момент:

Хорошим тоном является наличие изолированных хранилищ у каждого микросервиса. Под хранилищем можно понимать, например, базу данных. И все данные прокидываются между этими сервисами по сети. Но на практике это практически не достижимо (как минимум сразу). Так или иначе, в самом начале у вас будет, например, пошарена база данных между этими сервисами. Не обязательно, между всеми.  Или вам нужно использовать отдельное хранилище для коммуникаций (результат работы одного сервиса использовать в другом, при этом объем данных может быть весьма большим (У нас было такое, что несколько миллионов строк из одного сервиса нужно передать в другой). В таком случае получается, что изменение, например, части конфигурации (информации о местонахождении подобного хранилища) затрагивает не все сервисы системы, а только ограниченное число этих сервисов, соответственно, эти изменения не должны влиять на другие.

В таком случае, нужна возможность сообщить приложению о том, что конфигурация устарела и нужно ее обновить. Как это правильно сделать?

### API интерфейс для обновлении конфигурации

Почему-то чаще всего предлагают реализовать API на стороне каждого сервиса, в которое можно постучать и сообщить о том, что произошло какое-то изменение. Ну... Так себе решение. Как-то лениво __реализовывать__ подобный интерфейс для каждого сервиса самостоятельно.

### SIGHUP для поддержки перезагрузки конфигурации

Идея сводится к тому, чтобы послать **SIGHUP** сигнал сервису, для того, чтобы он обновил конфигурацию (перечитал). Подобный подход пока что еще имеет широкого применения в мире Ruby. Но это логично, так как все привыкли перезапускать приложение, если что-то поменялось.

### Проблема конфликта конфигурации (изменение конфигурации в рантайме)

Давайте представим такую ситуацию: Мы умеем перезагружать конфигурацию в реальном времени. В таком случае можем получить ситуацию, когда код был запущен на одной конфигурации, произошло её обновление и он начал работать на основании другой конфигурации. С одной стороны, это может быть не страшным, с другой...может быть и полезным.
Например вы писали часть данных в одно место, тут прилетела информация о том, что нужно писать в другое место и вы решили переключиться. Выглядит это как-то не очень хорошо. Часть данных в одном месте, часть данных в другом и не известно где и сколько.
Или у вас было ограничение на длину строки. Было `1000` символов, стало `800`. Часть сгенерированных строк будут соответствовать требуемому результату, часть нет. Хорошо это или плохо? Я не могу сказать однозначно. Для нас подобные ситуации были и полезными и вредными. Но лучше, когда результат соответствует конкретному критерию. Либо все `1000` символов, либо все `800`.

Соответственно, лучше как-то это держать под контролем. Как много людей использует `config object` паттерн в своей работе?

## Использование config object подхода

Прокидываем в класс/метод объект конфигурации и используем этот снепшот. За время выполнения операции конфиг может поменяться, снепшот остается неизменным. Если изменения конфигурации были критичными - выполнение завалится и сервис восстановится с новой конфигурацией, продолжит работу и все будет хорошо.

```ruby
def generate_ad_headline(headline_parts: [])
  headline = headline_parts.each_pair.each_with_object({}) do |(ad_id, parts), acc|
    acc[ad_id] = parts.each_with_object("") do |part, result_headline|
      if (result_headline + part).size < YourApp.config.ads.headline.max_size
        result_headline << part
      end
    end
  end
end

def generate_ad_headline(headline_parts: [], config_object: YourApp.config.ads.headline)
  headline = headline_parts.each_pair.each_with_object({}) do |(ad_id, parts), acc|
    acc[ad_id] = parts.each_with_object("") do |part, result_headline|
      if (result_headline + part).size < config.max_size
        result_headline << part
      end
    end
  end
end
```

## pub/sub модель обновления конфигурации проекта

Consul config
etcd
подобные сервисы
Проблема: у вас есть несколько сервисов






# Конфигурация проекта не только магические строки
Гибкий код с минимальным набором изменений.

## Сервис-локатеры как часть конфигурации (DI)
Не завязывайтесь на конкретный сервис (интерфейс гема) или класс, общающийся с внешним миром, напрямую. Используйте сервис-локатеры.

**Пример реализации с использование Dry-container**

```ruby
class ServiceLocator
  class Container
    include Dry::Container::Mixin
  end

  class << self
    attr_reader :instance

    def configure
      container = Container.new
      yield(container)
      @instance = new(Rails.application, container)
      freeze
    end

    def [](name)
      instance[name]
    end
  end

  attr_reader :app, :container

  def initialize(app, container)
    @app = app
    @container = container
  end

  def [](name)
    container[name]
  end
end


ServiceLocator.configure do |container|
  load_environment = begin
                       if ENV['PRODUCTION_SERVICE_LOCATOR']
                         :prod
                       elsif Rails.env.test? || Rails.env.development?
                         :fake
                       else
                         :prod
                       end
                     end

  case load_environment
  when :fake
    container.register(:sms_gateway, lambda do
      Test::Nexmo::Client.new(
        key: PactApi.config.nexmo.api_key,
        secret: PactApi.config.nexmo.api_secret
      )
    end)

    container.register(:mailgun_client, lambda do
      Test::Mailgun::Client.new(PactApi.config.mailgun.api_key)
    end)
  else
    container.register(:sms_gateway, lambda do
      Nexmo::Client.new(
        key: PactApi.config.nexmo.api_key,
        secret: PactApi.config.nexmo.api_secret
      )
    end)

    container.register(:mailgun_client, lambda do
      Mailgun::Client.new(PactApi.config.mailgun.api_key)
    end)
  end
end

email_message.take_to_send!

mg_client = ServiceLocator[:mailgun_client]
begin
  response = mg_client.send_message PactApi.config.mailgun.domain, payload

  if response.code == 200
    mailgun_message_id = response.to_h['id']
    mailgun_message_id.gsub!(/[<>]/, '')
    email_message.update(external_id: mailgun_message_id)
    email_message.sent!
  else
    email_message.fail_send!
  end
rescue
  email_message.fail_send!
end

```

## Rail-way написания кода

При реализации старайтесь явно реализовать true-way процесса.

# Вывод
