---
description: 'Документация для Clickhouse Obfuscator'
slug: /operations/utilities/clickhouse-obfuscator
title: 'clickhouse-obfuscator'
---

Простой инструмент для обфускации данных таблиц.

Он читает входную таблицу и создает выходную таблицу, которая сохраняет некоторые свойства входных данных, но содержит другие данные. 
Это позволяет публиковать почти реальные производственные данные для использования в бенчмарках.

Он предназначен для сохранения следующих свойств данных:
- кардинальности значений (число различных значений) для каждой колонки и каждого кортежа колонок;
- условные кардинальности: число различных значений одной колонки при условии на значение другой колонки;
- распределения вероятностей абсолютных значений целых чисел; знак знаковых целых чисел; экспонента и знак для чисел с плавающей запятой;
- распределения вероятностей длины строк;
- вероятность нулевых значений чисел; пустых строк и массивов, `NULL`s;

- коэффициент сжатия данных при сжатии с помощью LZ77 и семейства кодеков энтропии;
- непрерывность (размеры различий) временных значений по всей таблице; непрерывность значений с плавающей запятой;
- дата компоненты значений `DateTime`;

- корректность UTF-8 строковых значений;
- строковые значения выглядят естественно.

Большинство свойств выше применимы для тестирования производительности:

чтение данных, фильтрация, агрегация и сортировка будут работать почти с той же скоростью, что и с исходными данными из-за сохраненных кардинальностей, размеров, коэффициентов сжатия и т. д.

Он работает детерминированным образом: вы задаете значение начального значения, и преобразование определяется входными данными и начальным значением. 
Некоторые преобразования являются взаимно однозначными и могут быть обращены, поэтому вам нужно иметь большое начальное значение и хранить его в секрете.

Он использует некоторые криптографические примитивы для преобразования данных, но с криптографической точки зрения он не делает это должным образом, поэтому не следует считать результат безопасным, если у вас нет другой причины. Результат может сохранить некоторые данные, которые вы не хотите публиковать.

Он всегда оставляет числа 0, 1, -1, даты, длины массивов и нулевые флаги точно такими же, как в исходных данных. 
Например, у вас есть колонка `IsMobile` в вашей таблице со значениями 0 и 1. В преобразованных данных она будет иметь то же значение.

Таким образом, пользователь сможет точно вычислить соотношение мобильного трафика.

Приведем еще один пример. Когда у вас есть некоторые конфиденциальные данные в вашей таблице, например, адреса электронной почты пользователей, и вы не хотите публиковать ни один отдельный адрес электронной почты. 
Если ваша таблица достаточно велика и содержит множество различных электронных адресов и ни один адрес электронной почты не имеет очень высокой частоты по сравнению с другими, она анонимизирует все данные. Но если у вас небольшое количество различных значений в колонке, она может воспроизвести некоторые из них. 
Вам следует изучить работающий алгоритм этого инструмента и настроить его параметры командной строки.

Этот инструмент работает хорошо только при наличии как минимум умеренного количества данных (по крайней мере, 1000 строк).
