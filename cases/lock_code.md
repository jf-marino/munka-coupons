# Use case: Locking a coupon code

When a user attempts to redeem the code, possibly through a checkout process,
we will lock the code to prevent further use. This prevents a double spend
attack, where a user attempts to use the same code multiple times. The code will be locked for 10min by default. If the code is redeemed within
the lock period, it will be unlocked and can be used again.

In order to lock a coupon the user must have the coupon assigned to them.

---

## Request

```http
POST /coupons/lock/:code
Content-Type: application/json

{
  "email": "user@example.com"
}
```

## Responses

### **200 - Coupon locked**

```json
{
  "success": true,
  "lockedUntil": "2023-03-01T12:00:00Z"
}
```

### **404 - User has no coupon with that code assigned**

```json
{
  "error": "User has no coupon with that code assigned"
}
```

### **400 - Coupon has reached its maximum redeem count per user**

```json
{
  "error": "Coupon has reached its maximum redeem count per user"
}
```

### **400 - Cannot lock coupon**

This will be returned if the coupon is assigned to the user but it is
already locked.

```json
{
  "error": "Cannot lock coupon"
}
```

### **400 - Failed to lock coupon**

This error is generic and it may happen when mutliple
lock attempts are made concurrently.

```json
{
  "error": "Failed to lock coupon"
}
```

---

## Pseudo Code

```typescript
const DEFAULT_LOCK_DURATION = 10 * 60 * 1000;

async function handler(request) {
  const { code } = request.params;
  const { email } = request.body;

  try {
    const lockedCode = await manager.transaction(async tx => {
      const coupon = await tx.findOne(CouponCode, {
        where: { id: code, assignedTo: email }
      })

      if (!coupon)
        throw new NotFound({
          error: 'User has no coupon with that code assigned'
        });

      // Here we are not locking on the book, even though we probably should.
      // This is because we only need to read the max redeem count per user,
      // If the value of the redeem count changes while the transaction is
      // happening, we might allow a code to be locked more times than allowed.
      // We will assume it won't change often (there is no operation for it in
      // the API at the moment in fact). This is simply to avoid
      // paying the performance cost of locking the book on every code
      // lock operation when most of the time the value is the same.
      const book = await tx.findOne(CouponBook, {
        where: { id: coupon.bookId }
      });

      if (book.maxRedeemCountPerUser && book.maxRedeemCountPerUser <= coupon.redeemedCount)
        throw new BadRequest({
          error: 'Coupon has reached its maximum redeem count per user'
        });

      if (coupon.lockedUntil && coupon.lockedUntil > Date.now())
        throw new BadRequest({
          // This message is intentionally vague to
          // prevent information leakage.
          error: 'Cannot lock coupon'
        });

      // Set the lock timer to 10min.
      // This assumes server time is UTC. A more robust approach would
      // be to use something like luxon or moment-tz to ensure UTC is used.
      const lockedUntil = new Date(Date.now() + DEFAULT_LOCK_DURATION);
      return await tx.updateOne(CouponCode, { id: code }, { lockedUntil });
    })

    return { success: true, lockedUntil: lockedCode.lockedUntil }
  } catch (error) {
    if (error instanceof BadRequest) throw error
    if (error instanceof NotFound) throw error
    throw new BadRequest({
      message: 'Failed to lock coupon'
    })
  }
}
```
