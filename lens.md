# Теория. Билет 24 "Линзы ван Лаарховена"

## Определение

Линза — инструмент для манипулирования подструктурой
некоторой структуры данных. Лежат, например, в ```Control.Lens```.

## Примеры

```_1``` и ```_2``` — линзы для доступа к первому и второму элемениу пары соответственно.

```haskell
GHCi> view _1 (7,8)
7
GHCi> (7,8) ^. _2
8
```
## Свойства линз

Композиция линз — это линза:

```haskell
GHCi> view (_1 . _2) ((7,8),9) --ПОРЯДОК КОМПОЗИЦИИ ОБРАТНЫЙ!!!
8
GHCi> (7,8) ^. _2
8

GHCi> ((7,8),9) ^. _1 -- ^. инфиксный эквивалент view
(7,8)
GHCi> ((7,8),9) ^. _1 . _2
8
```
## Модификация через линзы

Можно модифицировать структуру, а не только смотреть

```haskell
GHCi> set _1 42 (7,8) -- засунь-ка на первую позицию
(42,8)
GHCi> set _1 "Hello" (7,8)
("Hello",8)
GHCi> over _1 length ("Hello","World") -- примени-ка функцию к первой позиции
(5,"World")

-- в инфиксной записи

GHCi> _1 .~ "Hello"  $ (7,8) -- set
("Hello",8)
GHCi> (7, 8) & _1 .~ "Hello"
("Hello",8)
GHCi> _1 %~ (^2) $ (7,8) -- over
(49,8)
```

## Законы линзы
Важно понимать, что линза — это абстрактный тип данных составленный из геттера и сеттера. Тип линзы:
```haskell
lens :: (s -> a) -> (s -> a -> s) -> Lens s a
```
Линза дает возможность проманипулировать над элементом типа ```a``` структуры типа ```s```.

Собственно законы:
```
view l (set l v s) ≡ v -- подставили и показали, получили то, что подставили
set l (view l s) s ≡ s -- показали и подставили то, что показали, по итогу не поменялось
set l v' (set l v s) ≡ set l v' s -- подставили два раза, результат ожидаемый
```
## Наивная реализация линз
Казалось сделать линзу просто. Например, вот так:
```haskell
data LensNaive s a = MkLens (s -> a) (s -> a -> s)

_1Naive :: LensNaive (a,b) a
_1Naive = MkLens (\(x,_) -> x) (\(_,y) v -> (v,y))

viewNaive :: LensNaive s a -> s -> a
viewNaive (MkLens get _) s = get s
```
Проблема в том, что непонятно как это расширять (композиция сломалась) и в том, что каждый раз дергаем конструктор, а это долго. Решение всего этого ниже.

## Линзы ван Лаарховена

Как написано выше: "Линза дает возможность проманипулировать над элементом типа ```a``` структуры типа ```s```."

Теперь введем еще одно определение.

Линза — это функция, которая превращает вложение ```a``` в
функтор ```f``` во вложение ```s``` в этот функтор.

```haskell
type Lens s a = forall f. Functor f => (a -> f a) -> s -> f s
```

Упаковка геттера и сеттера:
```haskell
-- (s -> a) -> (s -> a -> s) -> (a -> f a) -> s -> f s
lens :: (s -> a) -> (s -> a -> s) -> Lens s a
lens get set = \ret s -> fmap (set s) (ret $ get s)
```
Тогда уже знакомый нам ```_1``` выглядит так:

```haskell
_1 :: Lens (a,b) a -- (a -> f a) -> (a,b) -> f (a,b)
_1 = lens (\(x,_) -> x) (\(_,y) v -> (v,y))
```

Для полноты картины ниже реализации ```view```, ```over```, ```set```. У них всех одна общая идея: давайте просто подберем для каждой подходящий функтор (как понять, какой подходящий? походу на интуиции и исходя из законов для линз, нужно добавить какие-то комментарии).

```haskell
newtype Const a x = Const { getConst :: a }
-- Const :: a -> Const a x
-- getConst :: Const a x -> a
instance Functor (Const a) where
  fmap _ (Const v) = Const v

-- ((a -> Const a a) -> s -> Const a s) -> s -> a
view :: Lens s a -> s -> a
view lns s = getConst (lns Const s)

newtype Identity a = Identity { runIdentity :: a }
-- Identity :: a -> Identity a
-- runIdentity :: Identity a -> a
instance Functor Identity where
  fmap f (Identity x) = Identity (f x)

-- ((a->Identity a)->s->Identity s) -> (a -> a) -> s -> s
over :: Lens s a -> (a -> a) -> s -> s
over lns fn s = runIdentity $ lns (Identity . fn) s

-- ((a->Identity a)->s->Identity s) -> a -> s -> s
set :: Lens s a -> a -> s -> s
set lns a s = over lns (const a) s
```
