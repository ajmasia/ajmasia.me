---
title: "Practical javascript: reduce method"
canonicalURL: https://ajmasia.me/en/posts/2019/javascript-practico-metodo-reducer
date: 2019-11-19
draft: false
author: "Antonio José Masiá"
tags:
  - Javascript
---

According to the official documentation from [Mozilla](https://developer.mozilla.org/en-US/), the [reduce()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce) method executes a reducer function on each of the elements of an array, returning a single value as a result.

```js
array.reduce(callback(accumulator, currentValue, [index], [array]), [initValue])
```

This method must receive as a mandatory parameter the **reducer function** callback and optionally can receive an **initial value**.

In turn, the reducer function must mandatorily receive an **accumulator** and the **current iteration value**, and as optional values, the **iteration index** and the **array** on which the reduce() method was called.

Let's look at a simple example. Imagine we are developing a billing application and, logically, we need to generate invoices either from a shopping cart or for services that need to be billed.

All the invoice items are in an array:

```jsx
const invoiceItems = [
    { service: 'consultoría', price: 3000 },
    { service: 'desarrollo', price: 18000 },
    { service: 'soporte', price: 1000 }
]
```

We will define a reducer function that will help us calculate the total of the invoice:

```jsx
const sumItems = (invoiceAmount, nextItem) => invoiceAmount + nextItem.price; 
```

To obtain the total invoice amount, we apply the reduce() method to our array of invoice items:

```jsx
const invoiceAmount = invoiceItems.reduce(sumItems, 0);
```

Obtaining a total value of 3000 + 18000 + 1000 equal to 22000.

Now imagine, for example, that we can apply different types of discounts and taxes independently to each item on our invoice:


```jsx
const invoiceItems = [
  { service: "consultoría", price: 3000, discount: 0.1, tax: 0.18 },
  { service: "desarrollo", price: 18000, discount: 0.18, tax: 0.18 },
  { service: "soporte", price: 1000, discount: 0.08, tax: 0.18 }
];
```
We define a new reducer function that allows us to obtain the total discount of the invoice:


```jsx
const sumDiscount = 
  (invoiceDiscountAmount, nextItem) => invoiceDiscountAmount + (nextItem.price * nextItem.discount);  
```

Once the function is defined, to calculate the invoice discount we will again apply the `reduce` method:


```jsx
const invoiceDiscount = invoiceItems.reduce(sumDiscount, 0);
```

Obtaining a discount value according to the definition of our items of `3620`.

We could continue creating reducers to calculate everything related to obtaining the total data of our invoice, but the truth is that it would be a bit cumbersome and inefficient. Ideally, we would have a method that, by passing it an array of items, would return an object with all the total data of the invoice, like this one:


```jsx
const invoiceResult = {
    amount: 22000,
    discount: 3620,
    base: 18380,
    tax: 3859.9,
    total: 22239.8
}
```

And this is where the power of the reducer concept comes into play, which is used for example in [redux](https://redux.js.org/) and has also been implemented in [react hooks](https://reactjs.org/docs/hooks-reference.html#usereducer).

First, we will create a series of auxiliary helper methods:

```jsx
export const round = (value, decimals = 0) =>
  value ? Number(Math.round(value + "e" + decimals) + "e-" + decimals) : 0;

export const iOperators = {
  getDiscount: (price, discount = 0) => (price ? price * discount : 0),
  getBase: (price, discount = 0) => (price ? price - price * discount : 0),
  getTax: (price, discount = 0, tax = 0) =>
    price ? (price - price * discount) * tax : 0,
  getTotal: (price, discount = 0, tax = 0) => {
    const itemBase = price ? price - price * discount : 0;
    return itemBase + itemBase * tax;
  }
};
```

Now we define the necessary reducers to obtain the values of our invoice:

```jsx
const invoiceReducers = {
  sumItems: (state, item) => {
    state.amount = round(state.amount + item.price, 2);
    return state;
  },
  sumDiscount: (state, item) => {
    state.discount = round(
      state.discount + iOperators.getDiscount(item.price, item.discount),
      2
    );
    return state;
  },
  sumBase: (state, item) => {
    round(
      (state.base = state.base + iOperators.getBase(item.price, item.discount)),
      2
    );
    return state;
  },
  sumTax: (state, item) => {
    state.tax = round(
      state.tax + iOperators.getTax(item.price, item.discount, item.tax),
      2
    );
    return state;
  },
  sumInvoice: (state, item) => {
    state.total = round(
      state.total + iOperators.getTotal(item.price, item.discount, item.tax),
      2
    );
  }
};
```

We need a method that combines all the reducers at once:

```jsx
const invoiceReducer = reducers => (state, item) =>
  Object.keys(reducers).reduce((nextState, key) => {
    reducers[key](state, item);
    return state;
  }, {});

// Combine reducers
const invoiceValuesReducer = invoiceReducer(invoiceReducers);
```

To obtain the value of the invoice, we will define an initial state of it and then apply the reducers to obtain the values of the invoice:

```jsx
var initialState = {
	amount: 0,
    discount: 0,
    base: 0,
    tax: 0,
    total: 0
} 

const invoiceResult = invoiceItems.reduce(invoiceValuesReducer, initialState);
```

Obtaining the following object with the values of our invoice:

```jsx
{
	amount: 22000,
    discount: 3620,
    base: 18380,
    tax: 21348.4,
    total: 21688.4
}
```

You can see the code for this example at [CodeSandbox](https://codesandbox.io/embed/javascript-reducers-socdw?expanddevtools=1&fontsize=14&hidenavigation=1&theme=dark&view=editor)

