# Ошибки. Основание. Строгие и нестрогие функции. Ленивое и энергичное исполнение

Давайте посмотрим на на выражение, определенное рекурсивно

``` haskell
bot :: Bool
bot = not bot
```

Заметим, что его значение не True или False, а ⊥ — значение, разделяемое всеми типами

``` haskell 
⊥ :: forall {a}. a
```

Так же все ошибки имеют данное значение

``` haskell
GHCi> :set -fprint-explicit-foralls -- makes GHC print explicit forall quantification at the top level of a type; normally this is suppressed.
GHCi> :t undefined
undefined :: forall {a}. a
GHCi> :t error
error :: forall {a}. [Char] -> a
```

Нестрогие функции по аргументу это функции игнорирующие значение этого аргумента, например:

``` haskell 
GHCi> ignore x = 42
GHCi> ignore undefined
42
GHCi> ignore bot
42
```

А вот если функция строгая (не игнорирует аргумент), то выполняется:

``` haskell
f ⊥ = ⊥
```

Для форсированного вычисления используют `seq` , она синтаксически похожа на \a b -> b. Но она нарушает ленивую семантику языка, позволяя форсировать вычисление без необходимости.

``` haskell
GHCi> seq undefined 42
*** Exception: Prelude.undefined
GHCi> seq (id undefined) 42
*** Exception: Prelude.undefined
```

* seq производит вычисление своего первого аргумента, если в нем имеется редекс на верхнем уровне. 
* Однако конструкторы данных, лямбда-абстракции и частично примененные функции, являясь «значениями», обеспечивают барьер для распространения ⊥
``` haskell 
GHCi> seq (undefined,undefined) 42
42
GHCi> seq (\x -> undefined) 42
42
GHCi> seq ((+) undefined) 42
4
```
В коде выше не редексы - слабая головная нормальная форма (weak head
normal form, WHNF). 
Через seq определяется энергичная аппликация (с вызовом-по-значению)
``` haskell 
infixr 0 $!
($!) :: (a -> b) -> a -> b
f $! x = x `seq` f x
```
``` haskell 
GHCi> ignore undefined
42
GHCi> ignore $ undefined
42
GHCi> ignore $! undefined
*** Exception: Prelude.undefined
```
Теперь форсирование приводит к «худшей определенности». Видно на примере выше, так как с помощью `seq` мы проверяем на undefined