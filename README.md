Что же такое API Gateway ?

Из названия вытекает подсказка, что это некий Gateway, который предоставляет какое-то API, являясь единой точкой входа для всех сервисов. Так и есть.
API Gateway — это единая точка входа для всех клиентов. Некоторые запросы просто проксируются или перенаправляются в соответствующую службу.
В микросервисной архитектуре число компонентов растет довольно быстро.
Клиентское приложение могло бы запрашивать каждый из сервисов самостоятельно. Но такой подход сразу натыкается на массу ограничений: необходимость знать адрес каждого эндпоинта, делать запрос по каждому фрагменту информации отдельно и самостоятельно обрабатывать результат.

Для решения таких проблем применяют некий Gateway — единую точку входа. Ее используют для приема внешних запросов и маршрутизации в нужные сервисы внутренней инфраструктуры.
![image](https://github.com/Nekipel/API-gateway/assets/88710417/350111c1-21ac-449e-b1cf-54322a33d3c0)
Шаблон Gateway является хорошей отправной точкой для архитектуры микросервисов, поскольку он позволяет направлять определенные запросы к различным службам.

В дальнейшем мы будем рассматривать конкретную реализацию такого шаблона, и поможет нам в этом  Spring Cloud Gateway.


Spring Cloud Gateway

Для разработчика Spring Cloud Gateway представляется как Spring-Boot приложение, которое позволяет настраивать маршрутизацию в yaml-конфиге или в коде приложения, а также расширять логику путём написания своего кода. Разберемся в этом подробнее.

Прежде всего, Spring Cloud Gateway построен на:
Spring Boot 2.x
Spring Webflux
Project Reactor

Из этого вытекают его основные преимущества:
Тесно интегрирован в Spring.
Неблокирующий.
В сравнении, например, с тем же Zuul, поддерживает веб-сокеты.

Поскольку это обычное Spring Boot приложение, у разработчиков появляется возможность легко писать необходимую логику рядом с маршрутизацией, например фильтрация запросов на основе различных критериев, допустим, по  наличию или отсутствию заголовков.

Стоит сказать пару слов о Zuul, который упоминается выше.
Еще в 2018 году проект перешел на режим поддержания, а позже был полностью удален и заменен на Spring Cloud Gateway. По этой причине мы не будем рассматривать его в курсе, но ты можешь самостоятельно посмотреть, что онсобой представляет.

Как устроен Gateway
![image](https://github.com/Nekipel/API-gateway/assets/88710417/136cba5d-ebcb-44af-bbc1-ce8636085a3c)

Когда запрос достигает шлюза, он сопоставляет запрос с каждым доступным маршрутом на основе определенного предиката. Если маршрут соответствует, запрос переходит к веб-обработчику, который применяет фильтры к запросу.

В Spring Cloud Gateway есть такое понятие, как маршрут (route). Маршрут — это основной строительный блок шлюза. Он определяется идентификатором, URI назначением, набором предикатов и набором фильтров.



Маршрутизация на основе класса конфига

Ниже приведена простая реализация маршрута, которая направляет все запросы, соответствующие регулярному выражению /api/book-service/, в book-service. 

Для этого используется бин RouteLocator, в котором и происходит вся магия.

@Bean
 RouteLocator gatewayRoutes(RouteLocatorBuilder builder) {
  return builder.routes()
.route("book-service", r -> r.path("/api/book-service/**")
                      //.filters(f -> f.filter(filter))
                         .filters(f ->                  f.rewritePath("/api/book-service/(?.*)","/${remains}")
  .addRequestHeader("X-book-Header", "book-service-header")
.uri("lb://book-service/"))

код весь подсвечивается красным. работающая версия кода
@Configuration public class GatewayConfig {      
@Bean 
public RouteLocator customRouteLocator(RouteLocatorBuilder builder) {         
  return builder.routes().route("book-service", r -> r.path("/api/book-service/**").uri("lb://book-service")).build();}}
  
также что бы это заработало удалите из pom spring-boot-starter-web, добавьте spring-cloud-starter-gateway
и в конфиг файле yaml укажите spring:
  main:
    web-application-type: reactive
 

Здесь мы использовали встроенные фильтры для перезаписи URL-адреса и добавления настраиваемого заголовка в качестве X-book-Header на лету.
Закомментированная строка фильтра .filters(f -> f.filter(filter))
показывает, что возможно описать логику фильтра в другом классе и применить ее к набору routes. 
Пока все элементарно. Идем дальше.

Данный код создает маршрут в Spring Cloud Gateway с помощью RouteLocatorBuilder. Маршрут срабатывает при запросах на путь, начинающийся с "/api/book-service/". При срабатывании маршрута выполняются следующие действия:

Фильтр rewritePath изменяет путь запроса, удаляя сегмент "/api/book-service/". Например, запрос "/api/book-service/books/123" будет изменен на "/books/123". Это делается с помощью регулярного выражения "/api/book-service/(?.*)" и шаблона замены "/${remains}".
Фильтр addRequestHeader добавляет заголовок "X-book-Header" со значением "book-service-header" к запросу.
Маршрут перенаправляется на сервис "book-service", используя балансировщик нагрузки, указанный в URI "lb://book-service/".



Маршрутизация на основе файла property

Есть другой способ сделать маршрутизацию — на основе файла свойств.

spring:
 cloud:
   gateway:
     routes:
       - id: client-service
     uri: lb://client-service
     predicates:
       - Path=/api/client/**
     filters:
       - RewritePath=/api/client/(?.*), /$\{remains}
       - AddRequestHeader=X-client-Header, client-service-header
       - name: Hystrix
       args:
         name: hystrix
         fallbackUri: forward:/fallback/client
Мне кажется, здесь все очевидно, как и в примере из прошлой лекции.

Мы указываем путь (route), сопоставляем его с URL и навешиваем на него predicate. Также у нас есть фильтр, который к запросу добавляет некоторый хедер, например, являющийся обязательным в client-service, на который мы пытаемся попасть.

В разделе predicates мы указали путь, по которому движемся (client-service).

Для тех, кому не все очевидно:

Данные настройки конфигурируют маршрут в Spring Cloud Gateway с идентификатором "client-service". Маршрут срабатывает при запросах на путь, начинающийся с "/api/client/". При срабатывании маршрута выполняются следующие действия:

1. Фильтр RewritePath изменяет путь запроса, удаляя сегмент "/api/client/". Например, запрос "/api/client/orders/123" будет изменен на "/orders/123". Это делается с помощью регулярного выражения "/api/client/(?.*)" и шаблона замены "/${remains}".

2.Фильтр AddRequestHeader добавляет заголовок "X-client-Header" со значением "client-service-header" к запросу.

3. Фильтр Hystrix добавляет обработку отказов с помощью Hystrix. Hystrix позволяет добавлять обработку отказов на уровне маршрута, например, для обработки ошибок взаимодействия с удаленными сервисами. В этом случае, если удаленный сервис не отвечает или возвращает ошибку, запрос будет перенаправлен на URI, указанный в параметре fallbackUri. В данном случае, запрос будет перенаправлен на путь "/fallback/client".


Дополнительные возможности Spring Cloud Gateway

Spring Cloud Gateway — довольно мощный инструмент для разработчика. Он позволяет тонко настроить API Gateway под любые задачи.

Рассмотрим несколько примеров.

Существуют различные предикаты, например:

spring:
 cloud:
   gateway:
     routes:
       - id: client-service
     uri: lb://client-service
     predicates:
       - Path=/api/client/**
       - Before=2017-01-20T17:42:47.789-07:00[America/Denver]

spring:
 cloud:
   gateway:
     routes:
       - id: client-service
     uri: lb://client-service
     predicates:
       - Path=/api/client/**
       - After=2017-01-20T17:42:47.789-07:00[America/Denver]

spring:
 cloud:
   gateway:
     routes:
       - id: client-service
     uri: lb://client-service
     predicates:
       - Path=/api/client/**
       - Between=2017-01-20T17:42:47.789-07:00[America/Denver], 2017-01-21T17:42:47.789-07:00[America/Denver]
Before принимает один параметр — datetime (ZonedDateTime). Этот предикат соответствует запросам, которые происходят до указанного datetime.

After также принимает один параметр — datetime (ZonedDateTime). Этот предикат соответствует запросам, которые происходят после указанного datetime.

Between принимает два параметра — datetime1 и datetime2 (Java ZonedDateTime). Этот предикат соответствует запросам, которые происходят после datetime1 и до datetime2. Параметр datetime2 должен быть после datetime1.

Вывод: этот маршрут соответствует любому запросу, сделанному после 20 января 2017 г., 17:42 и до 21 января 2017 г., 17:42.

Существует и такой вид предиката.


spring:
 cloud:
   gateway:
     routes:
       - id: client-service
     uri: lb://client-service
     predicates:
       - Method=GET,POST

 

Method принимает аргумент метода, который представляет собой один или несколько параметров: HTTP методы для сопоставления.


spring:
 cloud:
   gateway:
     routes:
       - id: client-service
     uri: lb://client-service
     predicates:
       - RemoteSpring Cloud StreamAddr=192.168.0.1/16

RemoteAddr принимает список источников (минимум один), которые представляют собой алреса IPv4 или IPv6, например, 192.168.0.1/16, где 192.168.0.1 — это IP-адрес, а 16 — маска подсети.


filters:
 - name: CircuitBreaker
     args:
       name: myCircuitBreaker
       fallbackUri: forward:/inCaseOfFailureUseThis
 - RewritePath=/consumingServiceEndpoint, /backingServiceEndpoint

 

Добавление обработчика Circuit Breaker

Это означает, что в случае неудачи будет вызван Circuit Breaker и выполнена какая-то логика либо повторный запрос на текущий адрес будет недоступен, пока не будет устранена неполадка. С Circuit Breaker подробнее ознакомимся позже.

filters:
 - name: Retry
     args:
       retries: 3
       statuses: BAD_GATEWAY
       methods: GET,POST
       backoff:
         firstBackoff: 10ms
         maxBackoff: 50ms
         factor: 2
         basedOnPreviousValue: false
Retry устанавливает количество повторных попыток в случае неуспешного выполнения запроса.

Как настроить таймаут соединения? Очень просто для всех запросов.

spring:
 cloud:
   gateway:
     httpclient:
       connect-timeout: 1000
       response-timeout: 5s
 

Для конкретного запроса:


spring:
 cloud:
   gateway:
     routes:
       - id: per_route_timeouts
       uri: https://example.org
       predicates:
         - name: Path
           args:
             pattern: /delay/{timeout}
       metadata:
           response-timeout: 200
           connect-timeout: 200

Или через java код, как мы делали это выше.

@Bean
public RouteLocator customRouteLocator(RouteLocatorBuilder routeBuilder){
  return routeBuilder.routes()
     .route("test", r -> {
        return r.host("*.mysite.com").and().path("/somepath")
           .uri("http://someuri")
           .metadata(RESPONSE_TIMEOUT_ATTR, 200)
           .metadata(CONNECT_TIMEOUT_ATTR, 200);
     })
     .build();

Spring Cloud Gateway предоставляет несколько предикатов, которые можно использовать для определения условий маршрутизации запросов. Некоторые из наиболее распространенных предикатов:

Path - определяет, соответствует ли путь запроса шаблону. Например:
predicates:
  - Path=/api/**

Method - определяет, соответствует ли метод запроса заданному значению. Например:
predicates:
  - Method=GET

Header - определяет, содержит ли заголовок запроса заданное значение. Например:
predicates:
  - Header=X-Api-Key, my-api-key

Cookie - определяет, содержит ли куки запроса заданное значение. Например:
predicates:
  - Cookie=SESSION, ABC123

RemoteAddr - определяет, соответствует ли IP-адрес клиента заданному значению. Например:
predicates:
  - RemoteAddr=192.168.0.1

Weight - используется для балансировки нагрузки между несколькими экземплярами сервиса. Например:
predicates:
  - Weight=service1, 0.7
  - Weight=service2, 0.3Здесь указывается вес для каждого экземпляра сервиса. При обработке запроса выбирается сервис с наибольшим весом.

CloudFoundryRouteService - используется для маршрутизации запросов через Cloud Foundry Route Service. Например:
predicates:
  - CloudFoundryRouteService=service1

Custom - позволяет определить собственный предикат. Например:
predicates:
  - name: MyPredicate
    args:
      param1: value1
      param2: value2

Здесь MyPredicate - это имя класса, реализующего интерфейс java.util.function.Predicate. Параметры для предиката указываются в виде аргументов.

Также можно использовать несколько предикатов вместе, для создания более сложных условий маршрутизации
