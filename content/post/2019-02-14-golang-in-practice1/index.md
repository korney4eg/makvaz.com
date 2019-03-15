---
title: Go на практике. Фильтруем access-логи.
layout: post
archives: "2019"
tags: [golang, программирование, howto]
---
Не так давно ко мне пришла идея построить систему по анализу трафика вместо Google Analytics, Яндекс Метрики для того, чтобы как можно меньше внешних скриптов и трекеров использовалось на сайте. Будем [анализировать access-логи ](/2019/02/06/configuring-goaccess.html), но для начала хорошо бы их почистить.

Так как сейчас я изучаю Go(golang), то в этой статье попробуем создать утилиту командной строки(cli), которая вычистит "мусор" из логов web-сервера и эти "чистые" данные  можно будет использовать в goaccess.
<!--more-->

## Проблема

После [установки и настройки goaccess](/2019/02/06/configuring-goaccess.html) у меня получился такой график за декабрь:

![Посещения за месяц](/assets/img/parsing-access-logs-with-golang/blog-dashboard.png)

Цифры впечатляют. Но если пристально взглянуть на другие показатели, например, браузеры, с которых пришли пользователи (user agent), то видно, что не всё так хорошо.

![Список браузеров](/assets/img/parsing-access-logs-with-golang/user-agents.png)

Получается, что из **2800** запросов:

 - **426** запросов - от неизвестного user agent-a (браузера)
 - **834** запроса - от разных кроулеров и ботов (curl, wget, go http client, Firefox 5.0), из которых 20 - от Яндекс браузера и Avant браузера
 - **248** запросов - от разных агрегаторов RSS лент (Feedly)

В эти запросы включаются все картинки и скрипты. Мне же интереснее смотреть сколько было посещений на главной и в статьях (html страницы).

Формат логов у меня стандартный для nginx:

```
$remote_addr - $remote_user [$time_local] "$request" $status $body_bytes_sent "$http_referer" "$http_user_agent"
```

## Требования

Значит, нужно разработать утилиту, которая будет чистить логи от "мусора", а конкретно можно выразить в требованиях:

 1. читать логи из STDIN, выводить в STDOUT
    ```bash
    cat /var/log/nginx/access.log| access-log-filter | goaccess
    ```
 2. оставить только успешные запросы со статусом "200", `$status`
 1. оставить только запросы на главную страницу (путь "/") и на статьи (путь "*.html"), смотрим на `$request`
 3. игнорировать всякие кровлеры и неизвестные браузеры, фильтруем по `user_agent`
 4. анонимизация ip адресов (можно скинуть кому-нибудь логи и не боятся последствий), последнее число в `remote_addr` последнее число заменяется нулём.

## Реализация
Тоже самое можно написать и в bash при помощи `grep`, `aws`,`sed`, но со временем такой скрипт очень тяжело поддерживать, да и есть дополнительный повод улучшить свои навыки программирования на Go. Полный текст программы можно найти в самом конце статьи, если вам интересно почитать подробнее, то давайте приступим.

### 1. Разбиваем строку логов в словарь.

Первым делом напишем функцию, преобразующую строку лога в словарь.
Создаём файл `main.go`
```golang
package main

import (
	"regexp"
)

func ConvertLogLineToMap(logLine string) map[string]string {
	regexPattern := `(?P<remote_addr>\d+\.\d+\.\d+\.\d+) - -` +
		` \[(?P<time_local>[^\]]+)\] \"(?P<request>.*)\" (?P<status>[0-9]+)` +
		` (?P<body_bytes_sent>[0-9]+) \"(?P<http_referer>.*)\" \"` +
		`(?P<http_user_agent>.+)\"`
	re := regexp.MustCompile(regexPattern)
	parsedMap := make(map[string]string)
	match := re.FindStringSubmatch(logLine)
	if match == nil {
		return nil
	}
	for i, name := range re.SubexpNames() {
		if i != 0 {
			parsedMap[name] = match[i]
		}
	}
	return parsedMap
}


func main() {
}
```

Эта функция парсит строку
```
52.87.65.11 - - [02/Nov/2018:06:55:13 +0000] "GET /feed.xml HTTP/1.1" 200 9810 "-" "curl"
```
и преобразует её в словарь(map) вида:
```golang
parsedLine["remote_addr"] = "52.87.65.11"
parsedLine["remote_user"] = ""
parsedLine["time_local"] = "02/Nov/2018:06:55:13 +0000"
parsedLine["request"] = "GET /feed.xml HTTP/1.1"
parsedLine["status"] = "200"
parsedLine["body_bytes_sent"] = "10"
parsedLine["http_referer"] = "-"
parsedLine["http_user_agent"] = "curl"
``````

Это делается для того, чтобы легче было потом обрабатывать отдельные данные, а не парсить каждый раз всю строку.
Советую обратить внимание на [FindStringSubmatch](https://golang.org/pkg/regexp/#Regexp.FindStringSubmatch), он создаёт слайс(slice) c извлечёнными данными, а при помощи [SubexpNames](https://golang.org/pkg/regexp/#Regexp.SubexpNames) можно получить слайс из имён этих параметров. В цикле потом просто связываем два эти слайса в словарь, где ключом будет название поля(status, http_user_agent и т.д.), а значением распознанная информация.

Напишем несколько тестов в файле `main_test.go`:
```golang
package main

import (
	"testing"
)

func assertMaps(t *testing.T, got, want map[string]string) {
	t.Helper()
	for k, _ := range want {
		if got[k] != want[k] {
			t.Errorf("\nexpected value: %s\n     got value: %s", want[k], got[k])
		}
	}
}

func TestConvertLogLine(t *testing.T) {
	logLine := ""
	t.Run("Parsing normal log entry", func(t *testing.T) {
		logLine = `52.87.65.11 - - [02/Nov/2018:06:55:13 +0000] "GET /feed.xml HTTP/1.1" 200 9810 "-" "curl"`
		expectedResult := map[string]string{
			"remote_addr":     "52.87.65.11",
			"remote_user":     "",
			"time_local":      "02/Nov/2018:06:55:13 +0000",
			"request":         "GET /feed.xml HTTP/1.1",
			"status":          "200",
			"body_bytes_sent": "9810",
			"http_referer":    "-",
			"http_user_agent": "curl",
		}
		parsedLine := ConvertLogLineToMap(logLine)
		assertMaps(t, parsedLine, expectedResult)
	})

	t.Run("Parsing normal log entry with empty user agent", func(t *testing.T) {
		logLine = `190.128.131.6 - - [03/Nov/2018:23:25:08 +0000] "" 400 0 "-" "-"`
		expectedResult := map[string]string{
			"remote_addr":     "190.128.131.6",
			"remote_user":     "",
			"time_local":      "03/Nov/2018:23:25:08 +0000",
			"request":         "",
			"status":          "400",
			"body_bytes_sent": "0",
			"http_referer":    "-",
			"http_user_agent": "-",
		}
		parsedLine := ConvertLogLineToMap(logLine)
		assertMaps(t, parsedLine, expectedResult)
	})

	t.Run("Parsing strange line that should fail", func(t *testing.T) {
		logLine = `190.XXXX.XXXXX.XXX - - [03/Nov/2018:23:25:08 +0000] "" 400 0 "-" "-"`
		parsedLine := ConvertLogLineToMap(logLine)
		assertMaps(t, parsedLine, nil)
	})
}
```
В тестах мы прогоняем строки логов через нашу функцию и проверяем, что результат совпадает с ожидаемым.

Могу сказать, что написание тестов, хоть и отнимает время в начале, но зато существенно экономит его, когда хочется что-нибудь дописать. Рекммендую глянуть в сторону [TDD](https://ru.wikipedia.org/wiki/%D0%A0%D0%B0%D0%B7%D1%80%D0%B0%D0%B1%D0%BE%D1%82%D0%BA%D0%B0_%D1%87%D0%B5%D1%80%D0%B5%D0%B7_%D1%82%D0%B5%D1%81%D1%82%D0%B8%D1%80%D0%BE%D0%B2%D0%B0%D0%BD%D0%B8%D0%B5)

Проверить, что все тесты работают можно командой:
```bash
$ go test
PASS
ok      _/Users/myuser/projects/access-log-filter    0.009s
```

### 2. Преобразуем словарь в строку логов
Эта операция обратна предыдущей. Нужна она чтобы, при изменении каких-то данных(анонимизировали ip адрес), можно сформировать  правильную строку лога.

В файл `main.go` добавим функцию:

```golang
func ConvertMapToLogLine(parsedLine map[string]string) string {
	remote_addr := parsedLine["remote_addr"]
	time_local := parsedLine["time_local"]
	request := parsedLine["request"]
	status := parsedLine["status"]
	body_bytes_sent := parsedLine["body_bytes_sent"]
	http_referer := parsedLine["http_referer"]
	http_user_agent := parsedLine["http_user_agent"]
	return fmt.Sprintf(`%s - - [%s] "%s" %s %s "%s" "%s"`, remote_addr,
		time_local, request, status, body_bytes_sent, http_referer,
		http_user_agent)
}
```

В секцию **import** нужно дописать модуль "fmt".

Допишем тесты для этой функции в `main_test.go`:

```golang
func TestConvertMapToLogLine(t *testing.T) {
	t.Run("Parsing normal map entry", func(t *testing.T) {
		expectedLogLine := `52.87.65.11 - - [02/Nov/2018:06:55:13 +0000] "GET /feed.xml HTTP/1.1" 200 9810 "-" "curl"`
		parsedMap := map[string]string{
			"remote_addr":     "52.87.65.11",
			"remote_user":     "",
			"time_local":      "02/Nov/2018:06:55:13 +0000",
			"request":         "GET /feed.xml HTTP/1.1",
			"status":          "200",
			"body_bytes_sent": "9810",
			"http_referer":    "-",
			"http_user_agent": "curl",
		}
		generatedLogLine := ConvertMapToLogLine(parsedMap)
		if expectedLogLine != generatedLogLine {
			t.Errorf("Log line did not match!.\nExpected: '%s'\n      Got:'%s'", expectedLogLine, generatedLogLine)
		}
	})
	t.Run("Parsing normal map entry with anonimized ip", func(t *testing.T) {
		expectedLogLine := `190.128.131.6 - - [03/Nov/2018:23:25:08 +0000] "" 400 0 "-" "-"`
		parsedMap := map[string]string{
			"remote_addr":     "190.128.131.6",
			"remote_user":     "",
			"time_local":      "03/Nov/2018:23:25:08 +0000",
			"request":         "",
			"status":          "400",
			"body_bytes_sent": "0",
			"http_referer":    "-",
			"http_user_agent": "-",
		}
		generatedLogLine := ConvertMapToLogLine(parsedMap)
		if expectedLogLine != generatedLogLine {
			t.Errorf("Log line did not match!.\nExpected: '%s'\n      Got:'%s'", expectedLogLine, generatedLogLine)
		}
	})
}
```

### 3. Чтение из STDIN

Дописываем в функцию `main` в  файле `main.go`

```golang
func main() {
    scanner := bufio.NewScanner(os.Stdin)
    for scanner.Scan() {
        fmt.Println(scanner.Text())
    }

    if err := scanner.Err(); err != nil {
        fmt.Fprintln(os.Stderr, "error:", err)
        os.Exit(1)
    }
}
```

А в **import**:

```golang
    "bufio"
    "fmt"
    "os"
    "strings"
```

Для работы с вводом/выводом будем использовать объект типа [ Scanner ](https://golang.org/pkg/text/scanner/#Scanner) из пакета `bufio`, у которого есть функция [Scan()](https://golang.org/pkg/bufio/#Scanner.Scan). В качестве ввода передаём `STDIN`, в Go берётся из пакета `os`. Затем построчно читаем всё, что пришло на `STDIN`.

Чтобы посмотреть в текстовом виде, что пришло на ввод воспользуемся функцией [Text](https://golang.org/pkg/bufio/#Scanner.Text).

Если происходит какая-нибудь ошибка, например мы оборвали процесс чтения Ctrl-c, просто выводим эту ошибку и выходим.

Получается программа что читает, то и пишет.

Теперь, убедимся, что программа работает как надо:
```bash
$ cat old_access.log |go run main.go |wc -l
6838
$ cat old_access.log |wc -l
6838
$ cat old_access.log |go run main.go |head -n5
18.207.127.0 - - [01/Jan/2019:06:27:09 +0000] "GET /twe-feed.xml HTTP/1.1" 200 16988 "-" "curl"
164.52.24.0 - - [01/Jan/2019:06:39:54 +0000] "GET / HTTP/1.1" 400 280 "-" "Mozilla/5.0 (Windows NT 6.1; WOW64; rv:52.0) Gecko/20100101 Firefox/52.0"
157.55.39.0 - - [01/Jan/2019:07:17:12 +0000] "GET / HTTP/1.1" 200 4402 "-" "Mozilla/5.0 (compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm)"
3.82.126.0 - - [01/Jan/2019:07:57:15 +0000] "GET /twe-feed.xml HTTP/1.1" 200 16988 "-" "curl"
54.157.251.0 - - [01/Jan/2019:08:27:18 +0000] "GET /twe-feed.xml HTTP/1.1" 200 16988 "-" "curl"
$ cat old_access.log |head -n5
18.207.127.0 - - [01/Jan/2019:06:27:09 +0000] "GET /twe-feed.xml HTTP/1.1" 200 16988 "-" "curl"
164.52.24.0 - - [01/Jan/2019:06:39:54 +0000] "GET / HTTP/1.1" 400 280 "-" "Mozilla/5.0 (Windows NT 6.1; WOW64; rv:52.0) Gecko/20100101 Firefox/52.0"
157.55.39.0 - - [01/Jan/2019:07:17:12 +0000] "GET / HTTP/1.1" 200 4402 "-" "Mozilla/5.0 (compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm)"
3.82.126.0 - - [01/Jan/2019:07:57:15 +0000] "GET /twe-feed.xml HTTP/1.1" 200 16988 "-" "curl"
54.157.251.0 - - [01/Jan/2019:08:27:18 +0000] "GET /twe-feed.xml HTTP/1.1" 200 16988 "-" "curl"
```
Похоже не обманывает. Идём дальше по списку.

### 4. Фильтруем

Для фильтрации создадим функцию `matchAllRequirements`, которая принимает на вход словарь и проверяет все наши условия и возвращает `true` или `false`. В Go switch case работает несколько иначе, чем в других языках, а именно можно не быть привязанным к одной переменной и проверять разные условия.

```golang
func matchAllRequirements(parsedLine map[string]string) bool {
	request := regexp.MustCompile(`GET (.+\.html|/)(\?.*)? HTTP\/1\.[10]`)
	http_user_agent := regexp.MustCompile(`.*([Bb]ot|vkShare|Google-AMPHTML|feedly|[cC]rawler|[Pp]arser|curl|-).*`)
	switch {
	case parsedLine["status"] != "200":
		return false
	case !request.MatchString(parsedLine["request"]):
		return false
	case http_user_agent.MatchString(parsedLine["http_user_agent"]):
		return false
	default:
		return true
	}
}
```

Вот так легко и лаконично можно проверить по всем параметрам, причём не обязательно использовать регулярные выражения, как мы сделали для проверки статус кода(status)

В функцию `main` добавим вызов проверочной фунции:
```golang
func main() {
	scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		parsedLine := ConvertLogLineToMap(scanner.Text())
		if matchAllRequirements(parsedLine) == false {
			continue
		}
		fmt.Println(scanner.Text())
	}
}
```

Тесты в `main_test.go`:
```golang
func TestLogLineOK(t *testing.T) {
	cases := map[string]bool{
		`159.203.112.0 - - [05/Jan/2019:22:35:12 +0000] "GET / HTTP/1.0" 200 13444 "-" "Mozilla/5.0 (compatible; NetcraftSurveyAgent/1.0; +info@netcraft.com)"`:                                                                          true,
		`213.138.93.0 - - [05/Jan/2019:23:36:41 +0000] "GET /2018/08/25/aws-certification-preparation.html HTTP/1.1" 200 7779 "https://www.google.com/" "Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:64.0) Gecko/20100101 Firefox/64.0"`: true,
		`213.138.93.0 - - [05/Jan/2019:23:36:42 +0000] "GET / HTTP/1.1" 200 4402 "https://makvaz.com/2018/08/25/aws-certification-preparation.html" "Mozilla/5.0 (Windows NT 6.1; Win64; x64; rv:64.0) Gecko/20100101 Firefox/64.0"`:     true,
		`199.16.157.0 - - [02/Jan/2019:11:54:10 +0000] "GET /2018/09/25/effective-devops.html HTTP/1.1" 200 10399 "-" "Twitterbot/1.0"`:                                                                                                 false,
		`199.16.157.0 - - [02/Jan/2019:11:54:10 +0000] "GET /assets/img/header-pic.jpeg HTTP/1.1" 200 696591 "-" "Twitterbot/1.0"`:                                                                                                      false,
		`54.36.148.0 - - [02/Jan/2019:12:06:17 +0000] "GET /blog/page3/ HTTP/1.1" 200 3586 "-" "Mozilla/5.0 (compatible; AhrefsBot/6.1; +http://ahrefs.com/robot/)"`:                                                                    false,
		`125.212.217.0 - - [02/Jan/2019:16:03:08 +0000] "GET /robots.txt HTTP/1.1" 200 40 "-" "-"`:                                                                                                                                      false,
		`40.77.167.0 - - [02/Jan/2019:16:03:59 +0000] "GET /about/ HTTP/1.1" 200 3193 "-" "Mozilla/5.0 (compatible; bingbot/2.0; +http://www.bing.com/bingbot.htm)"`:                                                                    false,
	}
	for testCase, expectedResult := range cases {
		parsedLine := ConvertLogLineToMap(testCase)
		actualResult := matchAllRequirements(parsedLine)
		if actualResult != expectedResult {
			t.Errorf("result for line: '%s'\n returned wrong value: %t, expected: %t", testCase, actualResult, expectedResult)

		}
	}
}
```

Посмотрим, сколько осталось записей после отсева.

```bash
$ cat old_access.log |wc -l
6838
$ cat old_access.log |go run main.go |wc -l
458
```
Отсеялось больше 90% всего трафика, зато честно и понятно.

### 5. Добавляем анонимизацию ip адресов.

Чтобы не выдавать настоящие ip адреса посетителей, можно их анонимизировать. Для этого в файле `main.go` добавим функцию:
```golang
func AnonymizeIp(ip string) string {
	return ip[:strings.LastIndex(ip, ".")+1] + "0"
}
```
Вместо последнего числа(октета) в ip адресе записываем 0.

И куда без тестов в `main_test.go`:
```golang
func TestAnonymizeIp(t *testing.T) {
	cases := map[string]string{
		"5.255.250.183":   "5.255.250.0",
		"141.8.144.9":     "141.8.144.0",
		"178.154.244.157": "178.154.244.0",
		"40.77.167.89":    "40.77.167.0",
		"66.249.69.70":    "66.249.69.0",
	}
	for testCase, expectedResult := range cases {
		actualResult := AnonymizeIp(testCase)
		if actualResult != expectedResult {
			t.Errorf("result for line: '%s'\n returned wrong value: %s, expected: %s", testCase, actualResult, expectedResult)
		}
	}
}
```

## Заключение.

По итогу получился вот такой график:

![Подчищенные посещения за месяц](/assets/img/parsing-access-logs-with-golang/filtered-blog-dashboard.png)

Сегодня мы написали достаточно простую программу по очищению логов. Её можно использовать для генерации отчётов в связке с goaccess на сервере.

Что можно улучшить:

 - добавить фильтрацию по дате/диапазону дат
 - иметь возможность читать файл целиком а не из `STDIN`
 - Фильтры подгружать из внешнего файла
 - улучшить читаемость тестов
 - Строить графики )

Если вы хотите сами поупражняться, то вот вам [мой лог](/assets/other/access.log.zip)

## Полезные ссылки
 1. [ Документация по Go ]( https://golang.org/doc/ )
 2. [ Курсы по Go от mail.ru ]( https://www.coursera.org/learn/golang-webservices-1 )
 3. [ Интересный ресурс по Go ]( https://4gophers.ru/articles )

## Полный текст программы

```golang
package main
import (
	"bufio"
	"fmt"
	"os"
	"regexp"
	"sort"
	"strings"
)
func ConvertLogLineToMap(logLine string) map[string]string {
	regexPattern := `(?P<remote_addr>\d+\.\d+\.\d+\.\d+) - -` +
		` \[(?P<time_local>[^\]]+)\] \"(?P<request>.*)\" (?P<status>[0-9]+)` +
		` (?P<body_bytes_sent>[0-9]+) \"(?P<http_referer>.*)\" \"` +
		`(?P<http_user_agent>.+)\"`
	re := regexp.MustCompile(regexPattern)
	parsedMap := make(map[string]string)
	match := re.FindStringSubmatch(logLine)
	if match == nil {
		return nil
	}
	for i, name := range re.SubexpNames() {
		if i != 0 {
			parsedMap[name] = match[i]
		}
	}
	return parsedMap
}
func matchAllRequirements(parsedLine map[string]string) bool {
	request := regexp.MustCompile(`GET (.+\.html|/)(\?.*)? HTTP\/1\.[10]`)
	http_user_agent := regexp.MustCompile(`.*([Bb]ot|vkShare|Google-AMPHTML|feedly|[cC]rawler|[Pp]arser|curl|-).*`)
	switch {
	case parsedLine["status"] != "200":
		return false
	case !request.MatchString(parsedLine["request"]):
		return false
	case http_user_agent.MatchString(parsedLine["http_user_agent"]):
		return false
	default:
		return true
	}
}
func sortByPopularity(metric map[string]int) {
	n := map[int][]string{}
	var a []int
	for k, v := range metric {
		n[v] = append(n[v], k)
	}
	for k := range n {
		a = append(a, k)
	}
	sort.Sort(sort.IntSlice(a))
	for _, k := range a {
		for _, s := range n[k] {
			fmt.Printf("%d - %s\n", k, s)
		}
	}
}
func AnonymizeIp(ip string) string {
	return ip[:strings.LastIndex(ip, ".")+1] + "0"
}
func ConvertMapToLogLine(parsedLine map[string]string) string {
	remote_addr := parsedLine["remote_addr"]
	time_local := parsedLine["time_local"]
	request := parsedLine["request"]
	status := parsedLine["status"]
	body_bytes_sent := parsedLine["body_bytes_sent"]
	http_referer := parsedLine["http_referer"]
	http_user_agent := parsedLine["http_user_agent"]
	return fmt.Sprintf(`%s - - [%s] "%s" %s %s "%s" "%s"`, remote_addr,
		time_local, request, status, body_bytes_sent, http_referer,
		http_user_agent)
}
func main() {
	scanner := bufio.NewScanner(os.Stdin)
	for scanner.Scan() {
		parsedLine := ConvertLogLineToMap(scanner.Text())
		parsedLine["remote_addr"] = AnonymizeIp(parsedLine["remote_addr"])
		fmt.Println(ConvertMapToLogLine(parsedLine))
	}
	if err := scanner.Err(); err != nil {
		fmt.Fprintln(os.Stderr, "error:", err)
		os.Exit(1)
	}
}
```
