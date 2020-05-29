## В чем суть игры?

Надо написать своего бота для змейки, который обыграет других ботов по очкам. Все играют на одном поле. Змейка может передвигаться по свободным ячейкам во все четыре стороны, но не может возвращаться в предыдущую клетку.
Ваша задача – заставить вашу змейку следовать придуманному алгоритму. Алгоритм змейки должен блокировать (и тем самым уничтожать) змей соперников, используя бонусы.

## Правила
Игра пошаговая, каждую секунду сервер посылает твоему клиенту (боту) состояние обновленного поля на текущий момент и ожидает ответа команды герою. За следующую секунду игрок должен успеть дать команду герою. Если не успел — змейка продолжает движение в прежнем направлении(изначальное направление - направо). На своем пути змейка может повстречать камень, стенку, золото, таблетку ярости, яблоко или другую змейку. Единоразово на поле присутствует по 5 змеек.

**Побеждает игрок с большим числом очков** (до условленного времени). 
Мертвая змейка тут же появляется на одном из свободных стартовых полей и **ждёт начала следующего раунда** (события старта).

* `Если змейка нарвётся на камень` - уменьшается. 
* `Если змейка меньше двух клеток` - погибнет. 
* `За золото, яблоки, в случае убийства других змеек` - змейка получит бонусные очки. 
* `За свою смерть и за камни` - штрафные. 
* Очки суммируются.

_Негативные воздействия:_
* Змейка, врезавшаяся в стену, погибает.
* Змейка, врезавшаяся в другую змейку, погибает.
* Змейка должна быть не менее 2х клеток длиной, иначе она погибает.
* Змейка, съевшая камень, укорачивается на 3 единицы (если становится меньше 2 - погибает).

_Позитивные воздействия:_
* Змейка, съевшая яблоко, увеличивается на 1.
* Змейка, съевшая таблетку ярости, в течении 10 ходов может откусывать куски от других змей, а так же поедать камни без уменьшения длины.
* Змейка, съевшая золото, просто получает дополнительные очки.

_Особые случаи:_
* Змеям разрешено откусывать свой собственный хвост. При этом длина змеи уменьшается, больше никаких последствий.
* При столкновении змей лоб в лоб, меньшая змея погибает. Большая змея при этом укорачивается на длину меньшей. (если становится меньше 2, тоже погибает)
* При столкновении двух змей, всегда побеждает змея, находящаяся под воздействием таблетки ярости.
* При столкновении двух змей, когда обе под воздействием таблетки ярости, действуют обычные правила столкновений.

## Порядок обработки действий на сервере
1. Если первой была передана команда ACT - то оставляется камень на поле, если имеется таковая возможность. Камень будет оставлен на той клетке, где на данный момент времени находится хвост змейки.
2. Происходит обработка перемещения змеек.
3. Происходит обработка драки змеек.
4. Происходит обработка поедания змейками объектов на новой точке.
5. Снова происходит обработка драки змеек.
6. Воссоздаются новые объекты взамен тех, что были съедены змейками.
7. Если команда ACT была передана после команды на передвижение змейки - то оставляется камень на поле, если имеется таковая возможность. Камень будет оставлен на той клетке, где на данный момент времени находится хвост змейки.

### Порядок обработки передвижения змеек
1. Вычисляется направление движения для каждой змейки. Если от пользователя не поступило никакой команды, то змейка продолжит двигаться в предыдущем направлении. Сразу после старта движение по умолчанию - направо. Если поступила команда змейке на разворот на 180 градусов (то есть если змейка смотрела вверх, а пришла команда двигаться вниз), то такая команда будет проигнорирована, змейка продолжит двигаться в прежнем направлении.
2. Для каждой змейки проходит проверка на то, должна ли её длина быть сокращена. Если да – то проверяется, не погибает ли змейка от такого сокращения, после чего длина змейки сокращается.
3. Сокращается количество оставшихся тиков для действия таблетки ярости, если таковая была активна.
4. Производится проверка, что если змейка сама с собой пересекается на новой точке – тогда откусывается всё её тело от кончика хвоста до точки пересечения. Змейка при этом не погибает, так как минимальная длина - 2 единицы, а перекусить саму себя и оставить при этом длину меньше 2 единиц не представляется возможным.
5. Происходит перемещение змейки на новую клетку. То есть голова змейки передвигается на новую клетку, её шея передвигается на ту клетку, где ранее была голова и т.д.

### Порядок обработки драки змеек
1. Для каждой змейки находим, есть ли змейки, на голову которых зашла головой текущая змейка. Под головой понимается непосредственно голова змейки + её шея, то есть клетка сразу за головой змейки.
2. Если таковая имеется, то дальше обрабатываем ситуации с таблетками ярости.
2.1. Если текущая змейка в ярости, а вражеская нет - то убиваем вражескую змейку.
2.2. Если текущая змейка не в ярости, а вражеская в ярости - то текущая змейка умирает.
2.3. Если обе змейки в ярости или обе не в ярости - то от каждой отрезаем длину другой змейки. То есть, если столкнулись две змейки длиной в 5 и в 7 единиц, то змейка длиной в пять погибает, так как 5-7<2, а змейка длиной в 7 принимает длину 2 единицы.
3. Если не было змейки, на голову которой зашла головой текущая змейка - то ищем змейку, на тело которой зашла текущая змейка. Под телом понимаются все клетки тела вражеской змеи за исключением её головы и клетки, следующей непосредственно за головой змейки.
4. Если таковые имеются, то тогда проверяем, а текущая змейка находится под таблеткой ярости или нет:
4.1. Если да - тогда от вражеской змейки отгрызаем всё от хвоста до места куся.
4.2. Если нет - то текущая змейка погибает.

### Порядок обработки поедания змейками объектов на поле
Проверяется позиция головы змейки:
1. Если там находится яблоко - то увеличивается длина змейку на 1 и засчитываем очки за поедание яблока.
2. Если там находится камень - то увеличивается количество съеденных камней на 1 и сокращаем текущую длину змейки, если змейка не находится в состоянии ярости. Если находится - тогда змейка спокойно проглатывает камень без негативных последствий для себя.
3. Если там находится таблетка ярости - тогда она съедаются, то есть эта таблетка становится активна уже на этом тике.
4. Если там находится стена - тогда змейка погибает.
5. Если там находится монетка – тогда она съедается и за неё начисляются бонусные очки.

### Правила начисления очков
1. За монетку даётся +10 очков.
2. За яблоко даётся +1 очко.
3. За камень даётся +5 очков.
4. За уничтожение соперника +10 умножить на длина соперника очков.
5. За победу в раунде начисляются очки по следующему правилу - 110 + длина змейки \* 10 + 1 \* количество тиков, прошедшее от начала раунда. То есть, если игрок победил и остался один на поле по истечению 100 тиков, при этом его змейка имеет длину в 8 единиц, то ему начислится следующее количество очков за победу - 110 + 10 \* 8 + 1 \* 100. Если игра заканчивается по истечению времени раунда - то побеждают только те змейки, которые имеют максимальную длину!
6. За откусывание частей тела от вражеской змейки или за её убийство при лобовом столкновении очки начисляются по следующему правилу - 10 \* длину откушенной части тела (при убийстве в лобовом берётся общая длина вражеской змейки).

### Правила окончания раунда
Раунд завершается, если наступает одно из следующих условий:
1. Одна или несколько змеек достигают максимальной длины в 40 единиц.
2. Истекает отведённое на раунд время - 300 тиков.
3. Остаётся одна живая змейка или погибают все змейки на поле.

## Как подключиться? 
1. Регистрируетесь на сервере.
2. Скачиваете шаблоны клиентов и выбираете тот, который будете модифицировать для себя.
3. После входа в игру копируете URL из адресной строки браузера полностью, как есть.
4. Вставляете URL в строку подключения в клиенте.

## Как работает клиент?

После подключения, клиент через вебсокет будет регулярно (каждую секунду) получать строку символов с закодированным состоянием игрового поля. 
Само игровое поле извлекается из ответа сервера регулярным выражением:
```
^board=(.*)$
```
Длинна строки равна площади поля. Если вставить символ переноса строки каждые sqrt(length(string)) символов, то получится читабельное изображение поля. Пример ответа сервера с игровым полем:
```
☼☼☼☼☼☼☼☼☼☼☼☼☼☼☼
☼☼            ☼
☼☼     $      ☼
☼☼           @☼
☼#           ▲☼
☼☼           ║☼
☼☼   ○       ║☼
☼#           ║☼
☼☼         ╘═╝☼
☼☼      %     ☼
☼☼   ×—>      ☼
☼☼            ☼
☼☼       ●    ☼
☼☼            ☼
☼☼☼☼☼☼☼☼☼☼☼☼☼☼☼
```

- Первый символ строки соответствует ячейке, расположенной в левом верхнем углу и имеет координату `[0, 0]`. Последний символ, 
соответственно, соответствует правому нижнему углу игрового поля `[n, n]` 
- Максимальное количество игроков определяется количеством стартовых точек на карте.

Расшифровка символов игрового поля:

- `NONE(' ')` Пустое место – по которому может двигаться герой
- `WALL('☼')` Cтена, через которую нельзя пройти
- `START_FLOOR('#')` Стартовая точка. Отсюда начинает движение змейка
- `APPLE('○')` Яблоко
- `STONE('●')` Гиря (Камень)
- `FURY_PILL('®')` Маска дьявола (Таблетка ярости)
- `GOLD('$')` Золото
- `HEAD_DOWN('▼')` Голова змейки игрока в направлении вниз
- `HEAD_LEFT('◄')` Голова змейки игрока в направлении влево
- `HEAD_RIGHT('►')` Голова змейки игрока в направлении вправо
- `HEAD_UP('▲')` Голова змейки игрока в направлении вверх
- `HEAD_DEAD('☻')` Голова мёртвой змейки игрока
- `HEAD_EVIL('♥')` Голова змейки игрока со сверхспособностью "ярость"
- `HEAD_SLEEP('&')` Голова неактивной змейки игрока
- `TAIL_END_DOWN('╙')` Хвост змейки игрока повернутый вниз
- `TAIL_END_LEFT('╘')` Хвост змейки игрока повернутый влево
- `TAIL_END_UP('╓')` Хвост змейки игрока повернутый вверх
- `TAIL_END_RIGHT('╕')` Хвост змейки игрока повернутый вправо
- `TAIL_INACTIVE('~')` Тело неактивной змейки игрока
- `BODY_HORIZONTAL('═')` Вариант расположения тела активной змейки игрока
- `BODY_VERTICAL('║')` Вариант расположения тела активной змейки игрока
- `BODY_LEFT_DOWN('╗')` Вариант расположения тела активной змейки игрока
- `BODY_LEFT_UP('╝')` Вариант расположения тела активной змейки игрока
- `BODY_RIGHT_DOWN('╔')` Вариант расположения тела активной змейки игрока
- `BODY_RIGHT_UP('╚')` Вариант расположения тела активной змейки игрока
- `ENEMY_HEAD_DOWN('˅')` Голова змейки соперника в направлении вниз
- `NEMY_HEAD_LEFT('<')` Голова змейки соперника в направлении влево
- `ENEMY_HEAD_RIGHT('>')` Голова змейки соперника в направлении вправо
- `ENEMY_HEAD_UP('˄')` Голова змейки соперника в направлении вверх
- `ENEMY_HEAD_DEAD('☺')` Голова мёртвой змейки соперника
- `ENEMY_HEAD_EVIL('♣')` Голова змейки соперника со сверхспособностью "ярость"
- `ENEMY_HEAD_SLEEP('ø')` Голова неактивной змейки соперника
- `ENEMY_TAIL_END_DOWN('¤')` Хвост змейки соперника повернутый вниз
- `ENEMY_TAIL_END_LEFT('×')` Хвост змейки соперника повернутый влево
- `ENEMY_TAIL_END_UP('æ')` Хвост змейки соперника повернутый вверх
- `ENEMY_TAIL_END_RIGHT('ö')` Хвост змейки соперника повернутый вправо
- `ENEMY_TAIL_INACTIVE('*')` Тело неактивной змейки соперника
- `ENEMY_BODY_HORIZONTAL('—')` Вариант расположения тела активной змейки игрока
- `ENEMY_BODY_VERTICAL('|')` Вариант расположения тела активной змейки игрока
- `ENEMY_BODY_LEFT_DOWN('┐')` Вариант расположения тела активной змейки игрока
- `ENEMY_BODY_LEFT_UP('┘')` Вариант расположения тела активной змейки игрока
- `ENEMY_BODY_RIGHT_DOWN('┌')` Вариант расположения тела активной змейки игрока
- `ENEMY_BODY_RIGHT_UP('└')` Вариант расположения тела активной змейки игрока


## Протокол взаимодействия с сервером
Команд несколько: 
* `UP`
* `DOWN`
* `LEFT`
* `RIGHT` – приводят к движению героя в заданном направлении на 1 клетку; 
* `ACT` - оставить камень (если есть съеденные камни). 

Камень остаётся на месте хвоста змейки. При помощи камней можно строить препядствия и блокировать соперников. Команды движения можно комбинировать с командой ACT, разделяя их через запятую – это значит что за один такт игры будет оставлен камень, а потом движение `LEFT, ACT` или наоборот `ACT, LEFT`. Если передать команду `ACT(0)`, то змейка тут же погибнет.


**Удачи! И пусть победит хитрейший! =)**

## Подсказки:
Если не знаете что написать, попробуйте реализовать следующие варианты алгоритмов:
- Перемещение в случайную сторону, если соответствующая клетка свободна.
- Движение на свободную клетку в сторону ближайшего яблока.
- Движение к тому яблоку, добраться до которого можно быстрее других.
- Уклонение от более длинных соперников и от соперников в состоянии ярости.
- Блокировка собой предполагаемых маршрутов соперника.

## Окей, попробуем почитить!  
1. Нет, более 2х команд подряд в одном тике не сработают ☹
2. Нет, `ACT,ACT` или `LEFT,UP` не сработают - считай, что это пропуск хода 
3. Нет, если отправить невалидную команду, сервер не упадет


