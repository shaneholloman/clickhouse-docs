---
description: 'Документация по функциям сравнения'
sidebar_label: 'Сравнение'
sidebar_position: 35
slug: /sql-reference/functions/comparison-functions
title: 'Функции Сравнения'
---


# Функции Сравнения

Функции сравнения ниже возвращают `0` или `1` с типом [UInt8](/sql-reference/data-types/int-uint). Можно сравнивать только значения в одной группе (например, `UInt16` и `UInt64`), но не между группами (например, `UInt16` и `DateTime`). Сравнение чисел и строк возможно, так же как и сравнение строк с датами и дат с временем. Для кортежей и массивов сравнение происходит лексикографически, что означает, что сравнение осуществляется для каждого соответствующего элемента левой и правой стороны кортежа/массива.

Следующие типы могут быть сравнены:
- числа и десятичные дроби
- строки и фиксированные строки
- даты
- даты с временем
- кортежи (лексикографическое сравнение)
- массивы (лексикографическое сравнение)

:::note
Строки сравниваются посимвольно. Это может привести к неожиданным результатам, если одна из строк содержит многобайтовые символы в кодировке UTF-8.
Строка S1, которая имеет другую строку S2 в качестве префикса, считается длиннее S2.
:::

## equals, `=`, `==` операторы {#equals}

**Синтаксис**

```sql
equals(a, b)
```

Псевдоним:
- `a = b` (оператор)
- `a == b` (оператор)

## notEquals, `!=`, `<>` операторы {#notequals}

**Синтаксис**

```sql
notEquals(a, b)
```

Псевдоним:
- `a != b` (оператор)
- `a <> b` (оператор)

## less, `<` оператор {#less}

**Синтаксис**

```sql
less(a, b)
```

Псевдоним:
- `a < b` (оператор)

## greater, `>` оператор {#greater}

**Синтаксис**

```sql
greater(a, b)
```

Псевдоним:
- `a > b` (оператор)

## lessOrEquals, `<=` оператор {#lessorequals}

**Синтаксис**

```sql
lessOrEquals(a, b)
```

Псевдоним:
- `a <= b` (оператор)

## greaterOrEquals, `>=` оператор {#greaterorequals}

**Синтаксис**

```sql
greaterOrEquals(a, b)
```

Псевдоним:
- `a >= b` (оператор)
