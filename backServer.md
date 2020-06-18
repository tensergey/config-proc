# Описание backServer

Сервис занимается обработкой платежа, построенное по технологии SOA c 
использованием фраемворка [Spring Integration](https://docs.spring.io/spring-integration/reference/html/),
для понимания как оно все работает и по каким принципам необходимо прочитать книгу [Шаблоны интеграции корпоративных приложений](https://g.co/kgs/TfhcVH)

## Компоненты системы 
Cервис разделяется на следующие основные компоненты:
- gateFirst
- gateTransport /integrationFlowPayFirstToThirdPosting
- gateThirdPosting
- gateTransport / integrationFlowPayThirdPostingToScript
- scriptSimple
- gateTransport / integrationFlowPayScriptToPosting
- gateLast
- stepError
- watchDog

### gateFirst
Компонент забирает сообщения из кафки и запускает первичную обработку, и начинает обрабатывать сообщение по шагам, 
настройка представленна ниже
```java
        return IntegrationFlows
                .from(processorTrn.inputPayFrontTranId())
                .channel(mc -> mc.queue(50)) //устанавливаем количество сообщений в локальной очереди
                .bridge(s -> s.poller(getPollerBuilder().makePoller(fixedDelay, 15, "FirstTrnPay-"))) 
                .channel("integrationFlowFirstTrnPayChannelStatistic")
                .log(LoggingHandler.Level.TRACE)
                .handle(transformAqTrnToTrn)
                .channel(MessageChannels.direct("messageChannelPayFirstAfterSource"))
                .handle(sa2SelectFinAgentPay())
                .handle(sa3SelectVendorForTrn())
                .handle(sa3TrnServiceCopyExtFieldPay())
                .handle(sa4TrnServiceCopyAuthField)
                .handle(sa4FindDabbleTerminal())
                .handle(sa4TrnServiceCreateTrmToAgent)
                .handle(sa5TrnSetNumberTrn)
                .log(LoggingHandler.Level.DEBUG)
                .handle(sa6TrnServiceSendAqPay())
                .get();
```
разберем по порядку
1) `.from(processorTrn.inputPayFrontTranId())` забирает сообщение из кафки
2) `.channel(mc -> mc.queue(50)) ` устанавливаем количество сообщений в локальной очереди
3) `.bridge(s -> s.poller(getPollerBuilder().makePoller(fixedDelay, 15, "FirstTrnPay-"))) ` настройка выборки из кафки
4) `.channel("integrationFlowFirstTrnPayChannelStatistic")`  создаем канал поо которому можем получать статистику
5) `.handle(transformAqTrnToTrn)` загружаем транзакцию описывающую платеж
6) `.channel(MessageChannels.direct("messageChannelPayFirstAfterSource"))` канал для тестирования
7) `.handle(sa2SelectFinAgentPay())` определятся финансовый агент, на балансе которого прошел платеж 
8) `.handle(sa3SelectVendorForTrn())` выбирается вендора, который будет обрабатывать платеж, здесь строится граф,
 и сохраняется путь прохождения платежа
9) `.handle(sa3TrnServiceCopyExtFieldPay())` копируются поля от провайдера, к вендору
10) `.handle(sa4TrnServiceCopyAuthField)`  копируем поля с авторизации, к вендору
11) `.handle(sa4FindDabbleTerminal())` проводки по замещающим точка приема
12) `.handle(sa4TrnServiceCreateTrmToAgent)`  формируем проводки с точки приема 
13) `.handle(sa5TrnSetNumberTrn)` устанавливаем номер платежа, для вендоров
14) `.handle(sa6TrnServiceSendAqPay())` отправляем платеж на формирование проводок


### gateThirdPosting
Формирует проводки, для найденого пути 
```java
        return IntegrationFlows
                 .from(thirdTrnPostingMessageChannel())
                 .bridge(pl -> pl.poller(getPollerBuilder()
                         .makePoller(fixedDelay, 15, "ThirdTrnPosting-"))
                         .id("third_posting_poller"))
                 .channel("integrationFlowThirdTrnPostingChannelStatistic")
                 .<AQTrnType, TrnAbstractEntity>transform(transformAqTrnToTrn::load)
                 .channel(MessageChannels.direct("messageChannelThirdPostingAfterFirst"))
                 .handle(sa2TrnServiceCreatePostingNew)
                 .transform(sa3TrnServiceSendAqPosting::toAqTrnType)
                 .channel(thirdTrnPostingToScriptMessageChannel())
                 .get();
```
1) `.handle(sa2TrnServiceCreatePostingNew)` формирует проводки 
3) `.channel(thirdTrnPostingToScriptMessageChannel())` отправляет на выполнение скрипта                   

### scriptSimple
Запускает скрипт, отправляет информацию о платеже вендору
```java
        return IntegrationFlows
                .from(scriptGateScriptMessageChannel())
                .bridge(pl -> pl.poller(getPollerBuilder()
                        .makePoller(fixedDelay, 20, "script-"))
                        .id("script_poller"))
                .channel("integrationFlowScriptTrnChannelStatistic")
                .<AQTrnType, TrnTypeOper>route(AQTrnType::getOperKds, r -> r
                        .subFlowMapping(trnAuth, makeSubFlowScriptExecutedAuth(sa4SendAQAuth, transformAqTrnToTrnAuth))
                        .subFlowMapping(trnPay, makeSubFlowScriptExecutedPay(sa4SendAQPay, transformAqTrnToTrn))
                        .resolutionRequired(true))
                .get();

    private IntegrationFlow makeSubFlowScriptExecutedPay(
            SA4SendAQPay sa4SendAQPay,
            TransformAqTrnToEntity<TrnEntity> transformAqTrnToTrn) {
        return sf -> sf
                .transform(transformAqTrnToTrn::load)
                .channel(MessageChannels.direct("messageChannelPayScriptAfterSource"))
                .handle(sa2SendRequestServerPay())
                .handle(sa3CopyFieldVendor2Provider)
                .handle(sa4SendAQPay);
    }
```
1) `.handle(sa2SendRequestServerPay())` запускает скрипт
2) `.handle(sa3CopyFieldVendor2Provider)` копирует поля от вендора к провайдеру
3) `.handle(sa4SendAQPay)` отправляет платеж в компонент

### gateLast
Завершает обработку транзакции, устанавливает конечный статус платежа 
```java
return IntegrationFlows
                .from(lastPostingMessageChannel())
                .bridge(pl -> pl.poller(getPollerBuilder()
                        .makePoller(fixedDelay, 10, "LastPosting-"))
                        .id("posting_poller"))
                .channel("lastPostingIntegrationFlowChannelStatistic")
                .transform(transformAqTrnToTrn::load)
                .channel(MessageChannels.direct("messageChannelPayLastAfterSource"))
                .handle(sa2SendPosting)
                .handle(sa4SaveTrn)
                .handle((payload) -> MDC.remove("trnId"))
                .get();
```              
`.handle(sa2SendPosting)` запускает финальные проводки

### stepError
Сервис является перехватчиком ошибок, если ошибка критичная устанавливает платежу фатальную обработку, 
если ошибка не финальна устанавливает задержку     

### watchDog
Сервис запрашивает из бд список платежей у которых просрочилась дата обработки. Возможно при падаение,
 или при ошибке обработки, или если установленна задержка для проверки статуса.
 
запрос:
```sql
WITH cte AS (
    SELECT trn_id
    FROM pr_queue_watcher
    WHERE expiration_dt <= ?
    LIMIT 1
    FOR UPDATE SKIP LOCKED)
UPDATE pr_queue_watcher q
SET
  expiration_dt = ?,
  attempt       = attempt + 1
FROM cte
WHERE q.trn_id = cte.trn_id 
RETURNING q.trn_id, q.trn_version, q.trn_type, q.attempt, q.start_dt, q.expiration_dt
```    
передается текущая дата

Выбирается транзакция, определяется компонент в зависимости от пройденных шагов, и отправляется в компонент 
 
## Описание транзакции в бд