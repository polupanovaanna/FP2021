# Билет 20. Рекурсивные типы. µ-нотация.

## Создаём алгебру типов
Задаём сумму на ней
Задаём декартово произведение
Изоморфизм типов

## Теперь к рекурсии

```haskell
L = 1 + A + A^2 + A^3 + A^4 + ...
```

Дальше идут вещи от которых Храброву стало бы плохо

```haskell
L = 1 + A*L
```

```haskell
data List a = Nil | Cons a (List a)
```

Рекурсивное уравнение на типы 

```haskell
L = (λX. 1 + A ∗ X) L

L = FIX λX. 1 + A ∗ X
```

L - это некоторая неподвижная точка для данного терма

Для конструкции `FIX λX. T[X]` часто используют обозначение
`µX. T[X]`


Итоговое решение для случая списка записывается теперь так

```haskell
List A = µX. 1 + A ∗ X
```

Ещё примеры решений:

Числа

```haskell
data Nat = Zero | Succ Nat

Nat = µX. 1 + X
```

Деревья

```haskell
data Tree a = Empty | Node a (Tree a) (Tree a)

Tree = λA. µX. 1 + A ∗ X
```

## реализация на хаскелле

TODO если останется время

# Билет 6. Объявления type и newtype. Метки полей.

## 

в Haskell очень часто само имя или конструктор типа ялвяется очень большим

ключевое слово type (то есть синоним типа)

Пример:

```haskell
type String = [Char]
```

Они абсолютно идентичны изначальному типу с точки зрения языка

```haskell
GHCi> let xs = ['a', 'b', 'c'] :: String
GHCi> let ys = ['a', 'b', 'c'] :: [Char]
GHCi> xs == ys
True
```

## newtype

Ключевое слово newtype задаёт новый тип. У него есть конструктор, упаковывающий уже известный тип

```haskell
newtype AgeNT = AgeNT Int

newtype AgeNT = AgeNT { getAgeNT :: Int }
```

```haskell
GHCi> age = AgeNT 42
GHCi> :t age
age :: AgeNT
GHCi> age
AgeNT {getAgeNT = 42}
GHCi> getAgeNT age
42
```


Особенности newtype:

1.все представители пропадают, требуется deriving
2.newtype всегда имеет 1 конструктор
3.newtype более ленив, засчёт чего местами более эффективен.

```haskell
GHCi> ignoreNT undefined
42
GHCi> ignoreDT undefined
*** Exception: Prelude.undefined
```


ignoreNT — это переменная n типа Int, поэтому необходимость форсировать вычисление undefined отсутствует.

## Метки полей

```haskell
data Point a = Pt a a
  deriving Show
```

Давайте попробуем обратиться к Полям типа-произведения

```haskell
ptx :: Point a -> a
ptx (Pt x _) = x
pty :: Point a -> a
pty (Pt _ y) = y
```

это не круто и муторно, поэтому в haskell придумали для этого инструмент меток полей

```haskell
data Point a = Pt { ptX :: a, ptY :: a }
  deriving Show
```

ptX и ptY работают ровно также, как раньше, но добавляют много удобств

Возможности:

1.Инициализировать запись

```haskell
GHCi> myPt1 = Pt {ptY = 2, ptX = 3}
GHCi> myPt1
Pt {ptX = 3, ptY = 2}
```

Можно даже инициализировать не всё, но не советуется

2.Связывать с метками в образце

```haskell
absP' Pt {ptX = x, ptY = y} = sqrt (x ^ 2 + y ^ 2)
```

Было такое, но оно проиграет в некоторых ситуациях

```haskell
absP'' (Pt x y) = sqrt (x ^ 2 + y ^ 2)
```

3.Обновлять поля

```haskell
GHCi> myPt3 = Pt {ptX = 7, ptY = 8}
GHCi> myPt3 {ptX = 42}
Pt {ptX = 42, ptY = 8}
```



Поведение меток в разных типах, коллизии:

Можно

```haskell
data Homo = Known { name :: String, male :: Bool }
| Unknown { male :: Bool }
GHCi> john = Known "John" True
GHCi> stranger = Unknown False
GHCi> male john
True
GHCi> male stranger
False
```

Нельзя:

```haskell
data Bad = Bad { male :: Bool }
```

потому что метки определены глобально
