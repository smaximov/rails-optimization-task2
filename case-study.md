# Case-study оптимизации

## Актуальная проблема
В нашем проекте возникла серьёзная проблема.

Необходимо было обработать файл с данными, чуть больше ста мегабайт.

У нас уже была программа на `ruby`, которая умела делать нужную обработку.

Она успешно работала на файлах размером пару мегабайт, но для большого файла она работала слишком долго, и не было понятно, закончит ли она вообще работу за какое-то разумное время.

Я решил исправить эту проблему, оптимизировав эту программу.

## Формирование метрики
Для того, чтобы понимать, дают ли мои изменения положительный эффект на быстродействие программы, я придумал
использовать такую метрику: объём потребляемой процессом Ruby памяти после выполнения программы.

## Беглый анализ асимптотики

Я не стал использовать подход, который я использовал при выполнении
[задания №1](https://github.com/hardcode-dev/rails-optimization-task1/pull/60), а именно я не стал нарезать
выборки "на лету" при выполнении скриптов замера производительности/памяти и профилировании, т.к. это скорее
всего внесло бы изменения в потреблении памяти программой, что сделало бы оценку неточной. Вместо этого я
решил нарезать файлы для выборок разных размеров заранее с помощью скрипта `bin/setup`, который подготовил
выборки размерами 2500, 5000, 10000, 20000, 25000, 50000 и 100000 строк. Для этих же выборок я выполнил начальный
замер нашей метрики:

``` sh
$ for sample_size in 2500 5000 10000 20000 25000 50000 100000; do bin/bench $sample_size; done
```

Полученные результаты отражены в следующей таблице:

| Размер выборки | Потребляемая память, MB | Время выполнения, сек. |
|----------------|-------------------------|------------------------|
| 2500           | 41                      | 0.0874                 |
| 5000           | 63                      | 0.3099                 |
| 10000          | 98                      | 1.0644                 |
| 20000          | 135                     | 4.3705                 |
| 25000          | 162                     | 7.1858                 |
| 50000          | 254                     | 40.7473                |
| 100000         | 341                     | 169.2121               |

Из этих данных я делаю вывод, что потребляемая память зависит линейно от размера выборки. А это значит, что
на обработку исходного файла потребуется примерно 11 GB памяти:

``` sh
$ ruby -e 'puts (341 * 3250940.0 / 100000).round'
11086
```

При дальнейшей оптимизации я буду использовать размер выборки 20000, т.к. обработка такого файла выполняется
относительно быстро.

## Гарантия корректности работы оптимизированной программы
Программа поставлялась с тестом. Выполнение этого теста в фидбек-лупе позволяет не допустить изменения логики
программы при оптимизации.

## Feedback-Loop
Для того, чтобы иметь возможность быстро проверять гипотезы я выстроил эффективный `feedback-loop`,
который позволил мне получать обратную связь по эффективности сделанных изменений за 5-10 секунд.

Вот как я построил feedback-loop:

1. профилирую скрипт, чтобы выявить "точки роста"
2. вношу изменения в скрипт;
3. запускаю регрессионный тест, чтобы убедиться, что изменения не поломали код;
4. запускаю бенчмарк;
5. анализирую результаты;
6. коммит или возврат изменений.

Пункты 3-4 реализованы в скрипте bin/feedback.

## Вникаем в детали системы, чтобы найти главные точки роста
Для того, чтобы найти "точки роста" для оптимизации я воспользовался *инструментами, которыми вы воспользовались*

Вот какие проблемы удалось найти и решить

### Находка №1: потоковая обработка данных

- Согласно условию задачи надо переписать скрипт таким образом, чтобы он использовал потоковую обработку данных.
- Т.к. теперь по условию задачи гарантируется, что сессии пользователя в файле идут одним куском сразу после самого
   пользователя, то я переписал обработку следующим образом. Теперь мы читаем файл построчно и сохраняем информацию
   о пользователе и его сессияи хранится в переменной `last_user_stats`. Сначала читается строка, которая относится
   к пользователю, затем читаются его сессии, которые агрегируются в `last_user_stats`. Когда сессии текущего
   пользователя заканчиваются, мы выводим его в файл и переходим к парсингу следующего пользователя. Общая же
   информация о пользователях и сессиях агрегируется в объекте `summary`.
- Значение метрики для 20000 строк снизилось с 135 MB до 25 MB. Для интереса я решил попробовал прогнать скрипт на
   выборках большего размера (в том числе и `large`), и понял, что перестарался: скрипт начал выполняться в
   константном объеме памяти 25-26 MB на выборках любого размера (26 MB / 20 секунд на `large`, т.е. в бюджет
   мы уже попали).
- Теперь скрипт переписан на потоковую обработку данных, дальше я занялся профилированием.

### Находка №2: ненужный парсинг дат
- Я воспользовался `ruby-prof` с отчётом `callgrind` (смотрел в `kcachegrind`) и выявил главную точку роста:
   `Date.parse` с 22% self и ~17000 вызовов. Другие отчёты, а также `stack-prof`, показали такую же картину.
- Как и в прошлой задаче, я избавился от парсинга дат и просто отсортировал их строковое представление (т.к.
   даты для формата ISO8601 результат лексикографической сортировки совпадает с результатом "временной").
- По результатам `bin/bench` потребление памяти не изменилось (однако memory profiler показал, что Total allocated
  снизилось с 40.85 MB (581142 objects) до 17.19 MB (283770 objects)).
- В профилировщике метод `Date.parse` перестал быть точкой роста.

### Ваша находка №X
- какой отчёт показал главную точку роста
- как вы решили её оптимизировать
- как изменилась метрика
- как изменился отчёт профилировщика

## Результаты
В результате проделанной оптимизации наконец удалось обработать файл с данными.
Удалось улучшить метрику системы с *того, что у вас было в начале, до того, что получилось в конце* и уложиться в заданный бюджет.

*Какими ещё результами можете поделиться*

## Защита от регрессии производительности
Для защиты от потери достигнутого прогресса при дальнейших изменениях программы *о performance-тестах, которые вы написали*