# Use Case: Create Coupon Book

Creating a coupon book can vary based on the given optional parameters. Below
we outline the different scenarios.

## Input

The endpoint will accept the following parameters:

```json
{
  "bookName": "summer_discount",
  "maxCodesPerUser": 5,
  "maxRedeemCountPerUser": 3
}
```

## Logic

```typescript
// Received parameter
function handler(request) {
  const { bookName, maxCodesPerUser, maxRedeemCountPerUser } = request.body;

  const book = new CouponBook({
    name: bookName,
    maxCodesPerUser: maxCodesPerUser,
    maxRedeemCountPerUser: maxRedeemCountPerUser
  });

  const addedBook = await book.save(tx);
  return {
    bookId: addedBook.id
  }
}
```

## Output

```json
{
  "bookId": "1234567890",
}
```
