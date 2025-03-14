---
sidebar_label: 'Аутентификация пользователя с помощью сертификатов SSL'
sidebar_position: 3
slug: /guides/sre/ssl-user-auth
---


# Настройка SSL-сертификата пользователя для аутентификации
import SelfManaged from '@site/i18n/ru/docusaurus-plugin-content-docs/current/_snippets/_self_managed_only_no_roadmap.md';

<SelfManaged />

Этот гайд предоставляет простые и минимальные настройки для настройки аутентификации с SSL-сертификатами пользователей. Учебное пособие основано на руководстве по [Настройке SSL-TLS](../configuring-ssl.md).

:::note
Аутентификация пользователя по SSL поддерживается только при использовании `https` или нативных интерфейсов. В настоящее время она не используется в gRPC или портах эмуляции PostgreSQL/MySQL.
:::

## 1. Создание SSL-сертификатов пользователя {#1-create-ssl-user-certificates}

:::note
В этом примере используются самоподписанные сертификаты с самоподписанным CA. Для производственных сред создайте CSR и отправьте вашей команде PKI или поставщику сертификатов для получения подходящего сертификата.
:::

1. Сгенерируйте запрос на подпись сертификата (CSR) и ключ. Основной формат выглядит следующим образом:
    ```bash
    openssl req -newkey rsa:2048 -nodes -subj "/CN=<my_host>:<my_user>"  -keyout <my_cert_name>.key -out <my_cert_name>.csr
    ```
    В этом примере мы будем использовать это для домена и пользователя, которые будут использоваться в этой тестовой среде:
    ```bash
    openssl req -newkey rsa:2048 -nodes -subj "/CN=chnode1.marsnet.local:cert_user"  -keyout chnode1_cert_user.key -out chnode1_cert_user.csr
    ```
    :::note
    CN является произвольным и любая строка может быть использована в качестве идентификатора для сертификата. Он будет использоваться при создании пользователя на следующих шагах.
    :::

2.  Сгенерируйте и подпишите новый сертификат пользователя, который будет использоваться для аутентификации. Основной формат выглядит следующим образом:
    ```bash
    openssl x509 -req -in <my_cert_name>.csr -out <my_cert_name>.crt -CA <my_ca_cert>.crt -CAkey <my_ca_cert>.key -days 365
    ```
    В этом примере мы будем использовать это для домена и пользователя, которые будут использоваться в этой тестовой среде:
    ```bash
    openssl x509 -req -in chnode1_cert_user.csr -out chnode1_cert_user.crt -CA marsnet_ca.crt -CAkey marsnet_ca.key -days 365
    ```

## 2. Создание SQL пользователя и предоставление привилегий {#2-create-a-sql-user-and-grant-permissions}

:::note
Для получения подробной информации о том, как включить SQL пользователей и установить роли, обратитесь к руководству [Определение SQL пользователей и ролей](index.md).
:::

1. Создайте SQL пользователя, определенного для использования аутентификации сертификатом:
    ```sql
    CREATE USER cert_user IDENTIFIED WITH ssl_certificate CN 'chnode1.marsnet.local:cert_user';
    ```

2. Предоставьте привилегии новому пользователю с сертификатом:
    ```sql
    GRANT ALL ON *.* TO cert_user WITH GRANT OPTION;
    ```
    :::note
    Пользователю предоставляются полные привилегии администратора в этом упражнении в демонстрационных целях. Обратитесь к [документации ClickHouse по RBAC](/guides/sre/user-management/index.md) для настройки разрешений.
    :::

    :::note
    Рекомендуется использовать SQL для определения пользователей и ролей. Тем не менее, если вы в настоящее время определяете пользователей и роли в конфигурационных файлах, пользователь будет выглядеть так:
    ```xml
    <users>
        <cert_user>
            <ssl_certificates>
                <common_name>chnode1.marsnet.local:cert_user</common_name>
            </ssl_certificates>
            <networks>
                <ip>::/0</ip>
            </networks>
            <profile>default</profile>
            <access_management>1</access_management>
            <!-- дополнительные параметры-->
        </cert_user>
    </users>
    ```
    :::


## 3. Тестирование {#3-testing}

1. Скопируйте сертификат пользователя, ключ пользователя и сертификат CA на удаленный узел.

2. Настройте OpenSSL в [конфигурации клиента ClickHouse](/interfaces/cli.md#configuration_files) с сертификатом и путями.

    ```xml
    <openSSL>
        <client>
            <certificateFile>my_cert_name.crt</certificateFile>
            <privateKeyFile>my_cert_name.key</privateKeyFile>
            <caConfig>my_ca_cert.crt</caConfig>
        </client>
    </openSSL>
    ```

3. Запустите `clickhouse-client`.
    ```bash
    clickhouse-client --user <my_user> --query 'SHOW TABLES'
    ```
    :::note
    Обратите внимание, что пароль, переданный в clickhouse-client, игнорируется, когда сертификат указан в конфигурации.
    :::


## 4. Тестирование HTTP {#4-testing-http}

1. Скопируйте сертификат пользователя, ключ пользователя и сертификат CA на удаленный узел.

2. Используйте `curl` для тестирования пример SQL-команды. Основной формат выглядит следующим образом:
    ```bash
    echo 'SHOW TABLES' | curl 'https://<clickhouse_node>:8443' --cert <my_cert_name>.crt --key <my_cert_name>.key --cacert <my_ca_cert>.crt -H "X-ClickHouse-SSL-Certificate-Auth: on" -H "X-ClickHouse-User: <my_user>" --data-binary @-
    ```
    Например:
    ```bash
    echo 'SHOW TABLES' | curl 'https://chnode1:8443' --cert chnode1_cert_user.crt --key chnode1_cert_user.key --cacert marsnet_ca.crt -H "X-ClickHouse-SSL-Certificate-Auth: on" -H "X-ClickHouse-User: cert_user" --data-binary @-
    ```
    Вывод будет похож на следующий:
    ```response
    INFORMATION_SCHEMA
    default
    information_schema
    system
    ```
    :::note
    Обратите внимание, что пароль не был указан, сертификат используется вместо пароля и именно таким образом ClickHouse будет аутентифицировать пользователя.
    :::


## Резюме {#summary}

В этой статье были рассмотрены основы создания и настройки пользователя для аутентификации по SSL-сертификату. Этот метод может быть использован с `clickhouse-client` или любыми клиентами, которые поддерживают `https` интерфейс и где могут быть установлены HTTP-заголовки. Сгенерированный сертификат и ключ должны храниться в секрете и иметь ограниченный доступ, так как сертификат используется для аутентификации и авторизации пользователя для операций в базе данных ClickHouse. Обращайтесь с сертификатом и ключом так, как если бы это были пароли.
