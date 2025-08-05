Отчёт о выполнении тестового задания:
Развёртывание аналитического кластера OpenSearch


Техническое задание:
Разработать и развернуть систему для сбора, обработки и визуализации веб-логов 
с использованием следующего стека технологий:
– OpenSearch;
– OpenSearch Dashboards;
– Logstash.

Для начала выполнения задания было необходимо изучить перечисленные выше инструменты. После ознакомления с официальной документацией OpenSearch, а также со статьями и видеоматериалами я приступил к практической части.
Перед установкой и настройкой в задании было предложено выбрать метод развертывания. Я решил развернуть OpenSearch на виртуальной машине с Linux без использования Docker, но позже решил, что в рамках этого задания хочу познакомиться и разобраться с таким важным и неизвестным для меня инструментом как Docker, следовательно использовал свою гостевую Windows 10.

Установка и настройка:
1)	Для начала был установлен Docker Desktop для более наглядного и простого знакомства с программой;
2)	Установка и настройка OpenSearch
– выполнил команду: (docker pull bitnami/opensearch:latest) в PowerShell для загрузки образа OpenSearch. Далее создал сеть для последующего подключения других контейнеров: (docker network create app-tier --driver bridge);
– запустил контейнер: (docker run -d --name opensearch-server -p 9200:9200 -p 9300:9300 --network app-tier -v opensearch_data:/bitnami/opensearch bitnami/opensearch:latest) проверил его статус и логи;
– было необходимо настроить переменные окружения, такие как (opensearch_password, opensearch_cluster_name, opensearch_heap_size, opensearch_enable_security и т.д.). Пример настройки для opensearch_password: (docker run -d --name opensearch-server -p 9200:9200 -p 9300:9300 --network app-tier -e OPENSEARCH_PASSWORD=mysecurepassword -v opensearch_data:/bitnami/opensearch bitnami/opensearch:latest). Итогом настройки OpenSearch было успешное подключение к localhost:9200;
3)	Установка и настройка OpenSearch Dashboards
– выполнил команду для загрузки образа: (docker pull bitnami/opensearch-dashboards:latest). Важным шагом было запустить OpenSearch Dashboards с подключением к той же сети, которая была создана пунктом выше: (docker run -d --name opensearch-dashboards -p 5601:5601 `
  --network app-tier `
  -e OPENSEARCH_DASHBOARDS_OPENSEARCH_URL=http://opensearch-server:9200 `
  bitnami/opensearch-dashboards:latest);
 – после этих команл был успешно запущен localhost:5601.
 
4)	Установка, настройка и сбор данных Logstash 
– скачал официальный образ Logstash:( docker pull docker.elastic.co/logstash/logstash:8.12.0);
– создал следующую структуру папок: 
mkdir C:\docker\logstash
mkdir C:\docker\logstash\pipeline
mkdir C:\docker\logstash\data
– скачал датасет, https://www.kaggle.com/datasets/eliasdabbas/web-server-access-logs в котором логи соответствуют стандартному формату Apache Common Log Format (CLF), что полностью совмещается с будущим Grok-фильтром;
– поместил разархивированный access.log в C:\docker\logstash\data\;
– создал при помощи Notepad++ logstash.conf в C:\docker\logstash\pipeline\. Важными деталями являлось сохранять файл в UTF-8 без BOM и переводы строк – LF (Unix), а не CRLF (Windows);
– выделю команду (docker logs -f logstash), т.к. думаю, что ее применение было чаще остальных при выполнении данного задания, поскольку количество ошибок при конфигурации logstash.conf легко превысило десятки, а может и сотни. Об основных проблемах при настройке logstash.conf будет упомянуто в отдельном пункте;
– выполнил тестовый запрос к OpenSearch: (curl.exe -XGET "http://localhost:9200/_cat/indices?v" -u admin:admin), по итогам которого было найдено более 10,000 записей, создан индекс apache-logs-1995.07.01, а также такие поля как client, method, response, request, bytes, @timestamp корректно распарсяны.

Трудности конфигурации logstash.conf
Этап настройки logstash.conf вызвал небольшие затруднения при интеграции с другими узлами кластера.
Важно было понять структуру построения данного файла, и то, как он взаимодействует с другими компонентами кластера, а также основные плагины:
– File Input Plugin (читает логи из файла);
– Grok Filter Plugin (разбирает неструктурированные логи в поля, например, дату, IP, ошибку);
– OpenSearch Output Plugin (отправляет обработанные логи в OpenSearch).
Ошибки были различного типа, например, синтаксического: [2025-08-02T19:36:42,300][ERROR][logstash.agent ] Failed to execute action {:action=>LogStash::PipelineAction::Create/pipeline_id:main, :exception=>"LogStash::ConfigurationError", :message=>"Expected one of [ \\t\\r\\n], \"#\", \"input\", \"filter\", \"output\" at line 1, column 1 (byte 1)", :backtrace=>["/opt/bitnami/logstash/logstash-core/lib/logstash/compiler.rb:32:in `compile_imperative'", "org/logstash/execution/AbstractPipelineExt.java:285:in `initialize'", "org/logstash/execution/AbstractPipelineExt.java:223:in `initialize'", "/opt/bitnami/logstash/logstash-core/lib/logstash/java_pipeline.rb:47:in `initialize'", "org/jruby/RubyClass.java:950:in `new'", "/opt/bitnami/logstash/logstash-core/lib/logstash/pipeline_action/create.rb:50:in `execute'", "/opt/bitnami/logstash/logstash-core/lib/logstash/agent.rb:431:in `block in converge_state'"]}. 
Logstash ожидает, что конфигурационный файл будет содержать как минимум одну из секций: input, filter, output, как раз в этом логе ошибка Expected one of [ \t\r\n], "#", "input", "filter", "output" говорит, что файл либо пустой, либо содержит недопустимые символы в начале. 
Были ошибки ¬– конфликты портов, потому что Docker Desktop по умолчанию живет на порте 9600, как и logstash. Вот пример такого лога: [2025-08-02T19:46:01,079][INFO ][logstash.agent ] Successfully started Logstash API endpoint {:port=>9600, :ssl_enabled=>false} [2025-08-02T19:46:01,097][INFO ][logstash.runner ] Logstash shut down. [2025-08-02T19:46:01,109][FATAL][org.logstash.Logstash ] Logstash stopped processing because of an error: (SystemExit) exit. Решением этой проблемы стало изменение порта API Logstash.

Визуализация данных:
– в OpenSearch Dashboards был создан Index Pattern для того, чтобы Dashboards знал, где искать логи;
– создан ряд визуализаций (metric, pie, data table, line, vertical bar);
– настроены фильтры для выбора конкретной категории или элемента для анализа (например, ошибки 4хх/5xx можно отфильтровать, так: response:>=400);
– использована цветовая раскраска для улучшения восприятия;
– создан дашборд, в который были добавлены все ранее подготовленные визуализации с настройкой фильтров уже в рамках дашборда.








