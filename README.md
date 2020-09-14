## WSO2 инструкция
### Установка

### Общие понятия
#### Медиаторы
Выполнение обработки данных в WSO2 происходит по шагам. За каждый шаг отвечает определённый медиатор.

Медиатор - обработчик данных. 

Данные с предыдущего шага передаются на следующий. Эти данные могут быть в 2 форматах XML и JSON.
```xml
<payloadFactory media-type="xml">
    <format>
        <soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:web="http://webservices.ods.fors.ru/">
            <soapenv:Header/>
            <soapenv:Body>
                <web:getOdhPassport>
                    <odhId xmlns="">$1</odhId>
                </web:getOdhPassport>
            </soapenv:Body>
        </soapenv:Envelope>
    </format>
    <args>
        <arg evaluator="xml" expression="//odh/id/text()"/>
    </args>
</payloadFactory>
```
В данном примере формируется тело запрос для WSDL сервиса. При этом из входящего XML извлекается id по xpath и
подставляется в выходной XML.

#### Переменные
Если необходимо сохранить какие-то данные для использования в нескольких шагах существуют свойства.
```xml
<property expression="json-eval($.requestId)" name="requestId" scope="default" type="STRING"/>
```
В данном примере за входящего JSON извлекается requestId и записывается в свойство requestId с типом данных - строка
Далее это свойство можно использовать например в описанном выше payloadFactory медиаторе добавив
```xml
<arg evaluator="xml" expression="get-property('requestId')"/>
```

#### API
API позволяет делать запросы извне по HTTP. Для создания API необходимо добавить файл src/main/synapse-config/api/{api name}.xml с 
следующим содержимым:
```xml
<?xml version="1.0" encoding="UTF-8"?>
<api context="/myapi" name="myAPI" xmlns="http://ws.apache.org/ns/synapse">
    
</api>
```
и в файле artifact.xml добавить artifact слудующим образом
```xml
<?xml version="1.0" encoding="UTF-8"?><artifacts>
    <artifact name="myAPI" groupId="ru.my.wso2.integration.api" version="1.0.0" type="synapse/api" serverRole="EnterpriseServiceBus">
        <file>src/main/synapse-config/api/myAPI.xml</file>
    </artifact>
</artifacts>
```

Таким образом мы создадим API доступное по адресу {адрес wso2}/myapi

Адрес WSO2 по умолчанию - http://hostname:8290

Далее чтобы добавить ресурс доступный по данному API необходимо его добавить
```xml
<?xml version="1.0" encoding="UTF-8"?>
<api context="/myapi" name="myAPI" xmlns="http://ws.apache.org/ns/synapse">
    <resource methods="POST" uri-template="myresource">
    </resource>
</api>
```
Этот ресурс будет доступен как POST {адрес wso2}/myapi/myresource

При обработке запросов к API могут использоваться 3 последовательности шагов:
- входящая последовательность inSequence
- исходящая последовательность outSequence
- последовательность обработки ошибок faultSequence

Обязательной является только входящая последовательность.
```xml
<?xml version="1.0" encoding="UTF-8"?>
<api context="/myapi" name="myAPI" xmlns="http://ws.apache.org/ns/synapse">
    <resource methods="POST" uri-template="myresource">
        <inSequence></inSequence>
        <outSequence/>
        <faultSequence/>
    </resource>
</api>
```

### Выполнение запроса к удалённому ресурсу
Запрос к удаленному сервису состоит из следующих шагов
* создать тело запроса (необязательно)
* выставить заголовки (необязательно)
* описать endpoint на который сделан запрос
* сделать запрос

#### Тело запроса
Для формирования тела запроса используется медиатор payload factory. С помощью него можно описать тело запроса в форматах
JSON и XML. 

#### Заголовки запроса
Для того чтобы выствить заголовок запроса применяется медиатор header
```xml
<header name="Content-Type" scope="transport" value="application/json; charset=UTF-8"/>
```

#### Endpoint
Endpoint может быть описан как при испоьзовании так и заранее. Например для вызова WSDL сервисов имеет смысл описать 
endpoint отдельно так как запросы к разным методам будут отличться только телом запроса.

Пример описания endpoint для WSDL сервиса
```xml
<?xml version="1.0" encoding="UTF-8"?>
<endpoint name="ODS_WSDL_EP" xmlns="http://ws.apache.org/ns/synapse">
    <wsdl port="DictionaryServiceImplPort" service="dictionaryService" uri="http://ods.mos.ru/webservices/dictionaryService?wsdl">
        <suspendOnFailure>
            <initialDuration>-1</initialDuration>
            <progressionFactor>1.0</progressionFactor>
        </suspendOnFailure>
        <markForSuspension>
            <retriesBeforeSuspension>0</retriesBeforeSuspension>
        </markForSuspension>
    </wsdl>
</endpoint>
```
Этот endpoint должен быть в файле src/main/synapse-config/endpoints/ODH_WSDL.xml
и в файле artifact.xml должен быть следующий artifact
```xml
<?xml version="1.0" encoding="UTF-8"?><artifacts>
    <artifact name="ODS_WSDL_EP" groupId="ru.my.wso2.integration" version="1.0.0" type="synapse/endpoint" serverRole="EnterpriseServiceBus">
            <file>src/main/synapse-config/endpoints/ODS_WSDL_EP.xml</file>
        </artifact>
</artifacts>
``` 

В даном примере параметры port и service берутся из схемы описания WSDL сервиса

Пример описания endpoint при использовании
```xml
<send>
    <endpoint>
        <http method="post" uri-template="http://localhost:8080">
            <suspendOnFailure>
                <initialDuration>-1</initialDuration>
                <progressionFactor>1</progressionFactor>
            </suspendOnFailure>
            <markForSuspension>
                <retriesBeforeSuspension>0</retriesBeforeSuspension>
            </markForSuspension>
        </http>
    </endpoint>
</send>
```
В данном случае endpoint описывается внутри медиатора отправки send

В завершении разговора о endpoint следует сказать о endpoint для запроса на динамический адрес. 
```xml
<endpoint>
    <http method="post" uri-template="{uri.var.endpoint}">
        <suspendOnFailure>
            <initialDuration>-1</initialDuration>
            <progressionFactor>1</progressionFactor>
        </suspendOnFailure>
        <markForSuspension>
            <retriesBeforeSuspension>0</retriesBeforeSuspension>
        </markForSuspension>
    </http>
</endpoint>
```
В данном примере url находится в переменной uri.var.endpoint

**!!!ВАЖНО** для использования в шаблоке url endpointa название переменной должно начинаться на uri.var

#### Вызов
Для того чтобы сделать сам запрос на удалённый ресурс существует 2 медиатора:
- call
- send

Медиатор call вызывает endpoint и результат передаёт в слудующий медиатор последовательности.
```xml
<call>
    <endpoint key="MY_ENDPOINT"/>
</call>
```

Медиатор send вызывает endpoint и завершает цепочку исполнения в текущем sequence
```xml
<send>
    <endpoint key="MY_ENDPOINT"/>
</send>
```

Например мы можем в **inSequence** подготовить данные и сделать запрос используя **send** медиатор, а в **outSequence** на основе
полученного результата запросить дополнительные данные используя **call** медиатор.   

### Маппинг XML ответа в JSON (может должен быть не здесь)
Вернуть ответ в виде json можно 3 способами:
- задать media-type **json**  для ответа
- xslt преобразование
- data mapping медиатор

В данном примере мы описываем ответ в формате **json** с использованием media-type
```xml
<payloadFactory media-type="json">
    <format>{
        "requestId": "$1",
        "endpoint": "$2"	
    }</format>
    <args>
        <arg evaluator="xml" expression="get-property('requestId')"/>
        <arg evaluator="xml" expression="get-property('endpoint')"/>
    </args>
</payloadFactory>
```

### Циклы и агрегация
Для работы массивами существует 2 медиатора:
- **iterate** - проход по масиву 
- **aggregate** - соединение результатов iterate

Данные медиаторы используются совместно.

#### iterate
Медитор iterate имеет следующие свойства
- expression - выражение позволяющее получить массив
- sequential - обрабатывать части последовательно. Если true, то выполнение займёт больше времени, но порядок результатов
в исходящем массиве будет соответствовать входящему. Иначе выполнение будет параллельным и порядок не сохранится а время 
выполнения снижится.

```xml
<iterate expression="//odh[position()&lt;5]" sequential="true">
        <target>
            <sequence>
            </sequence>
        </target>
</iterate>
``` 
Внутки iterate описывается target/sequence. Здесь перечисляются все необходимые шаги по обработке элемента массива.

Медиаторы iterate могут быть вложенными.

#### aggregate
Медиатор aggregate имее два блока
- **completeCondition** - здесь описываются условия завершения агрегации
- **onComplete** - здесь описывается действие, которое производится, когда все результаты получены

```xml
<aggregate>
    <completeCondition>
        <messageCount max="-1" min="1"/>
    </completeCondition>
    <onComplete expression="//odh">
        <log level="full"/>        
    </onComplete>
</aggregate>
```

В данном примере выставлено минимальное количество полученных элементов 1 а максимальное не ограниченно. 

По завершении из каждого полученного элемента извлекается элемент odh. Результирующи массив будет состоять именно из
элементов odh.

### БД
Для работы с БД используются медиаторы dbreport и dblookup
#### Подключение
Для подключения к БД в медиаторах dbreport и dblookup добавляется секция connection
```xml
<dbreport>
    <connection>
        <pool>
            <driver>org.postgresql.Driver</driver>
            <url>jdbc:postgresql://192.168.2.121:5432/wso2</url>
            <user>db_user</user>
            <password>db_password</password>
        </pool>
    </connection>
</dbreport>
```
#### Вставка, изменение данных
Для модификации данных в БД используется медиатор dbreport
```xml
<dbreport>
    <connection>
        <pool>
            <driver>org.postgresql.Driver</driver>
            <url>jdbc:postgresql://192.168.2.121:5432/wso2</url>
            <user>db_user</user>
            <password>db_password</password>
        </pool>
    </connection>
    <statement>
        <sql><![CDATA[insert into wso2.request_endpoint(request_id, endpoint) values(?, ?)]]></sql>
        <parameter expression="get-property('requestId')" type="VARCHAR"/>
        <parameter expression="get-property('endpoint')" type="VARCHAR"/>
    </statement>
</dbreport>
```

При этом параметры попадают в запрос в том порядке в которо они объявлены.
#### Получение данных
Для получения данных из БД используется медиатор dblookup
```xml
<dblookup>
    <connection>
        <pool>
            <driver>org.postgresql.Driver</driver>
            <url>jdbc:postgresql://192.168.2.121:5432/wso2</url>
            <user>db_user</user>
            <password>db_password</password>
        </pool>
    </connection>
    <statement>
        <sql><![CDATA[select * from request_endpoint where request_id =?]]></sql>
        <parameter expression="get-property('requestId')" type="VARCHAR"/>
        <result column="endpoint" name="uri.var.endpoint"/>
    </statement>
</dblookup>
```

Данные из колонки endpoint будут записаны в переменную с именем **uri.var.endpoint**

### Очереди сообщений
Кроме основного применения очереди сообщений выполняют также и другуют функцию - с помощью очереди можно распараллеливать 
выполнение задачи. Например если необходимо вернуть результат из API и в то же время продолжить обработку запроса.
Например если необходимо вернуть идентификатор запроса из API и продолжить выполнение запроса, а результат вернуть на другой
endpoint. В таком случае необходимо использовать очередь.

Для того чтобы организовать отправку и обработку сообщений необходимо создать:
- message sore
- message processor
- sequence обработчик

**Message Store** получает и хранит сообщения. Он может использовать для этого как оперативную память, так и существующие 
очереди сообщений такие как RabbitMQ.

Для создания message store необходимо создать файл src/main/synapse-config/message-stores/{store name}.xml
В котором описать соответствующий message store
```xml
<?xml version="1.0" encoding="UTF-8"?>
<messageStore class="org.apache.synapse.message.store.impl.memory.InMemoryStore" name="odsMessageStore" xmlns="http://ws.apache.org/ns/synapse"/>
```
В данном примере создаётся простейший message store который хранит сообщения в памяти. Его хорошо применить для
распараллеливания выполнения, так как сообщения будут извлекаться тутже и риски потери данных при падении wso2 минимальные.

Также в файле artifact.xml необходимо добавить 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<artifacts>
    <artifact name="{store name}" groupId="ru.my.wso2.integration.message-store" version="1.0.0" type="synapse/message-store" serverRole="EnterpriseServiceBus">
        <file>src/main/synapse-config/message-stores/{store name}.xml</file>
    </artifact>
</artifacts>
``` 

Для отправки сообщения в message store используется медиатор **store**
```xml
<store messageStore="odsMessageStore"/>
```
При этом в message store передаются текущие данные полученные с последнего медиатора.

**Message Processor** слушает message store и при появлении новых сообщений вызывает указанный sequence для обработки полученного
сообщения.  

Для создания message processor с именем odsMessageProcessor необходимо создать файл src/main/synapse-config/message-processors/odsMessageProcessor.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<messageProcessor class="org.apache.synapse.message.processor.impl.sampler.SamplingProcessor" messageStore="odsMessageStore" name="odsMessageProcessor" xmlns="http://ws.apache.org/ns/synapse">
    <parameter name="sequence">COLLECT_ODS_DATA_SEQ</parameter>
    <parameter name="interval">1000</parameter>
    <parameter name="is.active">true</parameter>
    <parameter name="concurrency">1</parameter>
</messageProcessor>
```
В данном примере message processor будет опрашивать odsMessageStore раз в секунду и при появлении новых сообщений
передавать их в COLLECT_ODS_DATA_SEQ.

Также в файле artifact.xml необходимо добавить 
```xml
<?xml version="1.0" encoding="UTF-8"?><artifacts>
    <artifact name="odsMessageProcessor" groupId="ru.my.wso2.integration.message-processors" version="1.0.0" type="synapse/message-processors" serverRole="EnterpriseServiceBus">
        <file>src/main/synapse-config/message-processors/odsMessageProcessor.xml</file>
    </artifact>
</artifacts>
``` 

Чтобы создать COLLECT_ODS_DATA_SEQ из примера выше необходимо создать файл src/main/synapse-config/sequences/COLLECT_ODS_DATA_SEQ.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<sequence name="COLLECT_ODS_DATA_SEQ" trace="disable" xmlns="http://ws.apache.org/ns/synapse">
</sequence>
```
Также в файле artifact.xml необходимо добавить 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<artifacts>
    <artifact name="COLLECT_ODS_DATA_SEQ" groupId="ru.my.wso2.integration.sequence" version="1.0.0" type="synapse/sequence" serverRole="EnterpriseServiceBus">
        <file>src/main/synapse-config/sequences/COLLECT_ODS_DATA_SEQ.xml</file>
    </artifact>
</artifacts>
``` 

### Взаимодействие с источниками данных
В данном примере будут расмотренны варианты взаимодействия, где приложение запрашивает wso2 в асинхронном режиме.
Тоесть приложение отправляет wso2 запрос в котором содержится endpoint для получения ответов. В ответ будет возвращаться 
идентификатор запроса. 

В последствии когда ответ будет готов данные вместе с идентификатором запроса будут возвращены на переданный ранее endpoint.

#### Выполнение запроса к источнику данных
##### Отправка запроса к сервису описанному через WSDL
Для отправки запроса к сервису необходимо
- [создать endpoint типа wsdl](#endpoint) 
- [создать тело запроса](#Тело-запроса)
- [установить заголовки запроса](#Заголовки-запроса)
- [сделать запрос](#Вызов)

Тело запроса выглядит следующим образом
```xml
<soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:web="http://webservices.ods.fors.ru/">
    <soapenv:Header/>
    <soapenv:Body>
        <web:getOdhPassport>
            <odhId xmlns="">$1</odhId>
        </web:getOdhPassport>
    </soapenv:Body>
</soapenv:Envelope>
```
Где web:getOdhPassport - название вызываемого метода, а odhId параметр метода

Для вызова метода необходимо установить следующий заголовок
```xml
<header name="Content-Type" scope="transport" value="text/xml; charset=UTF-8"/>
```

Таким образом законченный пример вызова WSDL сервиса будет выглядеть следующим образом
```xml
<payloadFactory media-type="xml">
    <format>
        <soapenv:Envelope xmlns:soapenv="http://schemas.xmlsoap.org/soap/envelope/" xmlns:web="http://webservices.ods.fors.ru/">
            <soapenv:Header/>
            <soapenv:Body>
                <web:getOdhPassport>
                    <odhId xmlns="">$1</odhId>
                </web:getOdhPassport>
            </soapenv:Body>
        </soapenv:Envelope>
    </format>
    <args>
        <arg evaluator="xml" expression="//odh/id/text()"/>
    </args>
</payloadFactory>
<header name="Content-Type" scope="transport" value="text/xml; charset=UTF-8"/>
<call>
    <endpoint key="ODS_WSDL_EP"/>
</call>
```

#### Взаимодействие с асинхронным источником данных
Для взаимодействия с асинхронными источниками необходимо завести в API WSO2 следующие endpointы:
- **endpoint запроса** - на этот адрес будут приходить запросы от приложения
- **endpoint ответа** - на этот адрес будут приходить ответы от асинхронного источника данных

**Endpoint запроса** выполняет следуючщие действия:
- [генерирует тело запроса](#Тело-запроса) к источнику данных (если необходимо) 
- [делает запрос](#Вызов) к источнику данных
- получает идентификатор запроса и ответа от источника данных
- [сохраняет в бд](#Вставка,-изменение-данных) соответствие идентификатора запроса переданному endpoint
- генерирует ответ приложению содержащий идентификатор запроса
- возвращает ответ приложению 

**Endpoint ответов** выполняет следующие действия:    
- [получает endpoint для отправки ответа](#Получение-данных)
- [формирует тело ответа](#Тело-запроса)
- [отправляет ответ](#Вызов) на [динамический endpoint](#endpoint) 

[Пример](examples/asyncExample)

#### Взаимодействие с синхронным источником данных
Для взаимодействия с синхронным источником данных по [описанной выше содели](#Взаимодействие-с-источниками-данных). 
Необходимо:
- [создать endpoint в API](#API)
- [создать message store в памяти](#Очереди-сообщений)
- [создать message processor](#Очереди-сообщений)
- [добавить sequence обработчик](#Очереди-сообщений) запросов который
-- получает сообщение и извлекает из него endpoint и requestId
-- выполняет запрос к источнику данных
-- формирует тело ответа с полученными данными и requestId
-- отправляет сформированный запрос на динамический endpoint
- получить текущий MessageId (данный параметр содержит uuid текущего запроса)
- добавить MessageId к телу запроса как requestId
- [отпавить получившееся сообщение в очередь сообщений](#Очереди-сообщений)
- сформировать ответ содержащий MessageId как requestId
- вернуть получившийся ответ

[Пример](examples/odsSyncAsync)