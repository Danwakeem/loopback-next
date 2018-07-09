---
lang: en
title: 'Relations'
keywords: LoopBack 4.0, LoopBack 4
tags:
sidebar: lb4_sidebar
permalink: /doc/en/lb4/Relations.html
summary:
---

## Overview

Model relation in LoopBack 3 is one of its powerful features which help users
define real-world mappings between their models, access sensible CRUD APIs for
each of the models, and add querying and filtering capabilities for the relation
APIs after scaffolding their LoopBack applications. In LoopBack 4, with the
introduction of [repositories](Repositories.md), we aim to simplify the approach
to relations by creating constrained repositories. This means that, based on the
relation definition, certain constraints need to be honoured by the target model
repository, and thus, we produce a constrained version of it as a navigational
property on the source repository. The first relation type we have available is
the `has many` relation which operates by constraining the target repository by
the foreign key property on its associated model.

## Defining a HasMany Relation

Model relations in LoopBack 4 are defined by having a decorator on a designated
relational property. The definition of the `has many` relation is done by using
the `hasMany` decorator. The decorator takes in the target model class
constructor and optionally a custom foreign key to store the relation metadata.
The decorator logic also designates the relation type and tries to infer the
foreign key on the target model (or `keyTo` in the relation metadata) as a
sensible default value which is used in LoopBack 3. It also calls
`property.array()` to ensure that the type of the property is inferred properly
as an array of the target model instances. The decorated property name is used
as the relation name and stored as part of the source model definition's
relation metadata. The following example shows how to define a `hasMany`
relation on a source model `Customer`.

{% include code-caption.html content="/src/models/customer.model.ts" %}

```ts
import {Order} from './order.model.ts';
import {Entity, property, hasmany} from '@loopback/repository';

export class Customer extends Entity {
  @property({
    type: 'number',
    id: true,
  })
  id: number;

  @property({
    type: 'string',
  })
  name: string;

  @hasMany(Order) orders: Order[];
}
```

The metadata generated for the above decoration is as follows:

```js
{
  type: 2; // denotes hasMany relation
  keyTo: customerId; // Order model should have this property declared
}
```

The property type metadata is also preserved as an array of type `Order` as part
of the decoration. Another usage of the decorator with a custom foreign key name
for the above example is as follows:

```ts
// inside Customer model class
@hasMany(Order, {keyTo: 'custId'}) orders: Order[];
```

## Configuring a HasMany relation

Once a `HasMany` relation is defined on the source model, then there are a
couple of steps involved to configure it and use it. On the source repository,
the following are required:

- Use [Dependency Injectition](Dependency-injection.md) to inject an instance of
  the target repository in the constructor of your source repository class.
- Declare a property with the factory function type
  `HasManyRepositoryFactory<TargetModel, typeof TargetModel.prototype.id>` on
  the source repository class.
- call the `_createHasManyRepositoryFactoryFor` function in the constructor of
  the source repository class with the relation name (decorated relation propert
  on the source model) and target repository instance and assign it the property
  mentioned above.

The following code snippet shows how it would look like:

{% include code-caption.html
content="/src/repositories/customer.repository.ts.ts" %}

```ts
{import} {Order} from '../models/order.model.ts';
{import} {Customer} from '../models/customer.model.ts';
{import} {OrderRepository} from './order.repository.ts';
{import} {DefaultCrudRepository, juggler, HasManyRepositoryFactory} from '@loopback/repository';

class CustomerRepository extends DefaultCrudRepository<
  Customer,
  typeof Customer.prototype.id
> {
  public orders: HasManyRepositoryFactory<Order, typeof Order.prototype.id>;
  constructor(
    @inject('datasources.db') protected db: juggler.DataSource,
    @repository(OrderRepository) orderRepository: OrderRepository,
  ) {
    super(Customer, db);
    this.orders = this._createHasManyRepositoryFactoryFor(
      'orders',
      orderRepository,
    );
  }
}
```

The following CRUD APIs are now available in the constrained target repository
factory `orders` for instances of `customerRepository`:

- `create` for creating a target model instance
- `find` finding target model instance(s)
- `delete` for deleting target model instance(s)
- `patch` for patching target model instance(s)

## Using constrained repository in a controller

Once the has many relation has been defined and configured, it can be accessed
via controller methods to call the underlying constrained repository CRUD APIs
and exposed in LoopBack's swagger UI once decorated with
[Route decorators](Routes.md#using-route-decorators-with-controller-methods). It
will require the value of the foreign key and, depending on the request method,
a partial value for the target model instance as demonstrated below.

{% include code-caption.html
content="src/controllers/customer-orders.controller.ts" %}

```ts
import {post, param, requestBody} from '@loopback/rest';
import {customerRepository} from '../repositories/customer.repository.ts';
import {Order} from '../models/order.model.ts';

export class CustomerOrdersController {
  constructor(
    @repository(CustomerRepository)
    protected customerRepository: CustomerRepository,
  ) {}

  @post('/customers/{id}/order')
  async createCustomerOrders(
    @param.path.number('id') customerId: number,
    @requestBody() orderData: Partial<Order>,
  ): Promise<Order> {
    return await this.customerRepository.orders(customerId).create(orderData);
  }
}
```
