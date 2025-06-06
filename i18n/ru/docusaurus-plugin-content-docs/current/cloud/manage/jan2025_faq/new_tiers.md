---
title: 'Описание новых уровней'
slug: /cloud/manage/jan-2025-faq/new-tiers
keywords: ['новые уровни', 'функции', 'ценообразование', 'описание']
description: 'Описание новых уровней и функций'
---

## Резюме ключевых изменений {#summary-of-key-changes}

### Какие ключевые изменения ожидать в отношении соответствия функций уровням? {#what-key-changes-to-expect-with-regard-to-features-to-tier-mapping}

- **Private Link/Private Service Connect:** Частные соединения теперь поддерживаются для всех типов услуг на уровнях Scale и Enterprise (включая однорепликные услуги). Это означает, что вы теперь можете использовать Private Link как для ваших производственных (крупномасштабных), так и для разработческих (маломасштабных) сред.
- **Резервные копии:** Все услуги теперь по умолчанию поставляются с одной резервной копией, и дополнительные резервные копии оплачиваются отдельно. Пользователи могут использовать настраиваемые параметры резервного копирования для управления дополнительными резервными копиями. Это означает, что услуги с меньшими требованиями к резервному копированию не должны платить более высокую объединенную цену. Пожалуйста, смотрите больше деталей в [FAQ по резервным копиям](https://clickhouse.com/support/program).
- **Усиленное шифрование:** Эта функция доступна в услугах уровня Enterprise, включая однорепликные услуги, в AWS и GCP. Услуги по умолчанию шифруются нашим ключом и могут быть переведены на их ключ для включения Ключей Шифрования, Управляемых Заказчиком (CMEK).
- **Единый вход (SSO):** Эта функция предлагается на уровне Enterprise и требует создания тикета в службу поддержки для активации для Организации. Пользователи, имеющие несколько Организаций, должны убедиться, что все их организации находятся на уровне Enterprise, чтобы использовать SSO для каждой организации.

## Базовый уровень {#basic-tier}

### Какие соображения для базового уровня? {#what-are-the-considerations-for-the-basic-tier}

Базовый уровень предназначен для небольших нагрузок – пользователи хотят развернуть небольшое аналитическое приложение, которое не требует высокой доступности, или работать над прототипом. Этот уровень не подходит для нагрузок, которые требуют масштабирования, надежности (DR/HA) и долговечности данных. Уровень поддерживает однорепликные услуги фиксированного размера 1x8GiB или 1x12GiB. Пожалуйста, обратитесь к документации и [политике поддержки](https://clickhouse.com/support/program) для получения дополнительной информации.

### Могут ли пользователи на базовом уровне получить доступ к Private Link и Private Service Connect? {#can-users-on-the-basic-tier-access-private-link-and-private-service-connect}

Нет, пользователи должны перейти на уровень Scale или Enterprise, чтобы получить доступ к этой функции.

### Могут ли пользователи на базовом и уровне Scale настроить SSO для организации? {#can-users-on-the-basic-and-scale-tiers-set-up-sso-for-the-organization}

Нет, пользователи должны перейти на уровень Enterprise, чтобы получить доступ к этой функции.

### Могут ли пользователи запускать однорепликные услуги на всех уровнях? {#can-users-launch-single-replica-services-in-all-tiers}

Да, однорепликные услуги поддерживаются на всех трех уровнях. Пользователи могут масштабироваться, но не могут перейти на однореплику.

### Могут ли пользователи увеличить/уменьшить размер и добавить больше реплик на базовом уровне? {#can-users-scale-updown-and-add-more-replicas-on-the-basic-tier}

Нет, услуги на этом уровне предназначены для поддержки небольших и фиксированных нагрузок (однореплика `1x8GiB` или `1x12GiB`). Если пользователи нуждаются в масштабировании или добавлении реплик, им будет предложено перейти на уровни Scale или Enterprise.

## Уровень Scale {#scale-tier}

### Какие уровни на новых тарифах (Basic/Scale/Enterprise) поддерживают отделение вычислений? {#which-tiers-on-the-new-plans-basicscaleenterprise-support-compute-compute-separation}

Только уровни Scale и Enterprise поддерживают отделение вычислений. Также обратите внимание, что эта возможность требует запуска как минимум родительской услуги с 2+ репликами.

### Могут ли пользователи на устаревших планах (Production/Development) получить доступ к отделению вычислений? {#can-users-on-the-legacy-plans-productiondevelopment-access-compute-compute-separation}

Отделение вычислений не поддерживается в существующих услугах Development и Production, за исключением пользователей, которые уже принимали участие в Private Preview и Beta. Если у вас есть дополнительные вопросы, пожалуйста, свяжитесь со [службой поддержки](https://clickhouse.com/support/program).

## Уровень Enterprise {#enterprise-tier}

### Какие различные аппаратные профили поддерживаются для уровня Enterprise? {#what-different-hardware-profiles-are-supported-for-the-enterprise-tier}

Уровень Enterprise будет поддерживать стандартные профили (соотношение 1:4 vCPU:память), а также **высокопамятные (1:8)** и **высокопроцессорные (1:2)** настраиваемые профили, предоставляя пользователям больше гибкости в выборе конфигурации, которая лучше всего подходит под их нужды. Уровень Enterprise будет использовать совместно используемые вычислительные ресурсы, развернутые вместе с уровнями Basic и Scale.

### Какие функции предлагаются исключительно на уровне Enterprise? {#what-are-the-features-exclusively-offered-on-the-enterprise-tier}

- **Настраиваемые профили:** Опции выбора типа экземпляра стандартные профили (1:4 vCPU: память) и **высокопамятные (1:8)** и **высокопроцессорные (1:2)** настраиваемые профили.
- **Безопасность уровня Enterprise:**
    - **Единый вход (SSO)**
    - **Усиленное шифрование:** Для услуг AWS и GCP. Услуги по умолчанию шифруются нашим ключом и могут быть переведены на их ключ для включения Ключей Шифрования, Управляемых Заказчиком (CMEK).
- **Запланированные обновления:** Пользователи могут выбрать день недели/временной интервал для обновлений, как для базы данных, так и для облачных релизов.  
- **Соответствие HIPAA:** Заказчик должен подписать Соглашение бизнес-партнера (BAA) через юридический отдел, прежде чем мы включим регионы, соответствующие HIPAA, для них.
- **Частные регионы:** Это не поддерживается в режиме самообслуживания и требует от пользователей маршрутизировать запросы через отдел продаж sales@clickhouse.com.
- **Экспорт резервных копий** в облаковый аккаунт клиента.
