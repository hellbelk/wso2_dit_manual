## WSO2 инструкция
### Установка
### Отправка запроса к сервису описанному через WSDL
### Маппинг SOAP ответа в JSON
Вернуть ответ в виде json можно 3 способами:
- создать payload медиатор и задать у него параметр json
- xslt преобразование
- data mapping медиатор
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
### БД
#### Подключение
#### Вставка, изменение данных
#### Получение данных
### Взаимодействие с асинхронным источником данных
```puml
A -> B
```
### Взаимодействие с синхронным источником данных
### Запрос дополнительных данных
### Динамический endpoint
### Переменные