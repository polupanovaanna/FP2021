# 17 Аппликативные парсеры

- Парсер - программа, принимающая на ввход строку и возвращающая некоторую структуру данных, если строка удовлетворяет заданной грамматике или сообщение об ошибке в противоположном случае.

Простейшие примеры:
```haskell
type Parser a = String -> a  --простейший и неудобный
type Parser a = String -> (String, a)  --храним неразобранный остатотк строки
--эти два парсера ещё обрабатывают ошибку
type Parser a = Maybe (String -> a)
type Parser a = Either (String -> (String, a))
--для неоднозначных грамматик
type Parser a = String -> [(String, a)] 
```

Далее в качестве базового парсера будет использоваться 3-я версия:
```haskell
type Parser a = Maybe (String -> a)
```
Мы можем улучшить её абстрагировавшись по типу входного потока
```haskell
newtype Parser tok a = Parser { runParser :: [tok] -> Maybe ([tok], a) }
```
Стандартные функции для конструирования парсеров:
```haskell
satisfy :: (tok -> Bool) -> Parser tok tok
satisfy pr = Parser f where
    f (c:cs) | pr c = Just (cs, c)
    f _             = Nothing

lower :: Parser Char CHar
lower = satisfy isLower

char :: Char -> Parser Char Char
char c = satisfy (== c)

digit :: Parser Char Int
digit = satisfy isDigit   --возвращает Char
```
Мы могли бы сделать 
```haskell
digit = digitTiInt <$> satisfy isDigit   
```
тогда бы он возвращал нам `Int`. Но для этого нам нужно сделать пареср представителем функтора. Для этого нам надо написать ему `fmap`
```haskell
instance Functor (Parser tok) where
    fmap :: (a -> b) -> Parser tok a -> Parser tok b
    fmap g (Parser p) = Parser f where
        f xs = case p xs of
            Just (cs, c) -> Just (cs, g c)
            Nothing      -> Nothing
```
Однако мы можем записать это объявление короче, так как если посмотреть на наше определение `Parser`, то это просто композиция трёх функторов: `(->) [tok]`, `Maybe` и `(,) [tok]`. Более короткий варант:
```haskell
instance Functor (Parser tok) where
    fmap :: (a -> b) -> Parser tok a -> Parser tok b
    fmap g = Parser . (fmap . fmap . fmap) g . runParser
```
## Собственно аппликативные парсеры
- Общий смысл того, что мы хотим сделать - мы хотим научиться делать из маленьких парсеров-блоков большие парсеры с помощью оператора `ap` `(<*>)` у аппликативного функтора, то есть на строке запускается парсер, он что-то возвращает, затем на остатке строки запускается следующий парсер, он что-то возвращает и так далее. Для того, чтобы это сделать, нам надо наш парсер сделать представителем `Applicative`, написав ему `pure` и `ap`. По логике `pure` должен возвращать парсер, который не поглащает ничего из входной строки и возвращает всегда одно и то же заданное значение.
```haskell
instance Applicative (Parser tok) where
    pure :: a -> Parser tok a
    pure x = Parser $ \s -> Just (s, x) 
```
То есть работает он например так:
```ghci
> runParser (pure 42) "abcd"
Just ("abcd",42)
```
Теперь нужен `ap`

```haskell
    (<*>) :: Parser tok (a -> b) -> Parser tok a -> Parser tok b
    Parser u <*> Parser v = Parser f where
        f xs = case u xs of
            Nothing       -> Nothing
            Just (xs', g) -> case v xs' of
                Nothing        -> Notjing
                Just (xs'', x) -> Just (xs'', g x)
```
Если у нас хотя бы один парсинг закончился неудачей, то весь парсер заанчивается с неудачей.

Вот весь итоговый код аппликативного парсера с лекции:

```haskell
newtype Parser tok a = Parser { runParser :: [tok] -> Maybe ([tok], a) }

instance Functor (Parser tok) where
    fmap :: (a -> b) -> Parser tok a -> Parser tok b
    fmap g = Parser . (fmap . fmap . fmap) g . runParser

instance Applicative (Parser tok) where
    pure :: a -> Parser tok a
    pure x = Parser $ \s -> Just (s, x)

    (<*>) :: Parser tok (a -> b) -> Parser tok a -> Parser tok b
    Parser u <*> Parser v = Parser f where
        f xs = case u xs of
            Nothing       -> Nothing
            Just (xs', g) -> case v xs' of
                Nothing        -> Notjing
                Just (xs'', x) -> Just (xs'', g x)
```
Аппликативный парсер из дз, который допускает разбор неоднозначных грамматик (надеюсь вы помните, что это такое из фя):

```haskell
newtype Parser a = Parser { apply :: String -> [(a, String)] }

instance Functor Parser where
  fmap f x = Parser $ \s -> fmap (\p -> ((f (fst p)), (snd p))) ((apply x) s)

instance Applicative Parser where
  pure x = Parser (\s -> [(x, s)])
  (<*>) (Parser x) (Parser y) = Parser (\s -> [((g a), b) | (g, b1) <- x s
                                                          , (a, b) <- y b1])
                                                          --list comprehention

instance Alternative Parser where
  empty = Parser (\_ -> [])
  (<|>) (Parser a) (Parser b) = Parser (\s -> (++) (a s) (b s))
```
Ссылка на упражнение, где вы должны были его сделать: https://stepik.org/lesson/590408/step/5?unit=585345

По утверждению Васи аппликативные парсеры одна из немногих полезных идей, для которой используют Хаскель


