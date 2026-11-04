# SQL Injection in WHERE clause: retrieving hidden data

| | |
|---|---|
| **Platform** | PortSwigger Web Security Academy |
| **Category** | SQL Injection |
| **Difficulty** | Apprentice |
| **Status** | Solved |

## Objective

Bypass a product category filter that hides unreleased items, using SQL injection in a query parameter.

## Reconnaissance

- Identified the `category` parameter as the likely injection point.
- Tested reflected input to see whether it reached the database. A `<script>` payload was printed as plain text, so the vector was server-side and not XSS.
- Noted that the parameter value is wrapped in a string literal, which determines how the payload has to be closed.

## Reasoning

- The query filters rows with a `WHERE` clause that checks the category and hides unreleased items.
- Closing the string literal with a quote lets me append my own condition to the `WHERE` clause.
- A condition that is always true makes the filter stop excluding rows, and a comment sequence drops the rest of the original query.

## Impact

An attacker could bypass visibility restrictions. In a real application, the same injection point could be extended with `UNION SELECT` to read data from other tables.

## Mitigation

- Use parameterized queries or prepared statements instead of concatenating input into SQL.
- Validate input on the server as defense in depth.

## Lesson learned

- Always check how user input is quoted in the query before choosing a payload.
- Fixing the injection point matters more than memorizing any specific payload.
