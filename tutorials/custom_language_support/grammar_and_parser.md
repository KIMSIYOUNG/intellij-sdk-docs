---
title: 3. Grammar and Parser
---

### 3.1. Define a token type

Create a file in the `com.simpleplugin.psi` package.

```java
{% include /code_samples/simple_language_plugin/src/main/java/com/intellij/sdk/language/psi/SimpleTokenType.java %}
```

### 3.2. Define an element type

Create a file in the `com.simpleplugin.psi` package.

```java
{% include /code_samples/simple_language_plugin/src/main/java/com/intellij/sdk/language/psi/SimpleElementType.java %}
```

### 3.3. Define grammar

Define a grammar for the properties language with */com/simpleplugin/Simple.bnf* file.

```java
{
  parserClass="com.simpleplugin.parser.SimpleParser"

  extends="com.intellij.extapi.psi.ASTWrapperPsiElement"

  psiClassPrefix="Simple"
  psiImplClassSuffix="Impl"
  psiPackage="com.simpleplugin.psi"
  psiImplPackage="com.simpleplugin.psi.impl"

  elementTypeHolderClass="com.simpleplugin.psi.SimpleTypes"
  elementTypeClass="com.simpleplugin.psi.SimpleElementType"
  tokenTypeClass="com.simpleplugin.psi.SimpleTokenType"
}

simpleFile ::= item_*

private item_ ::= (property|COMMENT|CRLF)

property ::= (KEY? SEPARATOR VALUE?) | KEY
```

As you see a properties file can contain properties, comments and line breaks.

The grammar defines how flexible the support for a language can be.
We specified that a property may have or may not have key and value.
This lets the IDE still recognise incorrectly defined properties and provide corresponding code analysis and quick-fixes.

Note that the `SimpleTypes` class in the `elementTypeHolderClass` field above specifies the name of a class that gets generated from the grammar, it doesn't exist at this point.

### 3.4. Generate a parser

Now when the grammar is defined we can generate a parser with PSI classes via *Generate Parser Code* from the context menu on *Simple.bnf* file.
This will generate a parser and PSI elements in *gen* folder.
Mark this folder as *Generated Sources Root* and make sure everything is compiled without errors.

![Parser](img/generated_parser.png)
