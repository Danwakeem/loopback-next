---
lang: en
title: 'Relations'
keywords: LoopBack 4.0, LoopBack 4
tags:
sidebar: lb4_sidebar
permalink: /doc/en/lb4/HasMany-relation.html
summary:
---

## Defining a HasMany Relation

The first relation type we have available is the `hasMany` relation which
operates by constraining the target repository by the foreign key property on
its associated model. Model relations in LoopBack 4 are defined by having a
decorator on a designated relational property. The definition of the `hasMany`
relation is inferred by using the `hasMany` decorator. The decorator takes in
the target model class constructor and optionally a custom foreign key to store
the relation metadata. The decorator logic also designates the relation type and
tries to infer the foreign key on the target model (or `keyTo` in the relation
metadata) default value (source model name appended with `id` in camel case,
also used in LoopBack 3). It also calls `property.array()` to ensure that the
type of the property is inferred properly as an array of the target model
instances. The decorated property name is used as the relation name and stored
as part of the source model definition's relation metadata. The following
example shows how to define a `hasMany` relation on a source model `Customer`.

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
  type: 2; // denotes hasMany relation (this is an enum value)
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

- Use [Dependency Injection](Dependency-injection.md) to inject an instance of
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
import {Order, Customer} from '../models';
import {OrderRepository} from './order.repository.ts';
import {
  DefaultCrudRepository,
  juggler,
  HasManyRepositoryFactory,
} from '@loopback/repository';

class CustomerRepository extends DefaultCrudRepository<
  Customer,
  typeof Customer.prototype.id
> {
  public orders: HasManyRepositoryFactory<Order, typeof Customer.prototype.id>;
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

## Using HasMany constrained repository in a controller

Once the hasMany relation has been defined and configured, controller methods
can call the underlying constrained repository CRUD APIs and expose them as
routes once decorated with
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
    @requestBody() orderData: Order, // FIXME(b-admike): should be Partial<Order> once https://github.com/strongloop/loopback-next/issues/1443 is fixed
  ): Promise<Order> {
    return await this.customerRepository.orders(customerId).create(orderData);
  }
}
```
