---
title: Spring事务回滚小妙招
comments: true
categories: Spring
tags: Spring
abbrlink: 3d2f3823
date: 2025-05-30 16:46:58
---

> spring事务控制手动回滚 ~

<!--more--> 

### 1、说明

*   事务是我们开发过程中经常会使用到的，为了在业务执行过程中出现异常时，回滚到原始状态。而事务的回滚在大多数情况下都是靠着 **exception**（异常）来触发回滚的，当事务机制捕捉到异常，就会开始回滚。
*   但往往也会出现**情况**：在业务代码中，需要对异常单独进行处理，异常不会抛出，但需要事务回滚的情况，这个时候就需要手动调用回滚



### 2、@Transactional

@Transactional 声明式事务，是开发过程中最常用的开启事务的方式。

**优点是：使用方便，而且对代码的侵入性低**

*   这种方式，默认是通过 捕获 **`RunTimeException`** 来触发事务回滚，只要是 `RuntimeException` 以及 它的子类，都可以触发事务回滚。
*   也可以修改 触发事务回滚的异常范围，可以通过修改 @Transactional 中的 `rollbackFor` 属性，修改异常范围。比如：  
    `@Transactional(rollbackFor = Exception.class)`，修改完成后，只要是 Exception 类的子类，都可以触发事务回滚

当然也会有**事务失效的情况**，可以详细参考另一篇文章[@Transactional注解事务失效的几种场景及原因](https://www.cnblogs.com/xiangningdeguang/p/16955161.html)



### 3、@Transactional不适用场景

当在业务中，需要对异常进行特殊处理的时候，异常无法抛出时，声明式事务就不太适用了。  
**示例：**

```java
    //假设这是一个service类的片段
    try{ 
        //出现异常
    } catch (Exception e) {
        // 捕获异常，打印异常，或其他处理。但不抛出新的异常
        e.printStackTrace();
    }
    //此时return语句能够执行
    return  xxx;
```

当不抛出新异常时，就无法触发 事务回滚。

**此处，也可以将捕获后的异常，封装为新的业务异常抛出**



### 4、spring事务控制手动回滚

另外一种方式，就是使用 spring 事务控制的手动回滚，不抛出异常，直接调用回滚语句  
**示例：**

```java
    //假设这是一个service类的片段
    try{ 
        //出现异常
    } catch (Exception e) {
        // 捕获异常，打印异常，或其他处理。但不抛出新的异常
        e.printStackTrace();
        // 手动回滚
        TransactionAspectSupport.currentTransactionStatus().setRollbackOnly();
    }
    //此时return语句能够执行
    return  xxx;
```



### 5、spring编程式事务

`TransactionTemplate` 是 Spring 提供的 **编程式事务管理工具**，通过代码显式控制事务的开启、提交和回滚，适用于需要动态或复杂事务逻辑的场景。

#### （1）支持返回值

**`execute(TransactionCallback<T> action)`**

- **功能**：执行包含事务的代码块，支持返回值。

- 代码示例：

  ```java
  @Resource
  private TransactionTemplate transactionTemplate;
  
  User user = transactionTemplate.execute(status -> {
       
      try {
       
          User newUser = userDao.save(user); // 事务操作
          return newUser;
      } catch (DataAccessException e) {
          status.setRollbackOnly(); // 标记回滚
          throw new ServiceException("保存用户失败", e);
      }
  });
  ```

#### （2）支持无返回值

**`executeWithoutResult(Consumer<TransactionStatus> action)`**

- **功能**：执行无返回值的事务操作。

- 代码示例：

  ```java
  @Resource
  private TransactionTemplate transactionTemplate;
  
  transactionTemplate.executeWithoutResult(status -> {
      userDao.updateBalance(userId, amount); // 扣减余额
      orderDao.createOrder(order);           // 创建订单
  });
  ```