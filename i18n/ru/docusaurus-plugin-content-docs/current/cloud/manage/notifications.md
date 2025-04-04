---
title: Уведомления
slug: /cloud/notifications
description: Уведомления для вашего сервиса ClickHouse Cloud
keywords: [cloud, notifications]
---

import notifications_1 from '@site/static/images/cloud/manage/notifications-1.png';
import notifications_2 from '@site/static/images/cloud/manage/notifications-2.png';
import notifications_3 from '@site/static/images/cloud/manage/notifications-3.png';
import notifications_4 from '@site/static/images/cloud/manage/notifications-4.png';

ClickHouse Cloud отправляет уведомления о критических событиях, связанных с вашим сервисом или организацией. Есть несколько концепций, которые стоит учесть, чтобы понять, как отправляются и настраиваются уведомления:

1. **Категория уведомления**: Относится к группам уведомлений, таким как уведомления о выставлении счетов, уведомления, связанные с сервисом и т.д. В каждой категории есть несколько уведомлений, для которых можно настроить способ доставки.
2. **Серьезность уведомления**: Серьезность уведомления может быть `info`, `warning` или `critical` в зависимости от важности уведомления. Это не подлежит настройке.
3. **Канал уведомления**: Канал относится к способу получения уведомления, такому как интерфейс пользователя, электронная почта, Slack и т.д. Это настраивается для большинства уведомлений.

## Получение Уведомлений {#receiving-notifications}

Уведомления можно получать через различные каналы. В настоящее время ClickHouse Cloud поддерживает получение уведомлений по электронной почте и через интерфейс ClickHouse Cloud. Вы можете нажать на иконку колокольчика в верхнем левом меню, чтобы просмотреть текущие уведомления, что откроет выпадающее меню. Нажатие кнопки **Просмотреть все** внизу выпадающего меню приведет вас на страницу, которая показывает журнал активности всех уведомлений.

<br />

<img src={notifications_1}
    alt="Выпадающее меню уведомлений ClickHouse Cloud"
    class="image"
    style={{width: '600px'}}
 />

<br />

<img src={notifications_2}
    alt="Журнал активности уведомлений ClickHouse Cloud"
    class="image"
    style={{width: '600px'}}
 />

## Настройка Уведомлений {#customizing-notifications}

Для каждого уведомления вы можете настроить, как вы будете его получать. Вы можете получить доступ к экрану настроек из выпадающего меню уведомлений или из второй вкладки в журнале активности уведомлений.

Чтобы настроить доставку для конкретного уведомления, нажмите на значок карандаша, чтобы изменить каналы доставки уведомления.

<br />

<img src={notifications_3}
    alt="Экран настроек уведомлений ClickHouse Cloud"
    class="image"
    style={{width: '600px'}}
 />

<br />

<img src={notifications_4}
    alt="Настройки доставки уведомлений ClickHouse Cloud"
    class="image"
    style={{width: '600px'}}
 />

<br />

:::note
Некоторые **обязательные** уведомления, такие как **Неудача оплаты**, не подлежат настройке.
:::

## Поддерживаемые Уведомления {#supported-notifications}

В настоящее время мы отправляем уведомления, связанные с выставлением счетов (неудача платежа, превышение установленного порога использования и т.д.), а также уведомления, связанные с событиями масштабирования (масштабирование завершено, масштабирование заблокировано и т.д.). В будущем мы добавим уведомления для резервных копий, ClickPipes и других соответствующих категорий.
