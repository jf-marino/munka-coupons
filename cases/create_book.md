# Use Case: Create Coupon Book

Creating a coupon book can vary based on the given optional parameters. Below
we outline the different scenarios.

---

## Request

```http
POST /coupons
Content-Type: application/json
Authorization: Bearer <token>

{
  "bookName": "summer_discount",
  "maxCodesPerUser": 5,
  "maxRedeemCountPerUser": 3
}
```

## Responses

### **201 - Book created successfully**

```json
{
  "bookId": "1234567890",
}
```

---

## Pseudo Code

```typescript
async function handler(request) {
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
