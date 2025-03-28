# Use Case: Assign a Code to a Book

Assigns an existing code to a user. **Users are identified by their email address**.

The assignment can be done in one of two ways:

1. Random code assignment. Given a book, the system will find one random, still unassigned code and will assign it to the user.
2. Specific code assignment. The caller specifies a code and the system will assign it to the user, if possible.

In both cases, in order for the system to assign a code to a user, the user must not exceed the maximum number of assigned codes allowed within the book.

---

## Request

The parameter `code` is optional. If not provided, a random code will be assigned.

```http
POST /coupons/assign
content-type: application/json

{
  "bookId": "b123",
  "email": "user@example.com",
  "code": "ABC123"
}
```

## Responses

### **200 - Successfully assigned**

The code has been assigned successfully.

```json
{
  "assigned": true,
  "code": "ABC123"
}
```

### **404 - The code does not exist**

Used only when the code to assign is explicitly specified by the caller. If
the code is not found, the system will return a 404 error.

### **400 - User already has maximum number of assigned codes**

User already has maximum number of assigned codes in the book.

```json
{
  "error": "User already has maximum number of assigned codes in the book."
}
```

### **400 - No available codes left**

No available codes left in the book. This may be returned when the caller
tries to assign a random code.

```json
{
  "error": "No available codes left in the book."
}
```

### **400 - Code already assigned**

The code has already been assigned to another user.

```json
{
  "error": "Code already assigned"
}
```

---

## Pseudo Code

```typescript
async function handler(request) {
  const { bookId, email, code } = request.body;

  await manager.transaction(async tx => {
    const book = tx.find(CouponBook, { where: { id: bookId }})
    if (!book) throw new BadRequest({ error: 'Book not found' })

    const codesForUser = await tx.find(CouponCode, {
      where: { bookId, userId: email }
    })

    if (codesForUser.length >= book.maxCodesPerUser)
      throw new BadRequest({
        error: 'User already has maximum number of assigned codes in the book.'
      })

    if (code) {
      const selectedCode = await tx.find(CouponCode, {
        where: { bookId, code }
      })
      if (!selectedCode) throw new NotFound()
      if (selectedCode.assignedTo)
        throw new BadRequest({ error: 'Code already assigned' })
      selectedCode.assignedTo = email
      return tx.save(selectedCode)
    }

    // ------ [ random assignment below ] -------

    const freeCodes = await tx.find(CouponCode, {
      where: { bookId, assignedTo: IsNull() }
    })

    if (!freeCodes.length)
      throw new BadRequest({
        error: 'No available codes left in the book.'
      })

    // Pick a random code from the list of available codes.
    // This operation may still fail if two or more users try
    // to assign the same code at the same time.
    const randomPick = Math.floor(
      Math.random() * freeCodes.length
    )
    const codeToAssign = freeCodes[randomPick]
    codeToAssign.assignedTo = email

    const assignedCode = await tx.save(codeToAssign)

    return { assigned: true, code: assignedCode.code }
  })
}
```
