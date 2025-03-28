# Use case: List a user's codes

Provide a list of codes for a given user.

---

## Request

```http
GET /coupon/:bookId?email=example@example.com
Authorization: Bearer <token>
```

## Responses

### **200 - Returned a list of codes**

```json
{
  "bookId": "123456",
  "codes": [
    {
      "code": "ABC123",
    },
    {
      "code": "XYZ456",
    }
  ]
}
```

### **401 - Unauthorized**

```json
{
  "error": "Unauthorized"
}
```

### **404 - Not Found**

Returned if the user does not exist in the book.

```json
{
  "error": "Not Found"
}
```
