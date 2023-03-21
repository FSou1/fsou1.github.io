---
title: Azure Monitor- возможности и ограничения
tags: Azure Cloud
redirect_from: "/Azure_monitor_capabilities_and_constrains/"
---

Сегодня хотелось бы с вами поделиться заметкой, которая появилась в результате моего небольшого research на тему ключевых особенностей Azure Monitor.

## Возможности

Опираясь на [Microsoft docs](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview-azure-monitor) описать ключевые возможности инструмента можно следующим образом:

- route - streaming данных между сервисами и функционал оповещения;
- store and arhive - хранилище данных;
- query - предоставление доступа к данным на чтение;
- visualize - визуализация, представление и аналитика;
- automate - автоматизация и триггеры процессов (autoscale, email, webhook);

Представленное ниже изображение характерно для вычисляемых (исполняемых) ресурсов и отличается от невычисляемых лишь наличием Application Logs and metrics.

![monitor capabilities](/assets/images/posts/monitoring_azure_resources-compute_v6.png)

### Activity log

Данный раздел мониторинга содержит информацию об операциях, выполненных в рамках конкретных компонентов (ресурсов):

- Наименование (тип) операции;
- Кто её инициировал и когда она была завершена;
- Статусе её выполнения;

Кроме того, предоставляются возможности:

- Экспорта в Storage Account & Azure Event Hub;
- Доступа посредством PowerShell & REST API;
- Добавления правил оповещения;
- Анализа в PowerBI;

![metrics and events](/assets/images/posts/activity_log_overview_v3.png)

### Metrics and Events

Кроме возможности отслеживать информацию о метриках и событиях используемых компонентов Azure также доступно добавление собственных custom metrics & events.

Если мы хотим предпринять действие основанное на значении метрики, то в этом нам поможет система оповещения и реагирования, в арсенале которой присутствует возможность отправлять email уведомления, вызывать webhook или запускать logic app (с помощью request trigger).

![metrics and events](/assets/images/posts/metrics_overview_v4.png)

### Log search

Благодаря log search и специализированному языку запросов доступна возможность выполнять query/filter/aggregate применимо к логам и метрикам, а также визуализировать итоговые результаты в требуемом виде (table, list, bar).

![log search](/assets/images/posts/oms-search-select.png)

### Autoscale

Горизонтальное масштабирование основанно на наступлении одного из двух видов условий:

- На основании значения метрики- например, увеличить количество запущенных экземпляров того или иного ресурса в случае превышения использования % CPU или загрузки ОЗУ;
- На основании расписания- например, если нагрузка в выходные дни снижается, то можно уменьшать и количество экземпляров до необходимого минимума;

![autoscale overview](/assets/images/posts/autoscale_overview_v4.png)

### Azure Service Health

Благодаря такому компоненту как Azure Service Health появляется возможность своевременно узнавать о технических работах и сбоях в инфраструктуре Azure, которые могут затронуть доступность развернутых в облаке сервисов и ресурсов. В качестве оповещений могут выступать уведомления по почте, телефону или webhook.

Также появляется возможность заблаговременно подготовиться к плановому обслуживанию Azure.

![azure service health](/assets/images/posts/azure-service-health-overview-7.png)

## Ограничения

Задержка при отправке уведомлений о наступлении событий- ограничением это можно назвать весьма условно, однако, оповещение о срабатывании того или иного правила будет получено в интервале от одной до пяти минут.

Отсутствует вертикальное масштабирование- на данный момент возможно изменение количества запущенных экземпляров (как в большую, так и в меньшую сторону), но не их вычислительных мощностей. Связано это с накладываемыми на такой вид масштабирования ограничениями- возможные отсутствие аппаратных средств и необходимость перезапуска.

Не все Azure services на данный момент помещают информацию в Azure Monitor, однако, это будет реализовано в будущем.

Срок хранения metrics и activity log entry ограничен- он составляет 30 и 90 дней соответственно (для большей продолжительности необходимо подключить Azure storage).

## Заключение

В заключении можно сказать, что рассмотренный в данной статье инструмент позволяет:

- Однозначно ответить кто, когда и какие действия выполнял применимо к ресурсам или инфраструктуре;
- Получить, проанализировать и визуализировать информацию о выполненных действиях;
- Добавлять набор правил и действий для реагирования при их наступлении;

Полезные ссылки:

1. [Overview of Azure Monitor](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview-azure-monitor);
2. [Overview of the Azure Activity Log](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview-activity-logs);
3. [Overview of metrics in Microsoft Azure](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview-metrics);
4. [Overview of autoscale in Microsoft Azure](https://docs.microsoft.com/en-us/azure/monitoring-and-diagnostics/monitoring-overview-autoscale);
5. [Azure Service Health overview](https://docs.microsoft.com/en-us/azure/service-health/service-health-overview);
6. [Supported Resource Types through Azure Resource Health](https://docs.microsoft.com/en-us/azure/service-health/resource-health-checks-resource-types);
7. [Azure Monitor videos](https://azure.microsoft.com/ru-ru/resources/videos/index/?services=monitor);
