# Use case: Unlock coupons with expired locks

If a coupon has been locked for more than 10min and hasn't been redeemed yet,
we can remove the lock on it to allow users to use it again. This process
will likely be triggered by a background job or a scheduled task.

---

## Pseudo Code

```typescript
const DEFAULT_LOCK_DURATION = 10 * 60 * 1000;

async function handler() {
  await manager.transaction(async tx => {
    const expiredLockCoupons = await tx.find(CouponCode, {
      where: {
        lockedUntil: LessThan(Date.now()),
      }
    })

    await tx.update(CouponCode, {
      lockedUntil: null,
    }, {
      where: {
        id: In(expiredLockCoupons.map(c => c.id)),
      }
    })
  })
}
```
