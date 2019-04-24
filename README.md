# Многопоточные питоны
В данном практическом примере, мы посмотрим как ускорить обработку url запросов через 
requests с помощью модуля threading
## Простой деревенский пример
Для начала рассмотрим самый обычный пример.

В качестве тестового url будем использовать `http://httpstat.us` т.к у него есть моного полезных методов, о которых попозже

Обычный пример обработки url'ов через requests выглядит как-то так:

```python
import requests
import time

def ping(url):
    res = requests.get(url)
    print(f'{url}: {res.text}')

urls = [
    'http://httpstat.us/200',
    'http://httpstat.us/400',
    'http://httpstat.us/404',
    'http://httpstat.us/408',
    'http://httpstat.us/500',
    'http://httpstat.us/524'
]
start = time.time()
for url in urls:
    ping(url)
print(f'Вся работа заняла у нас: {time.time() - start : .2f} Секунд')
print('Done.')
```

Собственно данный код уже работает несколько секунд. Давайте посмотрим как мы его можем ускорить

## Ускоряемся!

```python
import threading
import requests
import time

def ping(url):
    res = requests.get(url)
    print(f'{url}: {res.text}')

urls = [
    'http://httpstat.us/200',
    'http://httpstat.us/400',
    'http://httpstat.us/404',
    'http://httpstat.us/408',
    'http://httpstat.us/500',
    'http://httpstat.us/524'
]

start = time.time()
threads = []
for url in urls:
    thread = threading.Thread(target=ping, args=(url,))
    threads.append(thread)
    thread.start()
for thread in threads:
    thread.join()

print(f'Вариант с потоками занял у нас всего: {time.time() - start : .2f} секунд')
```
Незнаю как у вас, но у меня данный вариант работает в 6 раз быстрее, и это еще не потолок

## Делаем по красоте
Главная прелесть модуля Threadind в том, что мы можем создать класс и унаследовать его от threading.Tread и весь код в этом классе будет выполняться в отдельном потоке

Перепишем наш многопоточный вариант более удобным способом
```python

import threading
import requests

class MyThread(threading.Thread):
    def __init__(self, url):
        threading.Thread.__init__(self)
        self.url = url
        self.result = None

    def run(self):
        res = requests.get(self.url)
        self.result = f'{self.url}: {res.text}'
urls = [
    'http://httpstat.us/200',
    'http://httpstat.us/400',
    'http://httpstat.us/404',
    'http://httpstat.us/408',
    'http://httpstat.us/500',
    'http://httpstat.us/524'
]

start = time.time()

threads = [MyThread(url) for url in urls]
for thread in threads:
    thread.start()
for thread in threads:
    thread.join()
for thread in threads:
    print(thread.result)

print(f'Вариант с потоками занял у нас всего: {time.time() - start : .2f} секунд')
```
Несмотря на то что скорость неувеличилась, вариант с использованием класса гораздо чище и проще поддерживать

Как правилньо создать клас
 - Создаем класс, лол
 - Наследуемся от класса threading.Thread
 - В методе `__init__` вызываем конструктор родителя
 - Определяем метод run() в нем все будет обрабатываться в отедльном потоке
 
## Более поднобробно про потоки, или зачем мы пишем join()

Очень важно понимать, что потоки запускаются отдельно от основного потока, и поумолчанию основной поток не будет ждать других потоков
и после своего завершения грохнет все остальные дочерние потоки

Собственно пример:
```python
import threading
import requests

class MyThread(threading.Thread):
    def __init__(self, url):
        threading.Thread.__init__(self)
        self.url = url
        self.result = None

    def run(self):
        res = requests.get(self.url)
        self.result = f'{self.url}: {res.text}'
urls = [
    'http://httpstat.us/200?sleep=1000',
    'http://httpstat.us/400?sleep=1000',
    'http://httpstat.us/404?sleep=2000',
    'http://httpstat.us/500?sleep=200',
    'http://httpstat.us/408?sleep=3000',
    'http://httpstat.us/524?sleep=20000'
]


threads = [MyThread(url) for url in urls]
for thread in threads:
    thread.start()
for thread in threads:
    print(thread.result)
```
Запустив этот код, мы увидим, что программа поехала дальше и ей совершенно плевать на то закончили у нас потоки или нет

Метод join заставляет главный поток ждать пока остальные потоки закончат свое выполнение

Так же join принимает интовый параметр таймаута, что позволяет ждать некоторые потокие некоторое время, после которого программа поедет дальше

попробуйте передать 2 в коде выше и вы увидите как выполнятся все запросы кроме последних двух
## На последок, еще более удобный вариант

Плодить отдельный поток на каждый запрос (особенно если их несколько тыщ) не очень хорошая идея, по крайней мере на каком-то количестве потоков прирост застопорится, а программа будет жрать непомерно много

Для того чтоб решить эту проблемму создадим некий пул из потоков которые будут обрабатывть по 20 запросов за раз, и потом брать еще

Смотрим пример:
```python
import concurrent.futures
import requests

def process_request(url):
    result  = requests.get(url)
    print(result.status_code)
urls = [
    'http://httpstat.us/200?sleep=20',
    'http://httpstat.us/400?sleep=20',
    'http://httpstat.us/404?sleep=20',
    'http://httpstat.us/408?sleep=20'
]
with concurrent.futures.ThreadPoolExecutor(max_workers=20) as executor:
    futures = [executor.submit(process_request, i) for i in urls]
    for i, future in enumerate(concurrent.futures.as_completed(futures)):
        if future.result():
            pass
```
В этом примере мы воспользовались модулем concurrent.futures и его методом ThreadPoolExecutor

Данный метод принимает на вход определенное число потоков, и потом получает задачи, и начинает их выполнять порциями по числу потоков, данный метод очень лаконичный сам по себе и я рекомендую его использовать в случае если у вас есть множество потокобезопасных задач и вам необходимо их распараллелить

**Важно:** Осторожно используйте данный метод с вашими библиотеками, с библиотекой requests ничего страшного не будет, но нет гарантий что с другими библиотеками все будет так же хорошо
