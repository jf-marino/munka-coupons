# Use case: Redeeming a code

This is the final operation on a code. In order to redeem a code the user must
have the code already assigned to them and the code must have an active lock.

---

## Request

```http
POST /coupons/redeem/:code
Content-Type: application/json

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

---

## Pseudo Code

```typescript
async function handler(request) {
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

      if (coupon.redeemedCount >= coupon.maxRedeemCount)
        throw new BadRequest({
          message: 'Coupon has reached its maximum redeem count per user'
        })

      const redeemedDate = new Date()
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
