---
layout: post
title: copyProperties 方法踩坑记（Spring BeanUtils 源码分析）
categories: spring
description: Spring BeanUtils
keywords: Java, Spring
---

使用 Spring BeanUtils 的时候，踩了一次坑，拷贝丢失了一个属性，debug 的时候，发现原本是 String 的属性，变成了 boolean ，引发了此文。

## 一、 先找到问题的源头

查看实体类，发现了如下方法：

```
public boolean isVerbose() {
    return "1".equals(getVerbose());
}
```

找到了 boolean 的源头，然后就看看 BeanUtils 的源码，找找为什么会把 isVerbose() 这个方法返回值当作 getxxx()。 

## 二、 查看 Spring BeanUtils 的源码

> 这里参考了[Spring BeanUtils源码分析](https://segmentfault.com/a/1190000014833730)

### copyProperties() 方法

```
private static void copyProperties(Object source, Object target, Class<?> editable, String... ignoreProperties)
        throws BeansException {
    // 检查source和target对象是否为null，否则抛运行时异常
    Assert.notNull(source, "Source must not be null");
    Assert.notNull(target, "Target must not be null");
    // 获取target对象的类信息
    Class<?> actualEditable = target.getClass();
    // 若editable不为null，检查target对象是否是editable类的实例，若不是则抛出运行时异常
    // 这里的editable类是为了做属性拷贝时限制用的
    // 若actualEditable和editable相同，则拷贝actualEditable的所有属性
    // 若actualEditable是editable的子类，则只拷贝editable类中的属性
    if (editable != null) {
        if (!editable.isInstance(target)) {
            throw new IllegalArgumentException("Target class [" + target.getClass().getName() +
                    "] not assignable to Editable class [" + editable.getName() + "]");
        }
        actualEditable = editable;
    }
    // 获取目标类的所有PropertyDescriptor，getPropertyDescriptors这个方法请看下方
    PropertyDescriptor[] targetPds = getPropertyDescriptors(actualEditable);
    List<String> ignoreList = (ignoreProperties != null ? Arrays.asList(ignoreProperties) : null);

    for (PropertyDescriptor targetPd : targetPds) {
        // 获取该属性对应的set方法
        Method writeMethod = targetPd.getWriteMethod();
        // 属性的set方法存在 且 该属性不包含在忽略属性列表中
        if (writeMethod != null && (ignoreList == null || !ignoreList.contains(targetPd.getName()))) {
            // 获取source类相同名字的PropertyDescriptor
            PropertyDescriptor sourcePd = getPropertyDescriptor(source.getClass(), targetPd.getName());
            if (sourcePd != null) {
                // 获取对应的get方法
                // *************************** 注意这里 ******************************
                Method readMethod = sourcePd.getReadMethod();
                // set方法存在 且 target的set方法的入参是source的get方法返回值的父类或父接口或者类型相同
                // 具体ClassUtils.isAssignable()
                if (readMethod != null &&
                        ClassUtils.isAssignable(writeMethod.getParameterTypes()[0], readMethod.getReturnType())) {
                    try {
                        //get方法是否是public的
                        if (!Modifier.isPublic(readMethod.getDeclaringClass().getModifiers())) {
                            //暴力反射，取消权限控制检查
                            readMethod.setAccessible(true);
                        }
                        //获取get方法的返回值
                        Object value = readMethod.invoke(source);
                        // 原理同上
                        if (!Modifier.isPublic(writeMethod.getDeclaringClass().getModifiers())) {
                            writeMethod.setAccessible(true);
                        }
                        // 将get方法的返回值 赋值给set方法作为入参
                        writeMethod.invoke(target, value);
                    }
                    catch (Throwable ex) {
                        throw new FatalBeanException(
                                "Could not copy property '" + targetPd.getName() + "' from source to target", ex);
                    }
                }
            }
        }
    }
}
```

### getReadMethod() 方法

重点就在这里了，看到这个方法就一目了然了，这里优先取到的是 is 方法，这样就正好导致了上面的拷贝的时候，缺了这个属性。

```
public synchronized Method getReadMethod() {
    Method readMethod = this.readMethodRef.get();
    if (readMethod == null) {
        Class<?> cls = getClass0();
        if (cls == null || (readMethodName == null && !this.readMethodRef.isSet())) {
            // The read method was explicitly set to null.
            return null;
        }
        String nextMethodName = Introspector.GET_PREFIX + getBaseName();
        if (readMethodName == null) {
            Class<?> type = getPropertyType0();
            if (type == boolean.class || type == null) {
                readMethodName = Introspector.IS_PREFIX + getBaseName();
            } else {
                readMethodName = nextMethodName;
            }
        }

        // Since there can be multiple write methods but only one getter
        // method, find the getter method first so that you know what the
        // property type is.  For booleans, there can be "is" and "get"
        // methods.  If an "is" method exists, this is the official
        // reader method so look for this one first.
        readMethod = Introspector.findMethod(cls, readMethodName, 0);
        if ((readMethod == null) && !readMethodName.equals(nextMethodName)) {
            readMethodName = nextMethodName;
            readMethod = Introspector.findMethod(cls, readMethodName, 0);
        }
        try {
            setReadMethod(readMethod);
        } catch (IntrospectionException ex) {
            // fall
        }
    }
    return readMethod;
}
```

