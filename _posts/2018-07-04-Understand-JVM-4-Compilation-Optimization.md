---
layout: post
title: 《深入理解Java虚拟机：JVM高级特性与最佳实践--第二版》学习日志（四）： 程序编译与代码优化
date: 2018-07-04 20:26:04
author: admin
comments: true
categories: [Java]
tags: [Java，JVM]
---

程序员对效率的追求，是永无停止的。

<!-- more -->
---

学习资料主要参考： 《深入理解Java虚拟机：JVM高级特性与最佳实践(第二版)》，作者：周志明

---
## 目录
{:.no_toc}

* 目录
{:toc}

---

# 早期（编译期）优化

## 1. 概述

Java的编译期，有很多意思：
- 可以是指前段编译器把 *.java 文件转为 *.class 文件的过程：Eclipse、Javac
- 可以是指虚拟机的后端运行期编译器（JIT编译器， just In time Compiler）把字节码转为机器码的过程：HotSpot VM的C1、C2编译器
- 也可能是指使用静态提前编译器（AOT，Ahead Of Time Compiler）直接把 *.java 文件编译成本地机器代码的过程：GNU Compiler for the java、excelsior JET

本章主要讨论第一类。

## 2. Javac 编译器

### 1）整体过程

从Sun Javac 的代码来看，编译过程大致可以分为3个过程：
1. 解析与填充符号表过程
2. 插入式注解处理器的注解处理过程
3. 分析与字节码生产过程

其中关键的处理由8个方法来完成，来看一看。

### 2）解析与填充符号表

#### 词法、语法分析
    
解析步骤由 parseFiles 方法完成。其中包括词法分析和语法分析两个过程。

词法分析是将字符流转为标记（Token）集合。

语法分析是根据token序列构造抽象语法树的过程。

#### 填充符号表

完成了词法分析和语法分析后，就是填充符号表了。由 enterTrees方法完成。

符号表是由一组符号地址和符号信息构成的表格。

符号表中所登记的信息在编译的不同阶段都要用到。在语义分析阶段，用于语义检查和产生中间代码。在目标代码生成阶段，符号表是对符号名进行地址分配的依据。

### 3）注解处理器

注解与普通的java代码一样，是在运行期间发挥作用的。

在JDK 1.6 中，提供了一组插入式注解处理器的标准API 在编译期间对注解进行处理。它们可以读取、修改、添加抽象语法树中的任意元素。如果修改了语法树，编译器将回到解析及填充符号表过程重新处理，直到所有插入式注解处理器都没有再对语法树进行修改为止，每一次的循环称为一个 Round。

### 4） 语义分析与字节码生成

语法树能够表示一个结构正确的源程序的抽象，但是无法保证源程序是符号逻辑的。这时候就需要语义分析了。包含标注检查和数据及控制流分析两个过程。

#### 标注检查

由attribute方法完成。

检查的内容包括诸如变量使用前是否已被声明、变量与赋值之间的数据类型是否能够匹配。

#### 数据及控制流分析

由flow方法完成。

这一步是对程序上下文逻辑更一步的验证，它可以检测出诸如程序局部吧在使用前是否有赋值、方法的每条路径是否都有返回值、是否所有的受查异常都被正确处理。

#### 解语法糖

使用语法糖能够增加程序的可读性，从而减少代码出错的机会。

比如泛型、变长参数、自动装箱/拆箱等。

#### 字节码生成

这个阶段，不仅仅是把起那么各个步骤所生成的信息转化为字节码写到磁盘上，编译器还进行了少了的代码添加和转换工作。

比如实例构造器 init 方法和类构造器 clinit 方法就是这个阶段添加到语法树之中的。

## 3. Java 语法糖的味道

语法糖，虽然不会提供实质性的功能改进，但是可以提高效率，或者提升语法的严谨性，或者减少编码出错的机会。

### 1）泛型与类型擦除

它的本质是参数化类型的应用，也就是说莎草纸的数据类型被指定为一个参数。这种参数类型可以用在类、接口和方法的创建中，分别称为泛型类、泛型接口和泛型方法。

Java 语言的泛型实现方法称为类型擦除，基于这种方法实现的泛型称为伪泛型。

### 2）自动装箱、拆箱和遍历循环

装箱、拆箱在编译之后，被转化为对应的包装和还原方法，如Integer.valueOf Integer.intValue 方法。

循环遍历把代码还原成迭代器的实现。

### 3）条件编译

```
public static void main(String[] args) {
	if (true) {
		System.out.println("block 1");
	} else {
		System.out.println("block 2");
	}
}
```

这段代码，在编译之后，就只有一段“System.out.println("block 1");”。

## 4. 实战：插入式注解处理器

### 1）实战目标

使用注解处理器API来编写一款拥有自己编码风格的校验工具：NameCheckProcessor。它主要做以下check：
- 类或接口：符合驼峰命名法，首字母大写
- 方法：符合驼峰命名法，首字母小写
- 字段：
    - 类或者实例变量：符合符合驼峰命名法，首字母小写
    - 常量：要求全部由大写字或下划线构成，并且第一个字符不能是下划线

### 2）代码实现

首先注解处理器的代码需要继承抽象类：javax.annotation.processing.AbstractProcessor，覆盖其中的abstract方法：process。

这个方法的第一个参数“annotations”中获取到此注解处理器所要处理的注解集合，从第二个参数“roundEnv”中访问到当前这个Round中的语法树节点，每个语法树节点在这里表示为一个Element。

```
// 可以用"*"表示支持所有Annotations
@SupportedAnnotationTypes("*")
// 只支持JDK 1.6的Java代码
@SupportedSourceVersion(SourceVersion.RELEASE_6)
public class NameCheckProcessor extends AbstractProcessor {

    private NameChecker nameChecker;

    /**
     * 初始化名称检查插件
     */
    @Override
    public void init(ProcessingEnvironment processingEnv) {
        super.init(processingEnv);
        nameChecker = new NameChecker(processingEnv);
    }

    /**
     * 对输入的语法树的各个节点进行进行名称检查
     */
    @Override
    public boolean process(Set<? extends TypeElement> annotations, RoundEnvironment roundEnv) {
        if (!roundEnv.processingOver()) {
            for (Element element : roundEnv.getRootElements())
                nameChecker.checkNames(element);
        }
        return false;
    }

}

```


```
/**
 * 程序名称规范的编译器插件：<br>
 * 如果程序命名不合规范，将会输出一个编译器的WARNING信息
 */
public class NameChecker {
    private final Messager messager;

    NameCheckScanner nameCheckScanner = new NameCheckScanner();

    NameChecker(ProcessingEnvironment processsingEnv) {
        this.messager = processsingEnv.getMessager();
    }

    /**
     * 对Java程序命名进行检查，根据《Java语言规范》第三版第6.8节的要求，Java程序命名应当符合下列格式：
     * 
     * <ul>
     * <li>类或接口：符合驼式命名法，首字母大写。
     * <li>方法：符合驼式命名法，首字母小写。
     * <li>字段：
     * <ul>
     * <li>类、实例变量: 符合驼式命名法，首字母小写。
     * <li>常量: 要求全部大写。
     * </ul>
     * </ul>
     */
    public void checkNames(Element element) {
        nameCheckScanner.scan(element);
    }

    /**
     * 名称检查器实现类，继承了JDK 1.6中新提供的ElementScanner6<br>
     * 将会以Visitor模式访问抽象语法树中的元素
     */
    private class NameCheckScanner extends ElementScanner6<Void, Void> {

        /**
         * 此方法用于检查Java类
         */
        @Override
        public Void visitType(TypeElement e, Void p) {
            scan(e.getTypeParameters(), p);
            checkCamelCase(e, true);
            super.visitType(e, p);
            return null;
        }

        /**
         * 检查方法命名是否合法
         */
        @Override
        public Void visitExecutable(ExecutableElement e, Void p) {
            if (e.getKind() == METHOD) {
                Name name = e.getSimpleName();
                if (name.contentEquals(e.getEnclosingElement().getSimpleName()))
                    messager.printMessage(WARNING, "一个普通方法 “" + name + "”不应当与类名重复，避免与构造函数产生混淆", e);
                checkCamelCase(e, false);
            }
            super.visitExecutable(e, p);
            return null;
        }

        /**
         * 检查变量命名是否合法
         */
        @Override
        public Void visitVariable(VariableElement e, Void p) {
            // 如果这个Variable是枚举或常量，则按大写命名检查，否则按照驼式命名法规则检查
            if (e.getKind() == ENUM_CONSTANT || e.getConstantValue() != null || heuristicallyConstant(e))
                checkAllCaps(e);
            else
                checkCamelCase(e, false);
            return null;
        }

        /**
         * 判断一个变量是否是常量
         */
        private boolean heuristicallyConstant(VariableElement e) {
            if (e.getEnclosingElement().getKind() == INTERFACE)
                return true;
            else if (e.getKind() == FIELD && e.getModifiers().containsAll(EnumSet.of(PUBLIC, STATIC, FINAL)))
                return true;
            else {
                return false;
            }
        }

        /**
         * 检查传入的Element是否符合驼式命名法，如果不符合，则输出警告信息
         */
        private void checkCamelCase(Element e, boolean initialCaps) {
            String name = e.getSimpleName().toString();
            boolean previousUpper = false;
            boolean conventional = true;
            int firstCodePoint = name.codePointAt(0);

            if (Character.isUpperCase(firstCodePoint)) {
                previousUpper = true;
                if (!initialCaps) {
                    messager.printMessage(WARNING, "名称“" + name + "”应当以小写字母开头", e);
                    return;
                }
            } else if (Character.isLowerCase(firstCodePoint)) {
                if (initialCaps) {
                    messager.printMessage(WARNING, "名称“" + name + "”应当以大写字母开头", e);
                    return;
                }
            } else
                conventional = false;

            if (conventional) {
                int cp = firstCodePoint;
                for (int i = Character.charCount(cp); i < name.length(); i += Character.charCount(cp)) {
                    cp = name.codePointAt(i);
                    if (Character.isUpperCase(cp)) {
                        if (previousUpper) {
                            conventional = false;
                            break;
                        }
                        previousUpper = true;
                    } else
                        previousUpper = false;
                }
            }

            if (!conventional)
                messager.printMessage(WARNING, "名称“" + name + "”应当符合驼式命名法（Camel Case Names）", e);
        }

        /**
         * 大写命名检查，要求第一个字母必须是大写的英文字母，其余部分可以是下划线或大写字母
         */
        private void checkAllCaps(Element e) {
            String name = e.getSimpleName().toString();

            boolean conventional = true;
            int firstCodePoint = name.codePointAt(0);

            if (!Character.isUpperCase(firstCodePoint))
                conventional = false;
            else {
                boolean previousUnderscore = false;
                int cp = firstCodePoint;
                for (int i = Character.charCount(cp); i < name.length(); i += Character.charCount(cp)) {
                    cp = name.codePointAt(i);
                    if (cp == (int) '_') {
                        if (previousUnderscore) {
                            conventional = false;
                            break;
                        }
                        previousUnderscore = true;
                    } else {
                        previousUnderscore = false;
                        if (!Character.isUpperCase(cp) && !Character.isDigit(cp)) {
                            conventional = false;
                            break;
                        }
                    }
                }
            }

            if (!conventional)
                messager.printMessage(WARNING, "常量“" + name + "”应当全部以大写字母或下划线命名，并且以字母开头", e);
        }
    }
}

```

### 3）运行与测试

在执行javac命令时，通过“-processor”参数来执行编译时需要附带的注解处理器。

---

# 未完待续.....
