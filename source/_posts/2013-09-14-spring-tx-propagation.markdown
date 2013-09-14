---
layout: post
title: "Spring 事务的传播特性"
date: 2013-09-14 18:06
comments: true
categories: [Programming, Java, Spring]
---

最近工作中涉及到了一个分布式事务的产品，这个产品是在 Spring 的事务上做的，我对其中涉及到的 Spring 的事务的传播特性不是很了解，所以今天花了一个下午的时间认真了解了一下，写了一堆的测试代码。

进入正题，Spring 的事务的传播特性分为以下的七种：

* PROPAGATION_REQUIRED
* PROPAGATION_SUPPORTS
* PROPAGATION_MANDATORY
* PROPAGATION_REQUIRES_NEW
* PROPAGATION_NOT_SUPPORTED
* PROPAGATION_NEVER
* PROPAGATION_NESTED

下面一种种来解释：

#### PROPAGATION_REQUIRED

默认的事务传播特性，通常情况下我们用这个事务传播特性就可以了。如果当前事务上下文中没有事务，那么就会新创建一个事务。如果当前的事务上下文中已经有一个事务了，那么新开启的事务会开启一个新的逻辑事务范围，但是会和原有的事务共用一个物理事务，我们暂且不关心逻辑事务和物理事务的区别，先看看这样会导致怎样的代码行为：

```java
@Override
public void txRollbackInnerTxRollbackPropagationRequires() {
    transactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
            jdbcTemplate.update("insert into user (name, password) values (?, ?)", "Huang",
                "1111112");
            transactionTemplate.execute(new TransactionCallbackWithoutResult() {
                @Override
                protected void doInTransactionWithoutResult(TransactionStatus status) {
                    jdbcTemplate.update("insert into user (name, password) values (?, ?)",
                        "Huang", "1111112");
                    // 内部事务设置了 setRollbackOnly，
                    status.setRollbackOnly();
                }
            });
        }
    });
}
```

这段代码在一个嵌套的事务内部（内外层的事务都是 PROPAGATION_REQUIRED 的）设置了回滚，那么对于外部的事务来说，它会收到一个 `UnexpectedRollbackException`，因为内部的事务和外部的事务是共用一个物理事务的，所以显然内部的事务必然会导致外部事务的回滚，但是因为这个回滚并不是外部事务自己设置的，所以外层事务回滚的时候会需要抛出一个 `UnexpectedRollbackException`，让事务的调用方知道这个回滚并不是外部事务自己想要回滚，是始料未及的。

但是，如果内层的事务不是通过设置 `setRollbackOnly()` 来回滚，而是抛出了 `RuntimeException` 来回滚，那么外层的事务接收到了内层抛出的 `RuntimeException` 也会跟着回滚，这个是可以预料到的行为，所以不会有 `UnexpectedRollbackException` 。

#### PROPAGATION_SUPPORTS

PROPAGATION_SUPPORTS 的特性是如果事务上下文中已经存在一个事务，那么新的事务（传播特性为 PROPAGATION_SUPPORTS）就会和原来的事务共用一个物理事务，其行为和 PROPAGATION_REQUIRED 一样。但是，如果当前事务上下文中没有事务，那么 PROPAGATION_SUPPORTS 就按无事务的方式执行代码：

```java
@Override
public void txRollbackInnerTxRollbackPropagationSupports() {
    supportsTransactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus status) {
            jdbcTemplate.update("insert into user (name, password) values (?, ?)", "Huang",
                "1111112");
            throw new CustomRuntimeException();
        }
    });
}
```

看上面这段代码，虽然我们在事务（PROPAGATION_SUPPORTS 的）中抛出了一个 `RuntimeException`，但是因为其事务上下文中没有事务存在，所以这段代码实际上是以无事务的方式执行的，因此代码中的 `jdbcTemplate.update()` 操作也不会被回滚。

#### PROPAGATION_MANDATORY

PROPAGATION_MANDATORY 要求事务上下文中必须存在事务，如果事务上下文中存在事务，那么其行为和 `PROPAGATION_REQUIRED` 一样。如果当前事务上下文中没有事务，那么就会抛出 `IllegalTransactionStateException`，比如下面这段代码就会这样：

```java
@Override
public void txRollbackInnerTxRollbackPropagationMandatory() {
    mandatoryTransactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
            jdbcTemplate.update("insert into user (name, password) values (?, ?)", "Huang",
                "1111112");
        }
    });
}
```

#### PROPAGATION_REQUIRES_NEW

PROPAGATION_REQUIRES_NEW 无论当前事务上下文中有没有事务，都会开启一个新的事务，并且和原来的事务完全是隔离的，外层事务的回滚不会影响到内层的事务，内层事务的回滚也不会影响到外层的事务（**这个说法得稍微打点折扣：因为如果内层抛出 `RuntimeException` 的话，那么外层还是会收到这个异常并且触发回滚**），我们分析下几段代码：


```java
@Override
public void txRollbackInnerTxRollbackPropagationRequiresNew() {
    transactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {

            requiresNewTransactionTemplate.execute(new TransactionCallbackWithoutResult() {
                @Override
                protected void doInTransactionWithoutResult(TransactionStatus status) {
                    jdbcTemplate.update("insert into user (name, password) values (?, ?)",
                        "Huang", "1111112");
                }
            });

            // 外部事务发生回滚，内部事务应该不受影响还是能够提交
            throw new RuntimeException();
        }
    });
}
```

这段代码外层的事务回滚了，但是不会影响到内层的事务的提交，内层事务不受外层的事务的影响。再看：

```java
@Override
public void txRollbackInnerTxRollbackPropagationRequiresNew2() {
    transactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
            jdbcTemplate.update("insert into user (name, password) values (?, ?)", "Huang",
                "1111112");
            // Nested transaction committed.
            requiresNewTransactionTemplate.execute(new TransactionCallbackWithoutResult() {
                @Override
                protected void doInTransactionWithoutResult(TransactionStatus status) {
                    jdbcTemplate.update("insert into user (name, password) values (?, ?)",
                        "Huang", "1111112");
                    // 内部事务发生回滚，但是外部事务不应该发生回滚
                    status.setRollbackOnly();
                }
            });
        }
    });
}
```

这段代码在内层事务上设置了 `setRollbackOnly`，内层事务肯定会回滚，但是由于内层事务和外层事务是隔离的，所以外层事务不会被回滚。再看：

```java
@Override
public void txRollbackInnerTxRollbackPropagationRequiresNew3() {
    transactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
            jdbcTemplate.update("insert into user (name, password) values (?, ?)", "Huang",
                "1111112");

            requiresNewTransactionTemplate.execute(new TransactionCallbackWithoutResult() {
                @Override
                protected void doInTransactionWithoutResult(TransactionStatus status) {
                    jdbcTemplate.update("insert into user (name, password) values (?, ?)",
                        "Huang", "1111112");
                    // 内部事务抛出 RuntimeException，外部事务接收到异常，依旧会发生回滚
                    throw new RuntimeException();
                }
            });
        }
    });
}
```

这段代码在内层事务抛出了一个 `RuntimeException`，虽然内层事务和外层事务在事务上是隔离，但是 `RuntimeException` 显然还会抛到外层去，所以外层事务也会发生回滚。

#### PROPAGATION_NOT_SUPPORTED

PROPAGATION_NOT_SUPPORTED 不管当前事务上下文中有没有事务，代码都会在按照无事务的方式执行，看下面这段代码：

```java
@Override
public void txRollbackInnerTxRollbackPropagationNotSupport() {
    transactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
            jdbcTemplate.update("insert into user (name, password) values (?, ?)", "Huang",
                "1111112");
            notSupportedTransactionTemplate.execute(new TransactionCallbackWithoutResult() {
                @Override
                protected void doInTransactionWithoutResult(TransactionStatus status) {
                    jdbcTemplate.update("insert into user (name, password) values (?, ?)",
                        "Huang", "1111112");
                }
            });
            // 外部事务回滚，不会把内部的也连着回滚 
            transactionStatus.setRollbackOnly();
        }
    });
}
```

上面这段代码中虽然外部事务发生了回滚，但是由于内部的事务是 PROPAGATION_NOT_SUPPORTED，根本不在外层的事务范围内，所以内层事务不会发生回滚。

#### PROPAGATION_NEVER

PROPAGATION_NEVER 如果当前事务上下文中存在事务，就会抛出 `IllegalTransactionStateException` 异常，自己也会按照非事务的方式执行

```java
@Override
public void txRollbackInnerTxRollbackPropagationNever2() {
    transactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
            jdbcTemplate.update("insert into user (name, password) values (?, ?)", "Huang",
                "1111112");
            neverTransactionTemplate.execute(new TransactionCallbackWithoutResult() {
                @Override
                protected void doInTransactionWithoutResult(TransactionStatus status) {
                    jdbcTemplate.update("insert into user (name, password) values (?, ?)",
                        "Huang", "1111112");
                }
            });
        }
    });
}
```

比如上面这段代码，在一个 PROPAGATION_REQUIRES 里面嵌入了一个 PROPAGATION_NEVER，内层就会抛出一个 `IllegalTransactionStateException`，导致外层事务被回滚。

#### PROPAGATION_NESTED

PROPAGATION_NESTED 只能应用于像 DataSource 这样的事务，可以通过在一个事务内部开启一个 PROPAGATION_NESTED 而达到一个事务内部有多个保存点的效果，一个内嵌的事务发生回滚，只会回滚到它自己的保存点，外层事务还会继续，比如下面这段代码：

```java
@Override
public void txRollbackInnerTxRollbackPropagationNested() {
    nestedTransactionTemplate.execute(new TransactionCallbackWithoutResult() {
        @Override
        protected void doInTransactionWithoutResult(TransactionStatus transactionStatus) {
            jdbcTemplate.update("insert into user (name, password) values (?, ?)", "Huang",
                "1111112");

            nestedTransactionTemplate.execute(new TransactionCallbackWithoutResult() {
                @Override
                protected void doInTransactionWithoutResult(TransactionStatus status) {
                    jdbcTemplate.update("insert into user (name, password) values (?, ?)",
                        "Huang", "1111112");
                    // 内部事务设置了 rollbackOnly，外部事务应该不受影响，可以继续提交
                    status.setRollbackOnly();
                }
            });
        }
    });
}
```

内层的事务发生了回滚，只会回滚其内部的操作，不会影响到外层的事务。

### 总结

Spring 的事务传播特性种类繁多，大多数都来自于 EJB 的事务，大家可以自己写一些小的程序来测试 Spring 各个事务特性的行为，加深印象，我自己也写了一个工程，通过单元测试去测试各个事务传播特性的行为，大家有兴趣的话，可以下过来跑一下：[https://github.com/khotyn/spring-tx-test](https://github.com/khotyn/spring-tx-test)