---
sidebar_label: 'Улучшенное Шифрование'
slug: /cloud/security/cmek
title: 'Ключи Шифрования, Управляемые Клиентом (CMEK)'
description: 'Узнайте больше о ключах шифрования, управляемых клиентом'
---

import Image from '@theme/IdealImage';
import EnterprisePlanFeatureBadge from '@theme/badges/EnterprisePlanFeatureBadge'
import cmek_performance from '@site/static/images/_snippets/cmek-performance.png';


# ClickHouse Улучшенное Шифрование

<EnterprisePlanFeatureBadge feature="Улучшенное Шифрование" support="true"/>

Данные в покое по умолчанию шифруются с использованием управляемых поставщиком облака ключей AES 256. Клиенты могут включить Прозрачное Шифрование Данных (TDE), чтобы предоставить дополнительный уровень защиты для данных сервиса, или предоставить свой собственный ключ для реализации Ключей Шифрования, Управляемых Клиентом (CMEK) для своего сервиса.

Улучшенное шифрование в настоящее время доступно в сервисах AWS и GCP. Azure будет доступен в ближайшее время.

## Прозрачное Шифрование Данных (TDE) {#transparent-data-encryption-tde}

TDE необходимо включить при создании сервиса. Существующие сервисы не могут быть зашифрованы после создания.

1. Выберите `Создать новый сервис`
2. Назовите сервис
3. Выберите AWS или GCP в качестве облачного провайдера и нужный регион из выпадающего меню
4. Нажмите на выпадающее меню для функций Enterprise и переключите Включить Прозрачное Шифрование Данных (TDE)
5. Нажмите Создать сервис

## Ключи Шифрования, Управляемые Клиентом (CMEK) {#customer-managed-encryption-keys-cmek}

:::warning
Удаление ключа KMS, используемого для шифрования сервиса ClickHouse Cloud, приведет к остановке вашего сервиса ClickHouse и данные станут недоступными, включая существующие резервные копии. Чтобы предотвратить случайную потерю данных при ротации ключей, вы можете сохранить старые ключи KMS в течение определенного времени перед удалением. 
:::

Как только сервис зашифрован с помощью TDE, клиенты могут обновить ключ, чтобы включить CMEK. Сервис автоматически перезапустится после обновления настроек TDE. В этом процессе старый ключ KMS расшифровывает ключ шифрования данных (DEK), а новый ключ KMS повторно шифрует DEK. Это гарантирует, что сервис при перезапуске будет использовать новый ключ KMS для операций шифрования в дальнейшем. Этот процесс может занять несколько минут.

<details>
    <summary>Включить CMEK с помощью AWS KMS</summary>
    
1. В ClickHouse Cloud выберите зашифрованный сервис
2. Нажмите на Настройки слева
3. В нижней части экрана разверните информацию о сетевой безопасности
4. Скопируйте ID роли шифрования (AWS) или Учетную запись Сервиса Шифрования (GCP) - вам это потребуется на этапе ниже
5. [Создайте ключ KMS для AWS](https://docs.aws.amazon.com/kms/latest/developerguide/create-keys.html)
6. Нажмите на ключ
7. Обновите политику ключа AWS следующим образом:
    
    ```json
    {
        "Sid": "Разрешить доступ ClickHouse",
        "Effect": "Allow",
        "Principal": {
            "AWS": "{ ID роли шифрования }"
        },
        "Action": [
            "kms:Encrypt",
            "kms:Decrypt",
            "kms:ReEncrypt",
            "kms:DescribeKey"
        ],
        "Resource": "*"
    }
    ```
    
10. Сохраните Политику ключа
11. Скопируйте ARN ключа
12. Вернитесь в ClickHouse Cloud и вставьте ARN ключа в раздел Прозрачное Шифрование Данных в Настройках Сервиса
13. Сохраните изменения
    
</details>

<details>
    <summary>Включить CMEK с помощью GCP KMS</summary>

1. В ClickHouse Cloud выберите зашифрованный сервис
2. Нажмите на Настройки слева
3. В нижней части экрана разверните информацию о сетевой безопасности
4. Скопируйте Учетную запись Сервиса Шифрования (GCP) - вам это потребуется на этапе ниже
5. [Создайте ключ KMS для GCP](https://cloud.google.com/kms/docs/create-key)
6. Нажмите на ключ
7. Предоставьте следующие разрешения Учетной записи Сервиса Шифрования GCP, скопированной на этапе 4 выше.
   - CryptoKey Shifrator/Dešifrator Cloud KMS
   - Просмотритель Cloud KMS
10. Сохраните разрешение на ключ
11. Скопируйте Путь Ресурса Ключа
12. Вернитесь в ClickHouse Cloud и вставьте Путь Ресурса Ключа в раздел Прозрачное Шифрование Данных в Настройках Сервиса
13. Сохраните изменения
    
</details>

## Ротация Ключей {#key-rotation}

После настройки CMEK, ротируйте ключ, следуя процедурам выше для создания нового ключа KMS и предоставления разрешений. Вернитесь в настройки сервиса, чтобы вставить новый ARN (AWS) или Путь Ресурса Ключа (GCP) и сохранить настройки. Сервис перезапустится, чтобы применить новый ключ.

## Резервное Копирование и Восстановление {#backup-and-restore}

Резервные копии шифруются с использованием того же ключа, что и связанный сервис. Когда вы восстанавливаете зашифрованную резервную копию, создается зашифрованный экземпляр, который использует тот же ключ KMS, что и оригинальный экземпляр. При необходимости вы можете ротировать ключ KMS после восстановления; см. [Ротация Ключей](#key-rotation) для получения дополнительной информации.

## Поллер Ключей KMS {#kms-key-poller}

При использовании CMEK действительность предоставленного ключа KMS проверяется каждые 10 минут. Если доступ к ключу KMS недействителен, сервис ClickHouse будет остановлен. Для возобновления работы сервиса восстановите доступ к ключу KMS, следуя шагам в этом руководстве, а затем перезапустите сервис.

## Производительность {#performance}

Как указано на этой странице, мы используем встроенный ClickHouse [Виртуальная Файловая Система для Функции Шифрования Данных](/operations/storing-data#encrypted-virtual-file-system) для шифрования и защиты ваших данных.

Алгоритм, используемый для этой функции, - `AES_256_CTR`, что подразумевает снижение производительности на 5-15% в зависимости от нагрузки:

<Image img={cmek_performance} size="lg" alt="Штраф за Производительность CMEK" />
