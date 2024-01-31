---
title: "Javascript práctico: método reduce"
canonicalURL: https://ajmasia.me/posts/2019/javascript-practico-metodo-reducer
date: 2019-11-19
draft: false
author: "Antonio José Masiá"
tags:
  - Javascript
---

Según la documentación oficial de [Mozilla](https://developer.mozilla.org/en-US/), el método [reduce()](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/reduce), ejecuta una función reductora sobre cada uno de los elementos de una array, devolviendo como resultado un único valor.

```js
array.reduce(callback(accumulator, currentValue, [index], [array]), [initValue])
```

Este método ha de recibir como parámetro obligatorio la **función reductora** callback y como opcional puede recibir un **valor inicial**.

A su vez, la función reductora ha de recibir de forma obligatoria un **acumulador** y el **valor actual de la iteración**, y como valores opcionales, el **índice de la iteración** y el **array** sobre el que se llamo el método reduce().

Veamos un ejemplo sencillo. Imaginemos que estamos desarrollando una aplicación de facturación y como es lógico necesitamos generar facturas a partir bien desde un carrito de la compra o bien unos servicios que hemos de facturar.

Todos los items de la factura los tenemos en un array:

```js
const invoiceItems = [
    { service: 'consultoría', price: 3000 },
    { service: 'desarrollo', price: 18000 },
    { service: 'soporte', price: 1000 }
]
```

Definiremos una función reductora que nos servirá para calcular el total de la factura:

```js
const sumItems = (invoiceAmount, nextItem) => invoiceAmount + nextItem.price; 
```

Para obtener el total de la factura, aplicamos el método reduce() sobre nuestro array de items de la factura:

```js
const invoiceAmount = invoiceItems.reduce(sumItems, 0);
```

Obteniendo un valor total de 3000 + 18000 + 1000 igual a 22000.

Imaginemos ahora por ejemplo que podemos aplicar distintos tipos de descuento e impuestos de forma independiente a cada item de nuestra factura:

```js
const invoiceItems = [
  { service: "consultoría", price: 3000, discount: 0.1, tax: 0.18 },
  { service: "desarrollo", price: 18000, discount: 0.18, tax: 0.18 },
  { service: "soporte", price: 1000, discount: 0.08, tax: 0.18 }
];
```

Definimos una nueva función reductora que nos permita obtener el descuento total que tendrá la factura:

```js
const sumDiscount = 
  (invoiceDiscountAmount, nextItem) => invoiceDiscountAmount + (nextItem.price * nextItem.discount);  
```

Una vez definida la función, para calcular el descuento de la factura aplicaremos de nuevo el método `reduce`:

```js
const invoiceDiscount = invoiceItems.reduce(sumDiscount, 0);
```

Obteniendo un valor de descuento según lo definición de nuestros items de `3620`.

Podríamos seguir creando reducers para calcular todo aquello relacionado con la obtención de los datos totales de nuestra factura, pero la verdad es que resultaría un poco engorroso e ineficiente. Lo ideal sería tener un método que pasándole un array de items nos devolviera un objeto con todos los datos que totales de la factura, como por ejemplo este:

```js
const invoiceResult = {
    amount: 22000,
    discount: 3620,
    base: 18380,
    tax: 3859.9,
    total: 22239.8
}
```

Y es aquí donde entra en juego la potencia del concepto de reducer que se usa por ejemplo en [redux](https://redux.js.org/) y que también se ha implementado en los [hooks de react](https://reactjs.org/docs/hooks-reference.html#usereducer).

En primer lugar crearemos una serie de métodos auxiliares de ayuda:

```js
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

Ahora definimos los reducers necesarios para obtener los valores de nuestra factura:

```js
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

Necesitamos un método que nos combine todos los reducers a la vez:

```js
const invoiceReducer = reducers => (state, item) =>
  Object.keys(reducers).reduce((nextState, key) => {
    reducers[key](state, item);
    return state;
  }, {});

// Combine reducers
const invoiceValuesReducer = invoiceReducer(invoiceReducers);
```

Para obtener el valor de la factura, definiremos un estado inicial de la misma y posteriormente aplicaremos los reducers para obtener los valores de la factura:

```js
var initialState = {
	amount: 0,
    discount: 0,
    base: 0,
    tax: 0,
    total: 0
} 

const invoiceResult = invoiceItems.reduce(invoiceValuesReducer, initialState);
```

Obteniendo el siguiente objeto con los valores de nuestra factura:

```js
{
	amount: 22000,
    discount: 3620,
    base: 18380,
    tax: 21348.4,
    total: 21688.4
}
```

Puedes ver el código de este ejemplo en [CodeSandbox](https://codesandbox.io/embed/javascript-reducers-socdw?expanddevtools=1&fontsize=14&hidenavigation=1&theme=dark&view=editor)
