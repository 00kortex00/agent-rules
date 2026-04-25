---
id: general-security
category: general
tags: [security, secrets, validation, dependencies]
description: Basic security rules
---

# Security

## Secrets and environment variables

Never hardcode secrets. Always use environment variables:

```ts
// ❌
const db = new Database("postgresql://user:password@localhost/mydb")

// ✅
const db = new Database(process.env.DATABASE_URL)
```

- Add `.env` to `.gitignore`
- Keep `.env.example` with all keys but no real values
- Validate required variables at startup:

```ts
// config.ts
const required = ["DATABASE_URL", "JWT_SECRET", "API_KEY"]

for (const key of required) {
  if (!process.env[key]) {
    throw new Error(`Missing required env variable: ${key}`)
  }
}

export const config = {
  databaseUrl: process.env.DATABASE_URL!,
  jwtSecret: process.env.JWT_SECRET!,
}
```

---

## Input validation

Always validate data at the boundary — user input, external APIs, database results. Use [TypeBox](https://github.com/sinclairzx81/typebox):

```ts
import { Type } from "@sinclair/typebox"
import { Value } from "@sinclair/typebox/value"

const CreateUserSchema = Type.Object({
  email: Type.String({ format: "email" }),
  password: Type.String({ minLength: 8 }),
  name: Type.String({ minLength: 1, maxLength: 100 }),
})

export const createUser = async (input: unknown) => {
  const data = Value.Parse(CreateUserSchema, input)
  // data is now typed and validated
}
```

---

## Dependencies

- Do not add dependencies without a clear reason
- Before installing a package check: weekly downloads, maintenance activity, last publish date
- Keep dependencies up to date: `bun update`
- Do not ignore `bun audit` warnings
