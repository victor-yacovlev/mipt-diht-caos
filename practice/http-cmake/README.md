# Протокол HTTP, библиотека cURL и система сборки CMake

## Протокол HTTP

### Общие сведения

Протокол HTTP используется преимущественно браузерами для загрузки и отправки контента. Кроме того, благодаря своей простоте и универсальности, он часто используется как высокоуровневый протокол клиент-серверного взаимодействия.

Большинтсво серверов работают с версией протокола `HTTP/1.1`, который подразумевает взаимодействие в текстовом виде через TCP-сокет. Клиент отправляет на сервер текстовый запрос, который содержит:
 * Команду запроса
 * Заголовки запроса
 * Пустую строку - признак окончания заголовков запроса
 * Передаваемые данные, если они подразумеваются

В ответ сервер должен отправить:
 * Статус обработки запроса
 * Заголовки ответа
 * Пустую строку - признак окончания заголовков ответа
 * Передаваемые данные, если они подразумеваются

Стандартным портом для `HTTP` является порт 80, для `HTTPS` - порт с номером 443, но это жёстко не регламентировано, и при необходимости номер порта может быть любым.

### Основные команды и заголовки HTTP

 * `GET` - получить содержимое по указанному URL;
 * `HEAD` - получить только метаинформацию (заголовки) по указанному URL, но не содержимое;
 * `POST` - отправить данные на сервер и получить ответ.

Кроме основных команд, в протоколе HTTP можно определять произвольные дополнительные команды в текстовом виде (естественно, для этого потребуется поддержка как со стороны сервера, так и клиента). Например, расширение WebDAV протокола HTTP, предназначенное для передачи файлов, дополнительно определяет команды `PUT`, `DELETE`, `MKCOL`, `COPY`, `MOVE`.

Заголовки - это строки вида `ключ: значение`, определяющие дополнительную метаинформацию запроса или ответа.

По стандарту `HTTP/1.1`, в любом запросе должен быть как минимум один заголовок - `Host`, определяющий имя сервера. Это связано с тем, что с одним IP-адресом, на котором работает HTTP-сервер, может быть связано много доменных имен.

Полный список заголовков можно посмотреть [в Википедии](https://en.wikipedia.org/wiki/List_of_HTTP_header_fields).

Пример взаимодействия:
```
$ telnet ejudge.atp-fivt.org
$ telnet ejudge.atp-fivt.org 80
Trying 87.251.82.74...
Connected to ejudge.atp-fivt.org.
Escape character is '^]'.
GET / HTTP/1.1
Host: ejudge.atp-fivt.org

HTTP/1.1 200 OK
Server: nginx/1.14.0 (Ubuntu)
Date: Tue, 23 Apr 2019 21:18:43 GMT
Content-Type: text/html; charset=UTF-8
Content-Length: 4383
Connection: keep-alive
Last-Modified: Mon, 04 Feb 2019 17:01:28 GMT
ETag: "111f-58114719b3ca3"
Accept-Ranges: bytes

<html>
  <head>
    <meta charset="utf-8"/>
    <title>АКОС ФИВТ МФТИ</title>
  </head>
...
```

### Протокол HTTPS

Протокол HTTP**S** - это реализация протокола HTTP поверх дополнительного уровня SSL, который, в свою очередь работает через TCP-сокет. На уровне SSL осуществляется проверка сертификата сервера и обмен ключами шифрования. После этого - начинается обычное взаимодействие по протоколу HTTP в текстовом виде, но это заимодейтвие передается по сети в зашифрованном виде.

Аналогом `telnet` для работы поверх SSL является инструмент `s_client` из состава OpenSSL:

```
$ openssl s_client -connect yandex.ru:443
```

### Утилита cURL

Универсальным инструментом для взаимодействия по HTTP в Linux считается [curl](https://curl.haxx.se), которая входит в базовый состав всех дистрибутивов. Работает не только по протоколу HTTP, но и HTTPS.

Основные опции `curl`:
 * `-v` - отобразить взаимодействие по протоколу HTTP;
 * `-X КОМАНДА` - отправить вместо `GET` произвольную текстовую команду в запросе;
 * `-H "Ключ: значение"` - отправить дополнительный заголовок в запросе; таких опций может быть несколько;
 * `--data-binary "какой-то текст"` - отправить строку в качестве данных (например, для `POST`);
 * `--data-binary @имя_файла` - отправить в качестве данных содержимое указанного файла.

## Библиотека `libcurl`

У утилиты `curl` есть программный API, который можно использовать в качестве библиотеки, не запуская отдельный процесс.

API состоит из двух частей: полнофункциональный асинхронный интерфейс (`multi`), и упрощённый с блокирующим вводом-выводом (`easy`).

Пример использования упрощённого интерфейса:
```
#include <curl/curl.h>

CURL *curl = curl_easy_init();
if(curl) {
  CURLcode res;
  curl_easy_setopt(curl, CURLOPT_URL, "http://example.com");
  res = curl_easy_perform(curl);
  curl_easy_cleanup(curl);
}
```

Этот код эквивалентен команде
```
$ curl http://example.com
```

Дополнительные параметры, эквивалентные отдельным опциям команды `curl`, определяются функцией [`curl_easy_setopt`](https://curl.haxx.se/libcurl/c/curl_easy_setopt.html).

Выполнение HTTP-запроса приводит к записи результата на стандартный поток вывода, но обычно бывает нужно получить данные для дальнейшей обработки.

Это делается установкой одной из callback-функций, которая ответственна за вывод:

```
#include <curl/curl.h>

typedef struct {
  char   *data;
  size_t length;
} buffer_t;

static size_t  
callback_function(
            char *ptr, // буфер с прочитанными данными
            size_t chunk_size, // размер фрагмента данных
            size_t nmemb, // количество фрагментов данных
            void *user_data // произвольные данные пользователя
            )
{
  buffer_t *buffer = user_data;
  size_t total_size = chunk_size * nmemb;

  // в предположении, что достаточно места
  memcpy(buffer->data, ptr, total_size);
  buffer->length += total_size;
  return total_size;
}            

int main(int argc, char *argv[]) {
  CURL *curl = curl_easy_init();
  if(curl) {
    CURLcode res;

    // регистрация callback-функции записи
    curl_easy_setopt(curl, CURLOPT_WRITEFUNCTION, callback_function);

    // указатель &buffer будет передан в callback-функцию
    // параметром void *user_data
    buffer_t buffer;
    buffer.data = calloc(100*1024*1024, 1);
    buffer.length = 0;
    curl_easy_setopt(curl, CURLOPT_WRITEDATA, &buffer);

    curl_easy_setopt(curl, CURLOPT_URL, "http://example.com");
    res = curl_easy_perform(curl);

    // дальше можно что-то делать с данными,
    // прочитанными в buffer

    free(buffer.data);
    curl_easy_cleanup(curl);
  }
}


```

## Система сборки `cmake`

Использование сторонних библиотек усложняет процесс воспроизводимости сборки. В случае, когда целевая операционная система одна, это не доставляет особых проблем и достаточно простого `Makefile`, но если предполагается разработка кросс-платформенного продукта, то возникают неоднозначности:
 * какой компилятор используется для сборки (gcc, clang, cl.exe);
 * расположение include-файлов библиотек для компиляции;
 * рссположение файлов библиотек для компоновки.

По этой причине часто практикуется не распространение `Makefile`, написанного для конкретного стека инструментов, а его генерация по декларативному описанию в процессе сборки: `./configure`, `qmake`, или `cmake`.

Наиболее гибкой системой сборки, и при этом относительно простой, является [CMake](https://cmake.org/cmake/help/v3.2/), которая реализована с поддержкой не только UNIX-подобных систем, но и Windows.

Описание проекта находится в файле `CMakeLists.txt` и имеет примерно следующий вид:

```
# признак того, что это файл для cmake
# номер версии - это минимально требуемый для сборки проекта
# не стоит злоупотребять указанием самой свежей версии
# CMake, поскольку в консервативных Linux-дистрибутивах
# может быть что-то более старое
cmake_minimum_required(VERSION 3.2)

# имя проекта - не обязательно, обычно используется IDE
project(my_great_project)

# команда `set` устанавливает значение переменной
# некоторые переменные (их имена начинаются с CMAKE_)
# имеют специальное значение

# дополнительные опции компилятора Си
set(CMAKE_C_FLAGS "-std=gnu11")

# дополнительные опции компилятора C++
set(CMAKE_CXX_FLAGS "-std=gnu14")

# в переменной SOURCES будет храниться список файлов;
# если файлов не много, то можно этого не делать,
# но некоторым IDE это требуется для навигации по проекту
set(SOURCES
  file1.c
  file2.cpp
  file3.cpp
)

# добавление цели для сборки - бинарного файла;
# синтаксис ${...} означает использование значения
# переменной, которая в данном примере будет раскрыта
# в список файлов, из которых собирается программа
add_executable(my_cool_program ${SOURCES})
```

Для сборки CMake-проекта необходимо выполнить две стадии:
 1. Сгенерировать `Makefile`из `CMakeLists.txt`
 2. Собрать проект обычнычным инструментом `make`.

Обычно в процессе генерации `Makefile` и при сборке проекта создается много временных файлов. По этой причине сборку принято проводить в отдельном каталоге, - чтобы не засорять каталог с исходными текстами.

```
$ mkdir build     # создаем каталог для сборки
$ cd build        # переходим в него
$ cmake ../       # генерируем Makefile
                  # аргумент cmake - это каталог, который
                  # содержит файл CMakeLists.txt
$ make            # запуск компиляции                   
```

Для многих OpenSource библиотек в стандартной поставке CMake уже готовы модули поддержки, которые выполняют поиск библиотеки. В случае c UNIX этот поиск осуществляется с помощью запуска команд конфигурации, либо проверки различных вариантов написания имен файлов в `/usr/include` и `/usr/lib`. Для Windows просматривается системный реестр.

Список поддерживаемых библиотек можно найти в поставке CMake, для Linux это может быть каталог (в разных дистрибутивах они разные) `/usr/share/cmake/Modules`. Все файлы модулей имеют название `FindИМЯБИБЛИОТЕКИ.cmake`.

Подключение библиотеки, которая поддерживается "из коробки", осуществляется с помощью команды `find_package`. В случае, если необходимые файлы присутствуют, то определяются переменные `ИМЯБИБЛИОТЕКИ_INCLUDE_DIRS` и `ИМЯБИБЛИОТЕКИ_LIBRARIES`.

Пример для curl:
```
# найти библиотеку CURL; опция REQUIRED означает,
# что библиотека является обязательной для сборки проекта,
# и если необходимые файлы не будут найдены, cmake
# завершит работу с ошибкой
find_package(CURL REQUIRED)

# добавляет в список каталогов, которые превратятся в
# опции -I компилятора все каталоги, которые перечислены
# в переменной CURL_INCLUDE_DIRECTORIES
include_directories(${CURL_INCLUDE_DIRECTORIES})

add_executable(my_cool_program ${SOURCES})

# для цели my_cool_program указываем библиотеки, с которыми
# программа будет слинкована
target_link_libraries(my_cool_program ${CURL_LIBRARIES})
```