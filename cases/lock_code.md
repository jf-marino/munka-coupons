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
Authorization: Bearer <token>

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

### **400 - Coupon book not found**

This error is returned if the coupon book is not found.

```json
{
  "error": "Coupon book not found"
}
```

---

## Pseudo Code

```typescript
const DEFAULT_LOCK_DURATION = 10 * 60 * 1000;

async function handler(request) {
  const { partnerId } = request.jwt;
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

      const book = await tx.findOne(CouponBook, {
        where: { id: coupon.bookId, partnerId: partnerId },
        // In Postgres this will translate to a FOR SHARE lock, which
        // will allow concurrent redeem operations to proceed without blocking.
        // If we used a PESSIMISTIC_WRITE lock, it would block other
        // transactions from locking or redeeming codes, even though
        // the value they're locking in is not being modified.
        lock: { mode: LockMode.PESSIMISTIC_READ }
      });

      if (!book)
        throw new BadRequest({
          error: 'Coupon book not found'
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
