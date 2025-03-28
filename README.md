# Coupons

Code challenge solution for coupon generation service.
The solution includes a set of use cases, each described in its own file.
Each one outlines a feature of the service, and includes API endpoint
(request/response) definitions as well as pseudo code describing
the internal logic for that feature.

Challenge rules and spec can be found [**here**](challenge.pdf).

## Use Cases

1. [**Creating a book**](cases/create_book.md)
2. [**Generate or upload codes**](cases/add_codes.md)
3. [**Assigning codes**](cases/assign_code.md)
4. [**Locking codes**](cases/lock_code.md)
5. [**Unlocking codes**](cases/unlock_codes.md)
6. [**Redeeming codes**](cases/redeem_code.md)

## Assumptions

This design assumes that admin users will be authenticated using JWT tokens.
However, it is not defined anywhere where these tokens are generated or
validated. Some operations will require authentication and authorization
(for example, creating a book) and others will be public (for example,
redeeming a code). In the cases where an operation requires auth an
`Authorization` header will be defined in the request of that operation.

## Database

The following ERD describes the shape of the data that the use cases assume.

![Database ERD](erd.png)

## Infrastructure

In order to deploy the service for scalability we should follow the architecture shown below.
