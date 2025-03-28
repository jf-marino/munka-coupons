# Use case: Redeeming a code

This is the final operation on a code. In order to redeem a code the user must
have the code already assigned to them and the code must have an active lock.

---

## Request

```http
POST /coupons/redeem/:code
Content-Type: application/json
Authorization: Bearer <token>

{
  "email": "user@example.com"
}
```

## Responses

### **200 - Code redeemed successfully**

```json
{
  "success": true
}
```

### **400 - Failed to redeem code**

This error is generic and it may happen when the code is redeemed concurrently.

```json
{
  "error": "Failed to redeem code"
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

### **400 - Coupon book not found**

```json
{
  "error": "Coupon book not found"
}
```

---

## Pseudo Code

```typescript
async function handler(request) {
  const { partnerId } = request.jwt
  const { code } = request.params
  const { email } = request.body

  try {
    await manager.transaction(async tx => {
      const coupon = await tx.findOne(CouponCode, {
        where: {
          id: code,
          assignedTo: email,
          lockedUntil: LessThan(Date.now())
        }
      })

      if (!coupon)
        throw new NotFound({
          message: 'User has no coupon with that code assigned'
        })

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
          message: 'Coupon book not found'
        })

      if (coupon.redeemedCount >= coupon.maxRedeemCount)
        throw new BadRequest({
          message: 'Coupon has reached its maximum redeem count per user'
        })

      const redeemedDate = new Date()
      // The log is not really required, however, it is useful in real
      // world applications to track the history of coupon usage as well as
      // to provide a record of the redemption for auditing purposes.
      const log = new CouponRedeemLog({
        couponId: coupon.id,
        redeemedOn: redeemedDate
      })

      await tx.save(log)
      await tx.update(
        CouponCode,
        { id: coupon.id },
        {
          redeemedCount: coupon.redeemedCount + 1,
          lastRedeemedOn: redeemedDate,
          lockedUntil: null
        }
      )
    })

    return {
      success: true
    }
  } catch (error) {
    if (error instanceof BadRequest)    throw error
    if (error instanceof NotFound)    throw error
    throw new BadRequest({
      message: 'Failed to redeem coupon'
    })
  }
}
```
