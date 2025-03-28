# Use Case: Add Codes to a Book

Adding codes to a book can be done in two ways:

1. **Manual Entry**: You can manually enter codes for each book. This is useful when you have a small number of codes or when you want to ensure that each code is unique.

2. **Automatic Generation**: You can automatically generate a set of codes for each book.

Both are supported at the same time, meaning in a single request we could
auto-generate 100 codes and add another 100 codes manually. However, some
assumptions must be taken into account when mixing both:

1. Manually entered codes will be committed first, and auto generated codes after. This ensures that the codes are unique and that the auto-generated codes do not collide with the manually entered codes.
2. If the manually entered code fails (because one or more had collisions), the entire request will fail.

---

## Request

```http
POST /coupons/codes
Content-Type: application/json
Authorization: Bearer <token>

{
  "bookId": "...",
  "generated": {
    "amount": 100,
    "prefix": "sd2025",
    "codeLength": 4,
    "expiration": "2025-12-31T23:59:59Z",
  },
  "manual": [
    {
      "code": "sd2025-0001",
      "expiration": "2025-12-31T23:59:59Z"
    },
    {
      "code": "sd2025-0002",
      "expiration": "2025-12-31T23:59:59Z",
    },
    ...
  ]
}
```

## Responses

### **201 - Codes added**

```json
{
  "codes": [
    {
      "code": "sd2025-0001",
      "expiration": "2025-12-31T23:59:59Z"
    },
    {
      "code": "sd2025-0002",
      "expiration": "2025-12-31T23:59:59Z",
    },
    ...
  ]
}
```

### **400 - Book not found**

Happens when the caller tries to add codes to a book that does not exist.

```json
{
  "error": "Book not found"
}
```

### **400 - Failed to generate {amount} random new codes**

Happens when the caller tries to generate {amount} of random
new codes but, after attempting multiple times the system is
unable to generate enough new codes. This will have higher
likelihood of happening when the length of the code is too
short for the amount of codes requested in the book.

```json
{
  "error": "Failed to generate 100 random new codes"
}
```

---

## Pseudo Code

Below is the pseudo code for the request:

```typescript
async function handler(request) {
  const { partnerId } = request.jwt;
  const { bookId, generated, manual } = request.body;

  const book = await manager.findOne(CouponBook, {
    where: {
      id: bookId,
      partnerId
    }
  });

  if (!book) {
    throw new BadRequest({ error: 'Book not found' });
  }

  let createdCodes = [];
  try {
    if (manual) {
      createdCodes = await manager.transaction(async tx => {
        return await tx.save(manual.codes.map(code => tx.create(CouponCode, {
          id: code.code,
          expiration: code.expiration,
          bookId
        })))
      })
    }

    if (generated) {
      // Method `autoGenerateCodes` defined in a section below.
      const generatedCodes = await autoGenerateCodes({
        bookId,
        amount: generated.amount,
        prefix: generated.prefix,
        codeLength: generated.codeLength,
        expiration: generated.expiration,
      });

      createdCodes = [...createdCodes, ...generatedCodes];
    }

    return {
      codes: createdCodes.map(code => ({
        code: code.id,
        expiration: code.expiration
      }))
    }
  } catch (error) {
    if (error instanceof BadRequest) throw error;
    console.error(error);
    // This is probably not the best approach, as the failure
    // could be expected if its because of a code collision when
    // adding codes manually, for instance.
    throw new ServerError(error.message);
  }
}
```

### Automatic code generation logic:

```typescript
const DEFAULT_GENERATED_CODE_LENGTH = 4;
const MAX_GENERATION_ATTEMPTS = 5;

function autoGenerateCodes(params) {
  const {
    bookId,
    amount,
    prefix,
    codeLength = DEFAULT_GENERATED_CODE_LENGTH,
    expiration = null
  } = params;

  // Attempt to generate the requested amount of random new codes
  // up to MAX_GENERATION_ATTEMPTS attempts.
  const createdCodes = await manager.transaction(async tx => {
    let attempts = 0;
    let codes = [];
    do {
      let codes = range(0, amount - codes.length).map(i => ({
        code: `${prefix ? prefix + '-' : ''}${generateCode(codeLength)}`,
        expiration,
      }));

      const existingCodes = await tx.find(CouponCode, {
        where: { bookId, id: In(codes) }
      });

      codes = codes.filter(code => {
        return !existingCodes.some(existingCode =>
          existingCode.code === code.code);
      });

      attempts++;
    } while (codes.length < amount && attempts < MAX_GENERATION_ATTEMPTS);

    if (codes.length < amount)
      throw new BadRequest({
        error: `Failed to generate ${amount} random new codes`
      });

    return await tx.save(codes.map(code => new CouponCode({
      id: code,
      bookId,
      expiration
    })));
  })
}

const generateCode = (length: number) => {
  // This alphabet should allow (48^length) code permutations. For a
  // default of 4 characters, there are 48^4 = 5,308,416 possible codes.
  const alphabet = 'ABCDEFGHJKLMNPQRSTUVWXYZabcdefghjklmnpqrstuvwxyz';
  return Array.from({ length }, () => {
    return chars[Math.floor(Math.random() * chars.length)]
  }).join('');
}

```
