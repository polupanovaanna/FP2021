# Основные сигнатуры и законы (Денис Шестаков)

```
foldr :: (a -> b -> b) -> b -> [a] -> b
foldr f ini [] = ini
foldr f ini (x:xs) = x `f` (foldr f ini xs)

foldl :: (b -> a -> b) -> b -> [a] -> b
foldl f ini [] = ini
foldl f ini (x:xs) = foldl f (ini `f` x) xs

scanl :: (b -> a -> b) -> b -> [a] -> [b]
scanl _ z [] = [z]
scanl (#) z (x:xs) = z : scanl (#) (z # x) xs

scanr :: (a -> b -> b) -> b -> [a] -> [b]
scanr _ z [] = [z]
scanr f z (x:xs) = f x q : qs
  where qs@(q:_) = scanr f z xs

unfoldr :: (b -> Maybe (a, b)) -> b -> [a]
unfoldr g ini
  | Nothing <- next = []
  | Just (a,b) <- next = a : unfoldr g b
  where next = g ini

  head (scanr f z xs) ≡ foldr f z xs
  last (scanl f z xs) ≡ foldl f z xs

infixr 6 <>
class Semigroup a where
  (<>) :: a -> a -> a
  sconcat :: NonEmpty a -> a
  sconcat (a :| as) = go a as where
  go b (c:cs) = b <> go c cs
  go b [] = b
  stimes :: Integral b => b -> a -> a
  stimes = stimesDefault

infixr 5 :|
data NonEmpty a = a :| [a]

  1. (x <> y) <> z ≡ x <> (y <> z)

class Semigroup a => Monoid a where
  mempty :: a
  mappend :: a -> a -> a
  mappend = (<>)
  mconcat :: [a] -> a
  mconcat = foldr mappend mempty
  
  1. mempty <> x ≡ x
  2. x <> mempty ≡ x
  3. (x `mappend` y) `mappend` z ≡ x `mappend` (y `mappend` z)

class Foldable t where
  fold :: Monoid m => t m -> m
  fold = foldMap id
  
  foldMap :: Monoid m => (a -> m) -> t a -> m
  foldMap f = foldr (mappend . f) mempty
  
  foldr, foldr' :: (a -> b -> b) -> b -> t a -> b
  foldr f z t = appEndo (foldMap (Endo . f) t) z
  
  foldl, foldl' :: (a -> b -> a) -> a -> t b -> a
  foldl f z t = appEndo (getDual (foldMap (Dual . Endo . flip f) t)) z
  
  foldr1, foldl1 :: (a -> a -> a) -> t a -> a

  1. foldr f z t ≡ appEndo (foldMap (Endo . f) t ) z
  2. foldl f z t ≡ appEndo (getDual (foldMap (Dual . Endo . flip f) t)) z
  3. fold ≡ foldMap id
  4. length ≡ getSum . foldMap (Sum . const 1)
  5. sum ≡ getSum . foldMap Sum
  6. product ≡ getProduct . foldMap Product
  7. minimum ≡ getMin . foldMap Min
  8. maximum ≡ getMax . foldMap Max
  9. foldr f z ≡ foldr f z . toList
  10. foldl f z ≡ foldl f z . toList
  
  1. foldMap f ≡ fold . fmap f
  2. foldMap f . fmap g ≡ foldMap (f . g)

concat :: Foldable t => t [a] -> [a]
concatMap :: Foldable t => (a -> [b]) -> t a -> [b]
and,or :: Foldable t => t Bool -> Bool
any,all :: Foldable t => (a -> Bool) -> t a -> Bool
maximumBy,minimumBy :: Foldable t => (a -> a -> Ordering) -> t a -> a
notElem :: (Foldable t, Eq a) => a -> t a -> Bool
find :: Foldable t => (a -> Bool) -> t a -> Maybe a

class Functor f where
  fmap :: (a -> b) -> (f a -> f b)

infixl 4 <$, <$>, $>
class Functor f where
  fmap :: (a -> b) -> f a -> f b
  (<$) :: a -> f b -> f a
  (<$) = fmap . const

(<$>) :: Functor f => (a -> b) -> f a -> f b
(<$>) = fmap
($>) :: Functor f => f a -> b -> f b
($>) = flip (<$)

  1. fmap id ≡ id
  2. fmap (f . g) ≡ fmap f . fmap g

  1. fmap g . pure ≡ pure . g

infixl 4 <*>, *>, <*, <**>
class Functor f => Applicative f where
  {-# MINIMAL pure, ((<*>) | liftA2) #-}
  pure :: a -> f a
  (<*>) :: f (a -> b) -> f a -> f b
  (<*>) = liftA2 id
  liftA2 :: (a -> b -> c) -> f a -> f b -> f c
  liftA2 g a b = g <$> a <*> b
  (*>) :: f a -> f b -> f b
  a1 *> a2 = (id <$ a1) <*> a2
  (<*) :: f a -> f b -> f a
  (<*) = liftA2 const

infixl 4 <*>

  1. g <$> xs ≡ pure g <*> xs
  2. pure id <*> v ≡ v  (Identity)
  3. pure g <*> pure x ≡ pure (g x) (Homomorphism)
  4. u <*> pure x ≡ pure ($ x) <*> u (Interchange)
  5. pure (.) <*> u <*> v <*> x ≡ u <*> (v <*> x) (Composition)

newtype Parser tok a = Parser { runParser :: [tok] -> Maybe ([tok],a) }

satisfy :: (tok -> Bool) -> Parser tok tok
satisfy pr = Parser f where
  f (c:cs) | pr c = Just (cs,c)
  f _ = Nothing

Easy:
instance Functor (Parser tok) where
  fmap :: (a -> b) -> Parser tok a -> Parser tok b
  fmap g (Parser p) = Parser f where
    f xs = case p xs of
      Just (cs, c) -> Just (cs, g c)
      Nothing -> Nothing

Average:
instance Functor (Parser tok) where
  fmap :: (a -> b) -> Parser tok a -> Parser tok b
  fmap g = Parser . (fmap . fmap . fmap) g . runParser

Hard:
instance Functor (Parser tok) where
  fmap = coerce (fmap . fmap . fmap :: ...)

instance Applicative (Parser tok) where
  pure :: a -> Parser tok a
  pure x = Parser $ \s -> Just (s, x)
  
  (<*>) :: Parser tok (a -> b) -> Parser tok a -> Parser tok b
  Parser u <*> Parser v = Parser f where
    f xs = case u xs of
      Nothing -> Nothing
      Just (xs', g) -> case v xs' of
        Nothing -> Nothing
        Just (xs'', x) -> Just (xs'', g x)

infixl 3 <|>

class Applicative f => Alternative f where
  empty :: f a
  (<|>) :: f a -> f a -> f a
  some, many :: f a -> f [a]
  some v = (:) <$> v <*> many v   -- One or more
  many v = some v <|> pure []     -- Zero or more

optional :: Alternative f => f a -> f (Maybe a)
optional v = Just <$> v <|> pure Nothing

class (Functor t, Foldable t) => Traversable t where
  sequenceA :: Applicative f => t (f a) -> f (t a)
  sequenceA = traverse id
  traverse :: Applicative f => (a -> f b) -> t a -> f (t b)
  traverse g = sequenceA . fmap g

  1. traverse Identity ≡ Identity (identity)
  2. traverse (Compose . fmap f2 . f1) ≡ Compose . fmap (traverse f2) . traverse f1 (composition)
    t a -> Compose g2 g1 (t с)
  3. h . traverse f ≡ traverse (h . f) (naturality)
    h :: (Applicative f, Applicative g) => f b -> g b - homomorphism
	  (1) h (pure x) = pure x;
      (2) h (x <*> y) = h x <*> h y.

infixl 1 >>, >>=

class Applicative m => Monad m where
  (>>=) :: m a -> (a -> m b) -> m b -- произносят bind
  (>>) :: m a -> m b -> m b
  m1 >> m2 = m1 >>= \_ -> m2
  return :: a -> m a
  return = pure

class Pointed m => Monad m where
  join :: m (m a) -> m a

(<=<) :: (b -> m c) -> (a -> m b) -> (a -> m c)

  1. return a >>= k ≡ k a
  2. m >>= return ≡ m
  3. m >>= k >>= k' ≡ m >>= \x -> k x >>= k'

class Monad m => MonadFail m where
  fail :: String -> m a

  1. fail s >>= k ≡ fail s

newtype Reader r a = Reader { runReader :: r -> a }
reader :: (r -> a) -> Reader r a
runReader :: Reader r a -> r -> a

ask :: Reader r r
asks :: (r -> a) -> Reader r a
local :: (r -> r) -> Reader r a -> Reader r a

newtype Writer w a = Writer { runWriter :: (a, w) }
writer :: (a, w) -> Writer w a
runWriter :: Writer w a -> (a, w)
execWriter :: Writer w a -> w

tell :: Monoid w => w -> Writer w ()
listen :: Monoid w => Writer w a -> Writer w (a, w)
listens :: Monoid w => (w -> b) -> Writer w a -> Writer w (a, b)
censor :: Monoid w => (w -> w) -> Writer w a -> Writer w a

newtype State s a = State { runState :: s -> (a,s) }
state :: (s -> (a,s)) -> State s a
runState :: State s a -> s -> (a,s)
execState :: State s a -> s -> s
evalState :: State s a -> s -> a

get :: State s s
get = state $ \s -> (s,s)
put :: s -> State s ()
put s = state $ \_ -> ((),s)
modify :: (s -> s) -> State s ()
modify f = do s <- get
              put (f s)
gets :: (s -> a) -> State s a
gets f = do s <- get
            return (f s)

replicateM :: Applicative m => Int -> m a -> m [a]
replicateM n = sequenceA . replicate n
mapM :: (Traversable t, Monad m) => (a -> m b) -> t a -> m (t b)
mapM f = sequence . fmap f -- traverse

newtype IO a = IO (SRW -> (SRW, a))
unIO :: IO a -> SRW -> (SRW, a)
unIO (IO f) = f
getChar :: IO Char
getLine :: IO String
getContents :: IO String
putChar :: Char -> IO ()
putStr, putStrLn :: String -> IO ()
print :: Show a => a -> IO ()
interact :: (String -> String) -> IO ()

class (Alternative m, Monad m) => MonadPlus m where
  mzero :: m a
  mzero = empty
  mplus :: m a -> m a -> m a
  mplus = (<|>)

  1. mzero >>= k ≡ mzero (Left and Right Zero)
  2. v >> mzero ≡ mzero
  3. (a `mplus` b) >>= k ≡ (a >>= k) `mplus` (b >>= k) (Left Distribution)
  4. return a `mplus` b ≡ return a (Left Catch law)

guard :: Alternative f => Bool -> f ()
asum :: (Foldable t, Alternative f) => t (f a) -> f a
asum = foldr (<|>) empty
msum :: (Foldable t, MonadPlus m) => t (m a) -> m a
msum = asum -- foldr mplus mzero
mfilter :: MonadPlus m => (a -> Bool) -> m a -> m a
mfilter p ma = do
  a <- ma
  if p a
  then return a
  else mzero

instance Monoid e => MonadPlus (Except e) where
  mzero = except $ Left mempty
  x `mplus` y = except $ let alt = runExcept y in
    case runExcept x of
      Left e -> either (Left . mappend e) Right alt
      r -> r

newtype Except e a = Except { runExcept :: Either e a }
except :: Either e a -> Except e a
except = Except
instance Monad (Except e) where
  return :: a -> Except e a
  return a = except $ Right a
  (>>=) :: Except e a -> (a -> Except e b) -> Except e b
  m >>= k = case runExcept m of
    Left e -> except $ Left e
    Right x -> k x

throwE :: e -> Except e a
throwE = except . Left
catchE :: Except e a -> (e -> Except e' a) -> Except e' a
m `catchE` h = case runExcept m of
  Left l -> h l
  Right r -> except $ Right r

throwError :: e -> Except e a
catchError :: Except e a -> (e -> Except e' a) -> Except e' a

class Monad m => MonadReader r m | m -> r where
  ask :: m r
  local :: (r -> r) -> m a -> m a
class (Monoid w, Monad m) => MonadWriter w m | m -> w where
  tell :: w -> m ()
  listen :: m a -> m (a, w)
class Monad m => MonadState s m | m -> s where
  get :: m s
  put :: s -> m ()
class Monad m => MonadError e m | m -> e where
  throwError :: e -> m a
  catchError :: m a -> (e -> m a) -> m a

instance Monad m => Monad (MyMonadT m) where
  return x = ...
  mx >>= k = ...
instance Monad m => MonadFail (MyMonadT m) where
  fail s = ...
instance MonadFail m => MonadFail (MyMonadT m) where
  fail = lift . fail
class MonadTrans t where
  lift :: Monad m => m a -> t m a

  1. lift . return ≡ return (Right Zero)
  2. lift (m >>= k) ≡ lift m >>= (lift . k) (Left Distribution)

from :: a -> b
to :: b -> a

  1. to . from ≡ id
  2. from . to ≡ id 

newtype Fix f = In (f (Fix f))

type Algebra f a = f a -> a
cata :: Functor f => Algebra f a -> Fix f -> a
cata phi (In x) = phi $ fmap (cata phi) x

type Coalgebra f a = a -> f a
ana :: Functor f => Coalgebra f a -> a -> Fix f
ana psi x = In $ fmap (ana psi) (psi x)

hylo :: Functor f => Algebra f a -> Coalgebra f b -> b -> a
hylo phi psi = cata phi . ana psi
cata' phi = hylo phi out
ana' psi = hylo In psi

lens :: (s -> a) -> (s -> a -> s) -> Lens s a
--lens :: (s -> a) -> (s -> b -> t) -> Lens s t a b

  1. view l (set l v s) ≡ v
  2. set l (view l s) s ≡ s
  3. set l v' (set l v s) ≡ set l v' s

{-# LANGUAGE Rank2Types #-}
type Lens s a = forall f. Functor f => (a -> f a) -> s -> f s

-- (s -> a) -> (s -> a -> s) -> (a -> f a) -> s -> f s
lens :: (s -> a) -> (s -> a -> s) -> Lens s a
lens get set = \ret s -> fmap (set s) (ret $ get s)

newtype Const a x = Const { getConst :: a }
instance Functor (Const a) where
  fmap _ (Const v) = Const v

view :: Lens s a -> s -> a
view lns s = getConst (lns Const s)

newtype Identity a = Identity { runIdentity :: a }
instance Functor Identity where
  fmap f (Identity x) = Identity (f x)

over :: Lens s a -> (a -> a) -> s -> s
over lns fn s = runIdentity $ lns (Identity . fn) s

set :: Lens s a -> a -> s -> s
set lns a s = over lns (const a) s

preview :: Prism' s a -> s -> Maybe a
review :: Prism' s a -> a -> s
```