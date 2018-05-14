---
layout: post
title: 《代码整洁之道 Clean Code》学习日志（三）：动手实践
date: 2018-5-13 13:16:04
author: admin
comments: true
categories: [Java]
tags: [Clean Code]
---

这一篇会从一个实际例子出发，结合上一篇笔记中整洁代码的知识点，来看一看如何写出整洁的代码。另外需要注意这一篇中会有大段的代码。

<!-- more -->
---

学习资料主要参考： 《代码整洁之道》，作者：(美国)马丁(Robert C. Martin) 译者：韩磊

---
## 目录
{:.no_toc}

* 目录
{:toc}
---

# 命令行参数解析模块

第一个例子是，对一个命令行参数解析程序的案例研究。

这个例子主要是强调一个重点：逐步改进。

我们都遇到过解析命令行参数的情况。现在我们来写一个Args，它非常易于使用，只用简单地用输入参数和格式化字符串构造Args类，再向Args对象查询参数值即可。
```java
public static void main(String[] args) {
    try {
        Args arg = new Args("l,p#,d*", args);
        boolean logging = arg.getBoolean('l');
        int port = arg.getInt('p');
        String directory = arg.getString('d');
        executeApplication(logging, port, directory);
    } catch (ArgsException e) {
        System.out.printf("Argument error: %s\n", e.errorMessage());
    }
}
```

## 1. Args的实现

```java
package com.objectmentor.utilities.args;

import static com.objectmentor.utilities.args.ArgsException.ErrorCode.*;
import java.util.*;

public class Args {
    private Map<Character, ArgumentMarshaler> marshalers;
    private Set<Character> argsFound;
    private ListIterator<String> currentArgument;

    public Args(String schema, String[] args) throws ArgsException {
        marshalers = new HashMap<Character, ArgumentMarshaler>();
        argsFound = new HashSet<Character>();

        parseSchema(schema);
        parseArgumentStrings(Arrays.asList(args));
    }

    private void parseSchema(String schema) throws ArgsException {
        for (String element : schema.split(","))
            if (element.length() > 0)
                parseSchemaElement(element.trim());
    }

    private void parseSchemaElement(String element) throws ArgsException {
        char elementId = element.charAt(0);
        String elementTail = element.substring(1);
        validateSchemaElementId(elementId);
        if (elementTail.length() == 0)
            marshalers.put(elementId, new BooleanArgumentMarshaler());
        else if (elementTail.equals("*"))
            marshalers.put(elementId, new StringArgumentMarshaler());
        else if (elementTail.equals("#"))
            marshalers.put(elementId, new IntegerArgumentMarshaler());
        else if (elementTail.equals("##"))
            marshalers.put(elementId, new DoubleArgumentMarshaler());
        else if (elementTail.equals("[*]"))
            marshalers.put(elementId, new StringArrayArgumentMarshaler());
        else
            throw new ArgsException(INVALID_ARGUMENT_FORMAT, elementId, elementTail);
    }

    private void validateSchemaElementId(char elementId) throws ArgsException {
        if (!Character.isLetter(elementId))
            throw new ArgsException(INVALID_ARGUMENT_NAME, elementId, null);
    }

    private void parseArgumentStrings(List<String> argsList) throws ArgsException
    {
        for (currentArgument = argsList.listIterator(); currentArgument.hasNext();)
        {
            String argString = currentArgument.next();
            if (argString.startsWith("-")) {
                parseArgumentCharacters(argString.substring(1));
            } else {
                currentArgument.previous();
                break;
            }
        }
    }
    private void parseArgumentCharacters(String argChars) throws ArgsException {
        for (int i = 0; i < argChars.length(); i++)
            parseArgumentCharacter(argChars.charAt(i));
    }

    private void parseArgumentCharacter(char argChar) throws ArgsException {
        ArgumentMarshaler m = marshalers.get(argChar);
        if (m == null) {
            throw new ArgsException(UNEXPECTED_ARGUMENT, argChar, null);
        } else {
            argsFound.add(argChar);
            try {
                m.set(currentArgument);
            } catch (ArgsException e) {
                e.setErrorArgumentId(argChar);
                throw e;
            }
        }
    }

    public boolean has(char arg) {
        return argsFound.contains(arg);
    }

    public int nextArgument() {
        return currentArgument.nextIndex();
    }

    public boolean getBoolean(char arg) {
        return BooleanArgumentMarshaler.getValue(marshalers.get(arg));
    }

    public String getString(char arg) {
        return StringArgumentMarshaler.getValue(marshalers.get(arg));
    }

    public int getInt(char arg) {
        return IntegerArgumentMarshaler.getValue(marshalers.get(arg));
    }

    public double getDouble(char arg) {
        return DoubleArgumentMarshaler.getValue(marshalers.get(arg));
    }

    public String[] getStringArray(char arg) {
        return StringArrayArgumentMarshaler.getValue(marshalers.get(arg));
    }
}
```

Args里面有个ArgumentMarshaler接口，来看看它是如何定义，以及它的派生类是什么样子的。

```java
//ArgumentMarshaler.java
public interface ArgumentMarshaler {
    void set(Iterator<String> currentArgument) throws ArgsException;
}
//BooleanArgumentMarshaler.java
public class BooleanArgumentMarshaler implements ArgumentMarshaler {
    private boolean booleanValue = false;

    public void set(Iterator<String> currentArgument) throws ArgsException {
        booleanValue = true;
    }

    public static boolean getValue(ArgumentMarshaler am) {
        if (am != null && am instanceof BooleanArgumentMarshaler)
            return ((BooleanArgumentMarshaler) am).booleanValue;
        else
            return false;
    }
}
//StringArgumentMarshaler.java
import static com.objectmentor.utilities.args.ArgsException.ErrorCode.*;
public class StringArgumentMarshaler implements ArgumentMarshaler {
    private String stringValue = "";

    public void set(Iterator<String> currentArgument) throws ArgsException {
        try {
            stringValue = currentArgument.next();
        } catch (NoSuchElementException e) {
            throw new ArgsException(MISSING_STRING);
        }
    }

    public static String getValue(ArgumentMarshaler am) {
        if (am != null && am instanceof StringArgumentMarshaler)
            return ((StringArgumentMarshaler) am).stringValue;
        else
            return "";
    }
}
```

以及ArgsException的定义：

```java
import static com.objectmentor.utilities.args.ArgsException.ErrorCode.*;

public class ArgsException extends Exception {
    private char errorArgumentId = '\0';
    private String errorParameter = null;
    private ErrorCode errorCode = OK;

    public ArgsException() {}

    public ArgsException(String message) {super(message);}

    public ArgsException(ErrorCode errorCode) {
        this.errorCode = errorCode;
    }

    public ArgsException(ErrorCode errorCode, String errorParameter) {
        this.errorCode = errorCode;
        this.errorParameter = errorParameter;
    }

    public ArgsException(ErrorCode errorCode,
    char errorArgumentId, String errorParameter) {
        this.errorCode = errorCode;
        this.errorParameter = errorParameter;
        this.errorArgumentId = errorArgumentId;
    }

    public char getErrorArgumentId() {
        return errorArgumentId;
    }

    public void setErrorArgumentId(char errorArgumentId) {
        this.errorArgumentId = errorArgumentId;
    }

    public String getErrorParameter() {
        return errorParameter;
    }

    public void setErrorParameter(String errorParameter) {
        this.errorParameter = errorParameter;
    }

    public ErrorCode getErrorCode() {
        return errorCode;
    }

    public void setErrorCode(ErrorCode errorCode) {
        this.errorCode = errorCode;
    }

    public String errorMessage() {
        switch (errorCode) {
            case OK:
                return "TILT: Should not get here.";
            case UNEXPECTED_ARGUMENT:
                return String.format("Argument -%c unexpected.", errorArgumentId);
            case MISSING_STRING:
                return String.format("Could not find string parameter for -%c.",
                                errorArgumentId);
            case INVALID_INTEGER:
                return String.format("Argument -%c expects an integer but was '%s'.",
                                errorArgumentId, errorParameter);
            case MISSING_INTEGER:
                return String.format("Could not find integer parameter for -%c.",
                                errorArgumentId);
            case INVALID_DOUBLE:
                return String.format("Argument -%c expects a double but was '%s'.",
                                errorArgumentId, errorParameter);
            case MISSING_DOUBLE:
                return String.format("Could not find double parameter for -%c.",
                                errorArgumentId);
            case INVALID_ARGUMENT_NAME:
                return String.format("'%c' is not a valid argument name.",
                                errorArgumentId);
            case INVALID_ARGUMENT_FORMAT:
                return String.format("'%s' is not a valid argument format.",
                                errorParameter);
        }
        return "";
    }

    public enum ErrorCode {
        OK, INVALID_ARGUMENT_FORMAT, UNEXPECTED_ARGUMENT, INVALID_ARGUMENT_NAME,
        MISSING_STRING,
        MISSING_INTEGER, INVALID_INTEGER,
        MISSING_DOUBLE, INVALID_DOUBLE}
}
```

上面的这段程序，并不是一开始就写出来的。
> 要编写整洁的代码，必须先写肮脏代码，然后再清理它。

## 2. Args：草稿

它可以工作，但是它很烂。

```java
import java.text.ParseException;
import java.util.*;

public class Args {
    private String schema;
    private String[] args;
    private boolean valid = true;
    private Set<Character> unexpectedArguments = new TreeSet<Character>();
    private Map<Character, Boolean> booleanArgs =
    new HashMap<Character, Boolean>();
    private Map<Character, String> stringArgs = new HashMap<Character, String>();
    private Map<Character, Integer> intArgs = new HashMap<Character, Integer>();
    private Set<Character> argsFound = new HashSet<Character>();
    private int currentArgument;
    private char errorArgumentId = '\0';
    private String errorParameter = "TILT";
    private ErrorCode errorCode = ErrorCode.OK;

    private enum ErrorCode {
        OK, MISSING_STRING, MISSING_INTEGER, INVALID_INTEGER, UNEXPECTED_ARGUMENT}

    public Args(String schema, String[] args) throws ParseException {
        this.schema = schema;
        this.args = args;
        valid = parse();
    }

    private boolean parse() throws ParseException {
        if (schema.length() == 0 && args.length == 0)
            return true;
        parseSchema();
        try {
            parseArguments();
        } catch (ArgsException e) {
        }
        return valid;
    }

    private boolean parseSchema() throws ParseException {
        for (String element : schema.split(",")) {
            if (element.length() > 0) {
                String trimmedElement = element.trim();
                parseSchemaElement(trimmedElement);
            }
        }
        return true;
    }

    private void parseSchemaElement(String element) throws ParseException {
        char elementId = element.charAt(0);
        String elementTail = element.substring(1);
        validateSchemaElementId(elementId);
        if (isBooleanSchemaElement(elementTail))
            parseBooleanSchemaElement(elementId);
        else if (isStringSchemaElement(elementTail))
            parseStringSchemaElement(elementId);
        else if (isIntegerSchemaElement(elementTail)) {
            parseIntegerSchemaElement(elementId);
        } else {
            throw new ParseException(
                    String.format("Argument: %c has invalid format: %s.",
                    elementId, elementTail), 0);
        }
    }

    private void validateSchemaElementId(char elementId) throws ParseException {
        if (!Character.isLetter(elementId)) {
            throw new ParseException(
                    "Bad character:" + elementId + "in Args format: " + schema, 0);
        }
    }

    private void parseBooleanSchemaElement(char elementId) {
        booleanArgs.put(elementId, false);
    }

    private void parseIntegerSchemaElement(char elementId) {
        intArgs.put(elementId, 0);
    }

    private void parseStringSchemaElement(char elementId) {
        stringArgs.put(elementId, "");
    }

    private boolean isStringSchemaElement(String elementTail) {
        return elementTail.equals("*");
    }

    private boolean isBooleanSchemaElement(String elementTail) {
        return elementTail.length() == 0;
    }

    private boolean isIntegerSchemaElement(String elementTail) {
        return elementTail.equals("#");
    }

    private boolean parseArguments() throws ArgsException {
        for (currentArgument = 0; currentArgument < args.length; currentArgument++)
        {
            String arg = args[currentArgument];
            parseArgument(arg);
        }
        return true;
    }

    private void parseArgument(String arg) throws ArgsException {
        if (arg.startsWith("-"))
            parseElements(arg);
    }

    private void parseElements(String arg) throws ArgsException {
        for (int i = 1; i < arg.length(); i++)
            parseElement(arg.charAt(i));
    }

    private void parseElement(char argChar) throws ArgsException {
        if (setArgument(argChar))
            argsFound.add(argChar);
        else {
            unexpectedArguments.add(argChar);
        errorCode = ErrorCode.UNEXPECTED_ARGUMENT;
        valid = false;
        }
    }

    private boolean setArgument(char argChar) throws ArgsException {
        if (isBooleanArg(argChar))
            setBooleanArg(argChar, true);
        else if (isStringArg(argChar))
            setStringArg(argChar);
        else if (isIntArg(argChar))
            setIntArg(argChar);
        else
            return false;
        return true;
    }

    private boolean isIntArg(char argChar) {return intArgs.containsKey(argChar);}

    private void setIntArg(char argChar) throws ArgsException {
        currentArgument++;
        String parameter = null;
        try {
            parameter = args[currentArgument];
            intArgs.put(argChar, new Integer(parameter));
        } catch (ArrayIndexOutOfBoundsException e) {
            valid = false;
            errorArgumentId = argChar;
            errorCode = ErrorCode.MISSING_INTEGER;
            throw new ArgsException();
        } catch (NumberFormatException e) {
            valid = false;
            errorArgumentId = argChar;
            errorParameter = parameter;
            errorCode = ErrorCode.INVALID_INTEGER;
            throw new ArgsException();
        }
    }

    private void setStringArg(char argChar) throws ArgsException {
        currentArgument++;
        try {
            stringArgs.put(argChar, args[currentArgument]);
        } catch (ArrayIndexOutOfBoundsException e) {
            valid = false;
            errorArgumentId = argChar;
            errorCode = ErrorCode.MISSING_STRING;
            throw new ArgsException();
        }
    }

    private boolean isStringArg(char argChar) {
        return stringArgs.containsKey(argChar);
    }

    private void setBooleanArg(char argChar, boolean value) {
        booleanArgs.put(argChar, value);
    }

    private boolean isBooleanArg(char argChar) {
        return booleanArgs.containsKey(argChar);
    }

    public int cardinality() {
        return argsFound.size();
    }

    public String usage() {
        if (schema.length() > 0)
            return "-[" + schema + "]";
        else
            return "";
    }

    public String errorMessage() throws Exception {
        switch (errorCode) {
            case OK:
                throw new Exception("TILT: Should not get here.");
            case UNEXPECTED_ARGUMENT:
                return unexpectedArgumentMessage();
            case MISSING_STRING:
                return String.format("Could not find string parameter for -%c.",
                                errorArgumentId);
            case INVALID_INTEGER:
                return String.format("Argument -%c expects an integer but was '%s'.",
                                errorArgumentId, errorParameter);
            case MISSING_INTEGER:
                return String.format("Could not find integer parameter for -%c.",
                                errorArgumentId);
        }
        return "";
    }

    private String unexpectedArgumentMessage() {
        StringBuffer message = new StringBuffer("Argument(s) -");
        for (char c : unexpectedArguments) {
            message.append(c);
        }
        message.append(" unexpected.");
        return message.toString();
    }

    private boolean falseIfNull(Boolean b) {
        return b != null && b;
    }

    private int zeroIfNull(Integer i) {
        return i == null ? 0 : i;
    }

    private String blankIfNull(String s) {
        return s == null ? "" : s;
    }

    public String getString(char arg) {
        return blankIfNull(stringArgs.get(arg));
    }

    public int getInt(char arg) {
        return zeroIfNull(intArgs.get(arg));
    }

    public boolean getBoolean(char arg) {
        return falseIfNull(booleanArgs.get(arg));
    }

    public boolean has(char arg) {
        return argsFound.contains(arg);
    }

    public boolean isValid() {
        return valid;
    }

    private class ArgsException extends Exception {
    }
}
```

上面这段代码是怎么产生的呢？

因为一开始Args类只是支持布尔类型，那时候代码还算是整洁。但是在相继加入了string的支持后，代码就失控了，变得混乱起来。

### 1）先暂停

想象一下，如果继续添加新的参数类型，代码将更加糟糕。

所以先暂停添加特性，开始重构代码。

### 2）渐进

毁坏程序的最好方法之一就是以改进之名，大动其结构。有些程序永远不能从这种所谓“改进”中恢复过来。

问题在于：很难让程序以“改进”之前的方式工作。

为了避免这种情况发生，我采用了TDD（测试驱动开发）的规程。

所以需要一套能随需运行、确保系统行为不会改动的自动化测试。那么通过这个测试系统，就可以确认系统是否以自己预期的方式运行。

接下来就是抽取概念，渐进改动，运行UT。

