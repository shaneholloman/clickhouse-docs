---
slug: /guides/sre/scaling-clusters
sidebar_label: 'Перебалансировка Шардов'
sidebar_position: 20
description: 'ClickHouse не поддерживает автоматическую перебалансировку шардов, поэтому мы предоставляем некоторые лучшие практики для перебалансировки шардов.'
title: 'Перебалансировка Данных'
---


# Перебалансировка Данных

ClickHouse не поддерживает автоматическую перебалансировку шардов. Тем не менее, существуют методы перебалансировки шардов в порядке предпочтения:

1. Настройте шард для [распределенной таблицы](/engines/table-engines/special/distributed.md), позволяя записям быть смещёнными к новому шард. Это потенциально может вызвать дисбаланс нагрузки и горячие точки в кластере, но может быть жизнеспособным в большинстве сценариев, когда пропускная способность записи не является крайне высокой. Это не требует от пользователя изменения целевого назначения записи, т.е. оно может оставаться как распределенная таблица. Это не помогает с перебалансировкой существующих данных.

2. В качестве альтернативы (1) измените существующий кластер и записывайте исключительно в новый шард до тех пор, пока кластер не будет сбалансирован - вручную взвешивая записи. Это имеет те же ограничения, что и (1).

3. Если вам нужно перебалансировать существующие данные и вы разделили свои данные на партиции, подумайте о том, чтобы отсоединить партиции и вручную переместить их на другой узел перед повторным присоединением к новому шард. Это более ручной процесс, чем последующие техники, но может быть быстрее и менее ресурсоемким. Это ручная операция и поэтому необходимо учитывать перебалансировку данных.

4. Экспортируйте данные из исходного кластера в новый кластер с помощью [INSERT FROM SELECT](/sql-reference/statements/insert-into.md/#inserting-the-results-of-select). Это не будет производительным при очень больших наборах данных и может вызвать значительный ввод-вывод на исходном кластере и использовать значительные сетевые ресурсы. Это представляет собой последний вариант.
