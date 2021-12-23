# 28Transformers

### Знакомство (пример)

```haskell
stInteger :: State Integer Integer
stInteger = do modify (+1)
							 a <- get
							 return a

stString :: State String String
stString = do modify (++"1")
							b <- get
							return b
```

```haskell
GHCi> evalState stInteger 0
1
GHCi> evalState stString "0"
"01"
```

Здесь у нас две похожих монады для вычислений, мы хотим эти вычисления объединить в одно монадическое вычисление.

Как вариант очень просто сделать `State`от пары `Integer, String`. С точки зрения трансформеров можно сказать, что внутри монада, которая `State String`, а сверху накручиваем дополнительное `State Integer`

*далее* *описание к коду ниже*

Подключим модуль `Control.Monad.State`, получим доступ к трансформеру монады `State` — `StateT` (Имя трансформера = имя монады + T).

Трансформер, в отличие от монад, получает дополнительный параметр: та монада, на которую мы что-то накручиваем. Внутренняя монада тоже трансформер, но чтобы не было рекурсии, заполняем ее “кочерыжкой” `Identity` (Трансформеры монад как капуста...). Накручиваем туда `State String`

Во внешней монаде хранится `Integer`, во внутренней `String` По значению будет возвращаться пара.

```haskell
stComb :: StateT Integer -- внешняя монада (со второй строкой вместе)
								(StateT String Identity) -- внутренняя монада
								(Integer, String)

stComb = do modify (+1) -- к внешней монаде StateT Integer ..
						lift $ modify (++"1") -- к внутренней монаде State String
						a <- get -- к внешней
						b <- lift $ get -- к внутренней
						return (a,b)
```

`lift` нужен, чтобы поднять вычисление из внутренней монады на внешнюю. Сколько `lift` написал - настолько глубока твоя монада.

Теперь запуск

```haskell
GHCi> runIdentity $ evalStateT (evalStateT stComb 0) "0"
(1,"01")
```

Внешний `evalStateT` вынимает из накрутки внутреннюю монаду, которая тоже `State`.

Запустили в `evalStateT` вычисление внешней монады `State Integer` с помощью`stComb` от `Integer` 0, он вернул нам внутреннюю монаду, поэтому мы снова запускаем `evalStateT` уже для внутренней монады `StateT String` и инициализируем строкой “0”, Но внутри лежала еще монада `Identity`, на которой можем просто запустить `runIdentity`

Можно помимо `Identity` использовать `IO` со специализированной `liftIO`

### Определение монады

**Трансформер монад — конструктор типа, который принимает монаду в качестве параметра и возвращает монаду как результат.**

**Требования:**

1. Кайнд монады `* -> *`, тогда у трансформера должен быть кайнд “из монады в монаду” `( * -> * ) -> * -> *`

— у `StateT` немного другой кайнд, в начало нужно еще добавить `*->`

1. Для любой монады `m`, аппликация `t m` должна быть монадой, то есть её `return` и `(>>=)` должны удовлетворять законам монад
2. Нужен `lift :: m a -> t m a`, “поднимающий” значение из трансформируемой монады в трансформированную.

В библиотеке `transformers` функция `lift` всегда вызывается вручную (если надо спуститься на определенную монаду, то надо вызвать несколько раз `lift`), в библиотеке `mtl` — только для неоднозначных ситуаций. `mtl` говорит о том, что мы научим внешнюю монаду свойствам внутренней, поэтому явно вызывать `lift` не нужно.

### Рецепт приготовления трансформера для MyMonad

**1 шаг.**

Пусть у нас есть какая-то монада `MyMonad`. Объявим трансформер

```haskell
newtype MyMonadT m a = MyMonadT { runMyMonadT :: m (MyMonad a) }
```

Когда мы запускаем `runMyMonadT`, нам нужно вынуть именно `m` - монаду, на которую мы все накручиваем. Где применить `MyMonad`? На самом деле записать можно только так: на параметр `a` навешать `MyMonad`, больше некуда.

**2 шаг.** Для любой монады `m`, аппликация `t m` должна быть монадой. 

Делаем аппликацию нашего трансформера к монаде `MyMonadT m` представителем `Monad`.

```haskell
instance Monad m => Monad (MyMonadT m) where
return x = ...
mx >>= k = ...
```

**2a шаг.** Функцию `fail` нужно реализовать обязательно. 

- Первый случай: монада заточена под обработку ошибок. Напишем обработчик

```haskell
instance Monad m => MonadFail (MyMonadT m) where
fail s = ...
```

- Второй случай: монада может не уметь обрабатывать ошибки, и раньше мы могли в этом случае не писать `fail`, но в трансформерах не так: внутренняя монада может уметь обрабатывать ошибку, тогда нужно поднять обработчик ошибок (то есть переадресовали обработку ошибок внутренней монаде)

```haskell
instance MonadFail m => MonadFail (MyMonadT m) where
fail = lift . fail
```

- NB: Если GHC < 8.6 fail нужно реализовывать в представителе класса типов Monad.

**3 шаг.** Реализовать операцию `lift`

```haskell
class MonadTrans t where -- метод класса типов MonadTrans
lift :: Monad m => m a -> t m a -- сигнатура как у трансформера
```

Напишем реализацию:

```haskell
instance MonadTrans MyMonadT where
lift mx = ...
```

Нужно делать это аккуратно, чтобы никаких эффектов нигде не породить. Для этого существуют специальные **законы для класса типов** `MonadTrans`

- **Right Zero** — Правый ноль

```haskell
lift . return ≡ return
-- справа налево -- эталонная реализация return в трансформере
```

- **Left Distribution** — Левая дистрибутивность

```haskell
lift (m >>= k) ≡ lift m >>= (lift . k)
```

### Трансформер для `Maybe`

Этот трансформер накручивает на имеющуюся монаду такую функциональность: возможность элементарной ошибки (`Nothing`)

```haskell
newtype MaybeT m a = MaybeT {runMaybeT :: m (Maybe a)}
MaybeT :: m (Maybe a) -> MaybeT m a
runMaybeT :: MaybeT m a -> m (Maybe a)

instance MonadTrans MaybeT where
	lift :: m a -> MaybeT m a
	lift = MaybeT . fmap Just 
-- fmap повесил Maybe на параметр а
```

Пояснение работы `lift` при поднятии `get` из `State` в `mtl`:

```haskell
GHCi> :t get
get :: MonadState s m => m s
GHCi> :t lift get
lift get :: (MonadState s m, MonadTrans t) => t m s
```

Написали `lift` (шаг 3 рецепта), теперь халявно получили `return` (шаг 2).

Написана версия, которая отражает, что происходит. Халявная версия указана в комментарии рядом. 

```haskell
newtype MaybeT m a = MaybeT {runMaybeT :: m (Maybe a)}

instance Monad m => Monad (MaybeT m) where
	return :: a -> MaybeT m a
	return = MaybeT . fmap Just . return -- lift . return

	(>>=) :: MaybeT m a -> (a -> MaybeT m b) -> MaybeT m b
	mx >>= k = MaybeT $ do -- inner monad do
		v <- runMaybeT mx
		case v of
			Nothing -> return Nothing
			Just y -> runMaybeT (k y)
```

`do` запускает вычисление в монаде `m`. Далее вынимаем в `v` значение из `runMaybeT mx`, то есть из монады `m`, то есть это значение типа `Maybe a`.  Далее просто парсим случаи.

Остался шаг 2а:

```haskell
newtype MaybeT m a = MaybeT {runMaybeT :: m (Maybe a)}

instance Monad m => MonadFail (MaybeT m) where
	fail :: String -> MaybeT m a
	fail _ = MaybeT $ return Nothing
```

*Пример использования (написаны 3 штуки)*

```haskell
-- 1)
mbSt :: MaybeT (StateT Integer Identity) Integer
mbSt = do
	lift $ modify (+1)
	a <- lift get
	True <- return $ a >= 3
	return a

{-
GHCi> runIdentity $ evalStateT (runMaybeT mbSt) 0
Nothing
GHCi> runIdentity $ evalStateT (runMaybeT mbSt) 2
Just 3
-}

-- 2)
-- Чтобы заработал guard нужно сделать MaybeT m представителем Alternative.
-- (Начиная с GHC 7.10 guard определен через Alternative, а не MonadPlus.)

instance Monad m => Alternative (MaybeT m) where
  empty = MaybeT $ return Nothing
  x <|> y = MaybeT $ do
    v <- runMaybeT x
    case v of
      Nothing -> runMaybeT y
      Just _  -> return v

-- Это необязательно, но может пригодиться для msum, mfilter
instance Monad m => MonadPlus (MaybeT m)
      
mbSt' :: MaybeT (State Integer) Integer
mbSt' = do lift $ modify (+1)
           a <- lift get
           guard $ a >= 3 -- !!
           return a
{-
GHCi> runIdentity $ evalStateT (runMaybeT mbSt') 0
Nothing
GHCi> runIdentity $ evalStateT (runMaybeT mbSt') 2
Just 3
-}

-- 3)
-- Для любой пары монад можно избавиться от подъёма 
-- стандартных операций вложенной монады

{-
class Monad m => MonadState s m | m -> s where
    get :: m s
    get = state (\s -> (s, s))
    
    put :: s -> m ()
    put s = state (\_ -> ((), s))
    
    state :: (s -> (a, s)) -> m a
    state f = do
      s <- get
      let ~(a, s') = f s
      put s'
      return a

modify :: MonadState s m => (s -> s) -> m ()
modify f = state (\s -> ((), f s))

gets :: MonadState s m => (s -> a) -> m a
gets f = do
    s <- get
    return (f s)

-}

-- требуются расширения FlexibleInstances, UndecidableInstances
instance MonadState s m => MonadState s (MaybeT m) where
  get = lift get
  put = lift . put

mbSt'' :: MaybeT (State Integer) Integer
mbSt'' = do 
  modify (+1) -- без lift
  a <- get -- без lift
  guard $ a >= 3
  return a
{-
GHCi> runIdentity $ evalStateT (runMaybeT mbSt'') 0
Nothing
GHCi> runIdentity $ evalStateT (runMaybeT mbSt'') 2
Just 3
-}
```

Про библиотеки.

![Снимок экрана от 2021-12-24 00-13-08.png](28Transformers%20cf66598d282841cfac4b9e54c305203b/%D0%A1%D0%BD%D0%B8%D0%BC%D0%BE%D0%BA_%D1%8D%D0%BA%D1%80%D0%B0%D0%BD%D0%B0_%D0%BE%D1%82_2021-12-24_00-13-08.png)

### Что во что вкладывать?

Если нам нужна функциональность `Except` и `State`, то есть наша монада должна быть представителем `MonadError` и `MonadState` то мы должны применить трансформер `ErrorT` к монаде  `State` или же трансформер `StateT` к `Except`? Решение зависит от того, какой в точности семантики мы ожидаем от комбинированной монады.

![Снимок экрана от 2021-12-24 00-17-59.png](28Transformers%20cf66598d282841cfac4b9e54c305203b/%D0%A1%D0%BD%D0%B8%D0%BC%D0%BE%D0%BA_%D1%8D%D0%BA%D1%80%D0%B0%D0%BD%D0%B0_%D0%BE%D1%82_2021-12-24_00-17-59.png)