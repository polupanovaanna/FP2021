# 15. Класс типов Applicative и его представители

[презентация](https://wiki.compscicenter.ru/images/6/6b/Fpc08HSE2021.pdf)

Идея: хотим для произвольного функтора `f :: * -> *` построить
```haskell
fmap2 :: (x -> y -> b) -> f x -> f y -> f b
fmap3 :: (x -> y -> z -> b) -> f x -> f y -> f x -> f b
...
```
К сожалению просто функтор так обощить нельзя. Поскольку `xs :: f x, ys :: f y` и `g :: x -> (y -> b)` имеем
```haskell
fmap g xs :: f (y -> b)     -- стрелка забралась в контейнер, нам нужен способ "вынуть" её

--придумаем функцию, чтобы её вынуть
ap :: f (y -> b) -> f y -> f b

fmap2 g xs ys    =  fmap g xs `ap` ys
fmap3 g xs ys zs = (fmap g xs `ap` ys) `ap` zs
```

Однако `ap` нельзя получить для произвольного функтора, поэтому определяют интерфейс

```haskell
class Functor f => Applicative f where
    pure  :: a -> f a
    (<*>) :: f (a -> b) -> f a -> f b   -- эта штука и называется ap

infixl 4 <*>
```

(про `pure` подробно написано у Москвина на слайдах в отдельном разделе под названием "Класс Pointed")

Оператор `<*>` похлж на оператор `($) :: (a -> b) -> a -> b`, но в "вычислительном контексте", задаваемым функтором.

На самом деле в библиотеке есть стандартный `fmap2`, он называется `liftA2`. И аналогично для `fmap3` есть `liftA3`.

</br></br>

## Полное определение класса типов `Applicative`

```haskell
infixl 4 <*>, *>, <*, <**>
class Functor f => Applicative f where
    {-# MINIMAL pure, ((<*>) | liftA2) #-}
    pure :: a -> f a

    (<*>) :: f (a -> b) -> f a -> f b
    (<*>) = liftA2 id

    liftA2 :: (a -> b -> c) -> f a -> f b -> f c
    liftA2 g a b = g <$> a <*> b

-- дополнительные фиговины, берут только значение оттуда, куда указывают
    (*>) :: f a -> f b -> f b       
    a1 *> a2 = (id <$ a1) <*> a2

    (<*) :: f a -> f b -> f a
    (<*) = liftA2 const
```

</br></br>

## Вспомогательные функции из `Control.Applicative`

```haskell
liftA :: Applicative f => (a -> b) -> f a -> f b
liftA f a = pure f <*> a
```

Писать `liftA = fmap` не хорошо, так как часто наоборот, содержательно реализуют как раз `Applicative`, а для функтора пишут определение через `fmap = liftA`

```haskell
liftA3 :: Applicative f => (a -> b -> c -> d)-> f a -> f b -> f c -> f d
liftA3 g a b c = g <$> a <*> b <*> c

(<**>) :: Applicative f => f a -> f (a -> b) -> f b
(<**>) = liftA2 (&)
```
Эта шняга с двумя звёздочками это как с одной звёздочкой, только выполнение происходит в обратном порядке и соответственно она имеет другую ассоциативностью

</br></br>

## Законы аппликативных функторов

- __Identity__ </br>
`pure id <*> v  = v`
- __Homomorphism__ </br>
`pure g <*> pure x = pure (g x)`
- __Interchange__ </br>
`u <*> pure x = pure ($ x) <*> u`
- __Composition__ </br>
`pure (.) <*> u <*> v <*> x = u <*> (v <*> x)`

Москвин про законы нам ничего не рассказал, так что тут мне дополнить нечем.

### __Закон, связывающий `Applicative` и `Functor`__
- `fmap g xs = pure g <*> xs` </br>
То есть `pure g` должно поднимать функцию `g` в наш контейнер таким образом, чтобы дальнейшая аппликация с помощью `ap` не меняла структуры контейнера.
Этот закон можно переписать используя инфиксный синоним `fmap`</br>
-  `g <$> xs = pure g <*> xs`


</br></br>

## Примеры аппликативных функторов

### Для `Maybe`
```haskell
instance Applicative Maybe where 
    pure           = Just       -- должно быть безэффектным
                                -- должно подымать значение в наш вычислительный контекст наиболее
                                -- безэффектным образом

    Nothing  <*> _ = Nothing
    (Just g) <*> x = fmap g x
```

Пример работы:
```GHCi
> Just (+) <*> Just 2 <*> Just 5
Just 7
> Just (+2) <*> Nothing
Nothing
> Just (+) <*> Just 2 <*> Just 5
Just 7
```
Так же по законам первое выражение можно переписать как

```GHCi
> Just (+) <*> Just 2 <*> Just 5
Just 7
> pure (+2) <*> Just 2 <*> Just 5
Just 7
> (+) <$> Just 2 <*> Just 5
Just 7
```
Последняя штука в точности `liftA2`.

### Для списка

Тут два варианта того, как можно реализовать - как зип и семантикой "каждый с каждым". 
```haskell
fs = [(2*), (3+), (4-)]
xs = [1, 2]
```

1. Список - контекст, задающий множественные результаты недетерминированного вычисления:
```haskell
fs <*> xs -> [(2*)1, (2*)2, (3+)1, (3+)2, (4-)1, (4-)2] -> [2,4,4,5,3,2]
```
2. Список - это коллекция упорядоченных элементов
```haskell
fs <*> xs -> [(2*)1, (3+)2] -> [2, 5]
```
По дефолту в Хаскеле Аппликативный Функтор Списка реализован первым способом. Тогда оператор `<*>` в этом случае должен реализовывать модель "каждый с каждым".

```haskell
instance Applicative [] where
    pure x = [x]
    gs <*> xs = [ g x | g <- gs, x <- xs ]      -- list comprehension
```
Но и второе применение нам иногда тоже хочется иметь, однако мы не можем написать два представителя для одного типа, поэтому мы сделаем `newtype`.
```haskell
newtype ZipList a = ZipList { getZipList :: [a] }

instance Functor ZipList where          -- функтор фактически такой же
    fmap f (ZipList xs) = ZipList (map f xs)

instance Applicative ZipList where      -- а вот здесь мы теперь реализуем другую логику
    pure x = ???                    -- тот, который был раньше, нам не подойдёт, тут должен 
                                    -- быть бесконечный список из иксов
    ZipList gs <*> ZipList xs = ZipList (zipWith ($) gs xs)
```

### Для пары

```haskell
instance Monoid s => Applicative ((,) s) where
    pure x            = (mempty, x)
    (u, f) <*> (v, x) = (u <> v, f x)
```