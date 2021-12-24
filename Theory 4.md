# Теория. Билет 4 "Кодирование булевых значений, кортежей в чистом бестиповом лямбда-исчислении"

## Булевы значения

Договоримся, что

```haskell
> tru = \x y → x
> fls = \x y → y
```

Конструкция if-then-else – это тоже функция!
Хотим:
```haskell
> iff B M N => M, если B = tru
> iff B M N => N, если B = fls
```

Ответ:
```haskell
> iff = \b m n → b m n
```

Проверим на примере:

```haskell
GHCi> iff tru 2 4
2
GHCi> iff fls 2 4
4
```

fun fact: iff = id
То есть iff, вообще говоря, бесполезна:)


Определим также логические операции not, or, and.

```haskell
> not = \b → b fls tru
> or  = \x y → x tru y
> and = \x y → x y fls
```


Можно ли определить not короче?

Ответ:
```haskell
> not' = \b x y → b y x
```

Заметьте, что это not и not' не эквивалентны, но при этом
на tru и fls работают одинаково, и потому обе нам подходят.

## Булевы значения


Договоримся, что список термов

```haskell
[A1, A2, A3, ..., Am]
```

-- это терм

```haskell
\c n -> c A1 (c A2 (c A3 (c ... (c Am n))))
```
Примеры:

```haskell
[]        = \c n -> n
[1]       = \c n -> c 1 n
[1, 2]    = \c n -> c 1 (c 2 n)
[1, 2, 3] = \c n -> c 1 (c 2 (c 3 n))
```

Далее следует задание основных операций над списками

определим конструкторы списка: nil и cons.
nil  возвращает пустой список,
cons принимает голову (элемент) и хвост (список) и конкатенирует их.

```haskell
> nil = [] = \c n -> n
```

```haskell
> cons = \h t -> (\c n -> c h (t c n))
```

```haskell
GHCi> cons 2 [3, 9]
[2, 3, 9]
```

Теперь вместо выдуманной нотации
[1, 2, 3]
можем писать честно:
```haskell
cons 1 (cons 2 (cons 3 nil))
```

Определим функцию nonempty, проверяющую, что список непуст:

```haskell
> nonempty = \l -> l (\x y -> tru) fls
```

```haskell
GHCi> nonempty []
fls
GHCi> nonempty [42]
tru
```


Определим функцию head, возвращающую голову списка:

```haskell
head = \l -> l tru undefined
```

```haskell
GHCi> head [42, 2, 6, 1]
42
```


Определим функцию sum, суммирующую элементы списка:

```haskell
sum = \l -> l plus 0
```

```haskell
GHCi> sum [1, 2, 3]
6
```
