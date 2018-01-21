---
layout: post
category: post
title: "Interfaces are great, but let's do better"
tags: [code, c#, AOP, Autofac, .net]
---

## Table of Contents

- [Introduction](#introduction)
- [Leaking implementation details through naming](#leaking-implementation-details-through-naming)
- [Leaking implementation details through consuming code](#leaking-implementation-details-through-consuming-code)
- [Impossible contracts or Inconvenient contracts](#impossible-contracts-or-inconvenient-contracts)

### Introduction

Interfaces are great. For many people the first thing that comes to mind when asked about interfaces is _unit testing_. When used in conjuction with the inversion of control principle it allows for very powerful testing. By utilizing mock implementations we can setup all kinds of test scenarios and create automated tests to assert our unit works as intended under these circumstances. Excellent.

Unfortunately for many people it stops there. They create interfaces just for the sake of creating interfaces and if they're in a productive mood they may even write a couple of unit tests. I find this very unfortunate because an interface is so much more than just an entrypoint to inject mocks.  
<mark>To me an interface is a contract the consumer can compose so it can invoke functionality it cannot and should not implement itself. The consuming part should be one hundred percent unaware of the underlying implementation of the interface. The contract should expose functionality in such a way that is most convenient for the consumer. This way the consumer can focus on its own job and keep the complexity out.</mark>

In this post I will go over a few common practices that (in my opinion) are a direct violation to these principles. I will explain how to recognize them, why they are an issue and what you can do to avoid and/or fix them. 
  
  &nbsp;

### Leaking implementation details through naming

```csharp
public sealed class Consumer {
  private readonly IBillingAdapter _billingAdapter;

  public Consumer(IBillingAdapter billingAdapter)
  {
    _billingAdapter = billingAdapter;
  }
}
```

Here the focus is on the **adapter** part in `IBillingAdapter`. Your consumer requires some functionality that is exposed by _billing_ - why even add the adapter suffix? My guess is the implementation is calling an external endpoint for which your implementation will be like the adapter pattern.  
Although this is not really an issue but rather one of my pet-peeves, it's not a good sign. I wouldn't block a code review for this but I would address this with the developer and hopefully they will consider this in the future.
  
  &nbsp;

### Leaking implementation details through consuming code

```csharp
public sealed class Consumer {
  private readonly IInvoices _invoices;

  ...

  public PriceJustification GetPriceJustification(List<PayClosing> payClosings)
  {
    var payClosingsPerClient = GroupPerClient(payClosings);

    foreach(var group in payClosingsPerClient)
    {
      var clientReference = group.Key;
      var invoices = _invoices.GetAllForClient(clientReference);
      ...
    }
  }

  private static IGrouping<string, PayClosing> GroupPerClient(List<PayClosing> payClosings)
  {
    ...
  }
}
```

Here the focus is on the complexity that was introduced because of premature optimization. The goal was to find the amount that was charged for a payclosing by getting the invoice and getting the amount from there. For some reason the consumer was afraid to query the invoice collection too often? Why? Most likely because the implementation will target an external system which may (or may not!) be slow/expensive.  
The developer was aware of the implementation and decided to introduce complexity from the start. **Premature optimization is the root of all evil.** 

There's a few ways you can address this issue:

1. Always start from the consuming end and take the simplest route to handle your usecase. More on this in my next point
2. Measure performance first before taking any decisions
3. Use Aspect Orient Programming (AOP) to take some load off the actual system. (Cache method invocation responses with an interceptor)

  &nbsp;

### Impossible contracts or Inconvenient contracts

```csharp
public interface IRepository<TEntity> 
{
  List<TEntity> Query(Expression<Func<TEntity, bool>> filterExpression);
}
```

This is in my opinion one of the worst interface violations out there. I will almost always reject a code review if I see this. It's bad in two ways:

***Impossible contract***

Imagine having to write an implementation for this - knowing the consumer can pass in **any** logic in the query and you are somehow supposed to be able to handle this? As far as I am aware only an in-memory collection can do this. This kind of pattern is often used in conjunction with EntityFramework however. EntityFramework can translate linq to sql and so it is able to support some of these cases, **but not all**. As soon as the consumer passes in a bit of custom code EntityFramework will no longer be able to translate this and your implementation will break. The consumer however sees this contract and it will, and should, expect that whatever the contract provides, will work, but it does **NOT**.

***Inconvenient contract***

It may be cool that you can reuse the same method in multiple scenarios. But this cool factor will wear off very quickly because soon you will realise you're constantly duplicating filter logic all accross your application layer. You could define those filter expressions somewhere common but honestly at this point it's already too late.  
Instead go for methods per kind of query:
`invoiceCollection.GetAllForClient(clientReference);`  
`invoiceCollection.GetAllInclusiveBetween(startDate, endDate);`


