llvm-hs-typed
=============

[![Build Status](https://travis-ci.org/llvm-hs/llvm-hs-typed.svg?branch=master)](https://travis-ci.org/llvm-hs/llvm-hs-typed)

An experimental branch of
[llvm-hs-pure](https://hackage.haskell.org/package/llvm-hs-pure) AST that
enforces the semantics of correct AST construction using the Haskell type system
to prevent malformed ASTs.

Usage
-----

### Typed AST

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE PolyKinds #-}
{-# LANGUAGE TypeFamilies #-}
{-# LANGUAGE TypeOperators #-}
{-# LANGUAGE ExplicitForAll #-}
{-# LANGUAGE TypeApplications #-}
{-# LANGUAGE FlexibleInstances #-}
{-# LANGUAGE OverloadedStrings #-}
{-# LANGUAGE ScopedTypeVariables #-}
{-# LANGUAGE AllowAmbiguousTypes #-}
{-# LANGUAGE UndecidableInstances #-}
{-# LANGUAGE MultiParamTypeClasses #-}

module Example where

-- AST
import GHC.TypeLits
import LLVM.Prelude
import LLVM.AST.Tagged
import LLVM.AST.Constant
import LLVM.AST.Tagged.Global
import LLVM.AST.Tagged.Constant
import LLVM.AST.Tagged.Tag
import LLVM.AST.TypeLevel.Type

import qualified LLVM.AST as AST
import qualified LLVM.AST.Global as AST

c0 :: Constant ::: IntegerType' 32
c0 = int 42

named :: forall (t :: Type'). ShortByteString -> Name ::: t
named s = assertLLVMType $ AST.Name s

type ArgTys = [(IntegerType' 32), (IntegerType' 32)]
type RetTy = IntegerType' 32

defAdd :: Global
defAdd = function nm (params, False) [body, body]
  where
    nm :: Name ::: (PointerType' (FunctionType' (IntegerType' 32) ArgTys) ('AddrSpace' 0))
    nm = named "add"

    -- Types of subexpression are inferred from toplevel LLVM function signature

    {-p1 :: Parameter ::: (IntegerType' 32)-}
    p1 = parameter (named "a") []

    {-p2 :: Parameter ::: (IntegerType' 32)-}
    p2 = parameter (named "b") []

    {-body :: BasicBlock ::: IntegerType' 32-}
    body = basicBlock "entry" [] (ret (constantOperand c0) [])

    {-params :: Parameter :::* ArgTys-}
    params = p1 :* p2 :* tnil

module_ :: AST.Module
module_ = defaultModule
  { moduleName = "basic"
  , moduleDefinitions = [GlobalDefinition defAdd]
  }
```

### Typed IRBuilder

```haskell
{-# LANGUAGE DataKinds #-}
{-# LANGUAGE PolyKinds #-}
{-# LANGUAGE RecursiveDo #-}
{-# LANGUAGE TypeOperators #-}
{-# LANGUAGE OverloadedStrings #-}

module Example2 where

import GHC.TypeLits
import LLVM.Prelude
import LLVM.AST.Constant
import LLVM.AST.Tagged.Global
import LLVM.AST.Tagged.Tag
import LLVM.AST.TypeLevel.Type
import qualified LLVM.AST as AST
import qualified LLVM.AST.Type as AST
import qualified LLVM.AST.Global as AST
import qualified LLVM.AST.Tagged as AST

import LLVM.AST.Tagged.IRBuilder as TBuilder
import qualified LLVM.IRBuilder as Builder

import Data.Coerce

simple :: AST.Module
simple = Builder.buildModule "exampleModule" $ do
    func
  where
  func :: Builder.ModuleBuilder (AST.Operand ::: IntegerType' 32)
  func =
    TBuilder.function "add" [(AST.i32, "a"), (AST.i32, "b")] $ \[a, b] -> do
      entry <- block `named` "entry"; do
        c <- add (coerce a) (coerce b)
        ret c
```

License
-------

Copyright (c) 2017, Joachim Breitner
