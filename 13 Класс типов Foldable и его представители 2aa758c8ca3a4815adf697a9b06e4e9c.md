# 13. Класс типов Foldable и его представители.

В классе Foldable определены функции foldr и foldl

```haskell
class Foldable t where
	foldr :: (a -> b -> b) -> b -> t a -> b
	foldl :: (b -> a -> b) -> b -> t a -> b
```

foldr и foldl – функции свертки, которые выполняют последовательную операцию с каждым элементом контейнера.

Примеры реализации для простейших контейнеров: 

**Для списка:** 

```haskell
instance Foldable [] where 
	foldr f ini [] = ini
	foldr f ini (x:xs) = f x (foldr f ini xs)

	foldl f ini [] = ini
	foldl f ini (x:xs) = foldl f (f ini x) xs
```

**Для Maybe:**

```haskell
instance Foldable Maybe where 
	foldr _ ini Nothing = ini
	foldr f ini (Just x) = f x ini

	foldl _ ini Nothing = ini
	foldl f ini (Just x) = f ini x
```

**Для Пары:**

В паре свертка использует один из элементов пары в зависимости от того правая она или левая

```haskell
instance Foldable ((,) a) where
	foldMap f (_, y) = f y 
	foldr f z (_, y) = f y z
```

Также мы можем выполнять свертку нелинейных типов данных. Например, собственной структуры дерево. Она будет различна в зависимости от того,

 в каком порядке мы хотим обойти дерево.

Если в контейнере лежит элемент типа Monoid, то мы можем для него определить функцию fold, которая в качестве инициализирующего значения использует 

нейтральный элемент, а в качестве функции бинарную оперцаию, которая для этого типа определена. (В классе моноид определена похожая функция,

которая называется mconcat)

```haskell
fold :: Monoid m => t m -> m
fold = foldr mappend mempty -- mconcat
```

Также существует функция foldMap, которая аргументом принимает функцию, умеющую превращать элемент, лежащий в контейнере, в моноид, который 

мы также умеем преобразовывать. 

```haskell
foldMap :: Monoid m => (a -> m) -> t a -> m
-- foldMap f cont = fold (fmap f cont)
foldMap f cont = foldr (mappend . f) mempty
```

Первая реализация очень узко направленная, так как позволяет работать только с теми контейнерами, у кого определена функция fmap, поэтому ее лучше

не использовать. Вторая реализация хорошая.

Также в классах типа Foldable определены дополнительные полезные функции, например, функции sum, product, null, foldr’, foldl’ (Энергичная свертка), 

foldr1, foldl1 (Для непустых списков) и т.д. Эти функции имеют реализацию по умолчанию через foldr, но мы также можем foldr определять через foldMap

**Минимально полное определение класса** – foldr или foldMap

Представителями класса Foldable являются список, Maybe, а также Set из Data.Set, Map k из Data.Map, Seq из Data.Sequence, Tree из Data.Tree и т.п.

**Полное определение класса Foldable** 

```haskell
class Foldable t where
  ...
  toList :: t a -> [a]
  null :: t a -> Bool
  null = foldr (\_ _ -> False) True
  length :: t a -> Int
  length = foldl' (\c _ -> c + 1) 0
  elem :: Eq a => a -> t a -> Bool
  sum, product :: Num a => t a -> a
  sum = getSum . foldMap Sum
  maximum, minimum :: Ord a => t a -> a
```

**Законы Foldable**

![Снимок экрана 2021-12-25 в 17.03.21.png](13%20%D0%9A%D0%BB%D0%B0%D1%81%D1%81%20%D1%82%D0%B8%D0%BF%D0%BE%D0%B2%20Foldable%20%D0%B8%20%D0%B5%D0%B3%D0%BE%20%D0%BF%D1%80%D0%B5%D0%B4%D1%81%D1%82%D0%B0%D0%B2%D0%B8%D1%82%D0%B5%D0%BB%D0%B8%202aa758c8ca3a4815adf697a9b06e4e9c/%D0%A1%D0%BD%D0%B8%D0%BC%D0%BE%D0%BA_%D1%8D%D0%BA%D1%80%D0%B0%D0%BD%D0%B0_2021-12-25_%D0%B2_17.03.21.png)