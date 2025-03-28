# Coupons

Code challenge solution for coupon generation service.
The solution includes a set of use cases, each described in its own file.
Each one outlines a feature of the service, and includes API endpoint
(request/response) definitions as well as pseudo code describing
the internal logic for that feature.

[**Problem description**](challenge.pdf)

## Use Cases

1. [**Creating a book**](cases/create_book.md)
2. [**Generate or upload codes**](cases/add_codes.md)
3. [**Assigning codes**](cases/assign_code.md)

## Database

The following ERD describes the shape of the data that the use cases assume.

## Infrastructure

In order to deploy the service for scalability we should follow the architecture shown below.
