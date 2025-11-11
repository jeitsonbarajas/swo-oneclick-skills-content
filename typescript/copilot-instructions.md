# TypeScript GitHub Copilot Instructions

**Generated:** 2024-11-09  
**Version:** 1.0  
**Technology:** TypeScript  
**Author:** SWO Team

## Overview
These instructions configure GitHub Copilot to generate TypeScript code with strict typing, modern patterns, and enterprise-grade quality standards.

## TypeScript Configuration

### tsconfig.json Recommendations
```json
{
  "compilerOptions": {
    "target": "ES2020",
    "module": "commonjs",
    "lib": ["ES2020"],
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true
  }
}
```

## Best Practices

### Type Definitions

#### Interfaces vs Types
```typescript
// Use interfaces for object shapes (extensible)
interface User {
  id: string;
  name: string;
  email: string;
}

interface Admin extends User {
  role: 'admin';
  permissions: string[];
}

// Use types for unions, intersections, utilities
type Status = 'pending' | 'active' | 'inactive';
type Result<T> = { success: true; data: T } | { success: false; error: string };
type ReadonlyUser = Readonly<User>;
```

#### Generics
```typescript
// Generic function with constraints
function findById<T extends { id: string }>(items: T[], id: string): T | undefined {
  return items.find(item => item.id === id);
}

// Generic class
class DataStore<T> {
  private items: Map<string, T> = new Map();

  set(key: string, value: T): void {
    this.items.set(key, value);
  }

  get(key: string): T | undefined {
    return this.items.get(key);
  }

  getAll(): T[] {
    return Array.from(this.items.values());
  }
}
```

### Strict Null Checks

```typescript
// Always handle null/undefined
function getUser(id: string): User | null {
  const user = database.findUser(id);
  return user ?? null;
}

// Use optional chaining
const email = user?.profile?.email;

// Use nullish coalescing
const displayName = user?.name ?? 'Anonymous';

// Type guards
function isAdmin(user: User | Admin): user is Admin {
  return 'role' in user && user.role === 'admin';
}

if (isAdmin(user)) {
  // TypeScript knows user is Admin here
  console.log(user.permissions);
}
```

### Async/Await with Types

```typescript
// Typed async function
async function fetchUserData(userId: string): Promise<User> {
  try {
    const response = await fetch(`/api/users/${userId}`);
    
    if (!response.ok) {
      throw new Error(`HTTP error! status: ${response.status}`);
    }
    
    const data = await response.json();
    return data as User;
  } catch (error) {
    if (error instanceof Error) {
      throw new Error(`Failed to fetch user: ${error.message}`);
    }
    throw error;
  }
}

// Generic async result type
type AsyncResult<T, E = Error> = Promise<
  | { success: true; data: T }
  | { success: false; error: E }
>;

async function safeUserFetch(userId: string): AsyncResult<User> {
  try {
    const user = await fetchUserData(userId);
    return { success: true, data: user };
  } catch (error) {
    return { 
      success: false, 
      error: error instanceof Error ? error : new Error('Unknown error') 
    };
  }
}
```

### Utility Types

```typescript
// Partial - make all properties optional
type PartialUser = Partial<User>;

// Required - make all properties required
type RequiredConfig = Required<Config>;

// Pick - select specific properties
type UserPreview = Pick<User, 'id' | 'name'>;

// Omit - exclude specific properties
type UserWithoutEmail = Omit<User, 'email'>;

// Record - typed object
type UserMap = Record<string, User>;

// ReturnType - extract return type
type UserResult = ReturnType<typeof fetchUserData>;

// Parameters - extract parameters
type FetchParams = Parameters<typeof fetchUserData>;
```

### Enums vs Const Objects

```typescript
// Numeric enum (avoid if possible)
enum Direction {
  Up = 1,
  Down,
  Left,
  Right
}

// String enum (better)
enum LogLevel {
  Error = 'ERROR',
  Warn = 'WARN',
  Info = 'INFO',
  Debug = 'DEBUG'
}

// Const object (recommended alternative)
const LogLevel = {
  Error: 'ERROR',
  Warn: 'WARN',
  Info: 'INFO',
  Debug: 'DEBUG'
} as const;

type LogLevel = typeof LogLevel[keyof typeof LogLevel];
```

### Decorators (if enabled)

```typescript
// Class decorator
function sealed(constructor: Function) {
  Object.seal(constructor);
  Object.seal(constructor.prototype);
}

// Method decorator
function log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;
  
  descriptor.value = function(...args: any[]) {
    console.log(`Calling ${propertyKey} with`, args);
    const result = originalMethod.apply(this, args);
    console.log(`Result:`, result);
    return result;
  };
  
  return descriptor;
}

@sealed
class UserService {
  @log
  async getUser(id: string): Promise<User> {
    return fetchUserData(id);
  }
}
```

## Project Structure

```
src/
├── types/
│   ├── index.ts         # Exported type definitions
│   ├── models.ts        # Domain models
│   └── api.ts           # API types
├── services/
│   └── userService.ts   # Business logic
├── utils/
│   ├── validators.ts    # Type guards & validation
│   └── helpers.ts       # Utility functions
├── config/
│   └── constants.ts     # Type-safe constants
└── index.ts             # Entry point
```

## Error Handling

```typescript
// Custom error classes
class ValidationError extends Error {
  constructor(
    message: string,
    public field: string,
    public value: unknown
  ) {
    super(message);
    this.name = 'ValidationError';
  }
}

class NotFoundError extends Error {
  constructor(
    message: string,
    public resource: string,
    public id: string
  ) {
    super(message);
    this.name = 'NotFoundError';
  }
}

// Type-safe error handling
function handleError(error: unknown): string {
  if (error instanceof ValidationError) {
    return `Validation failed for ${error.field}: ${error.message}`;
  }
  
  if (error instanceof NotFoundError) {
    return `${error.resource} with ID ${error.id} not found`;
  }
  
  if (error instanceof Error) {
    return error.message;
  }
  
  return 'An unknown error occurred';
}
```

## Testing with Types

```typescript
import { describe, it, expect } from '@jest/globals';

describe('UserService', () => {
  it('should fetch user by id', async () => {
    const userId = '123';
    const expectedUser: User = {
      id: userId,
      name: 'John Doe',
      email: 'john@example.com'
    };

    const user = await userService.getUser(userId);
    
    expect(user).toEqual(expectedUser);
  });

  it('should handle fetch errors', async () => {
    const invalidId = 'invalid';
    
    await expect(userService.getUser(invalidId))
      .rejects
      .toThrow(NotFoundError);
  });
});

// Type-safe mocks
type MockFunction<T extends (...args: any[]) => any> = jest.Mock<
  ReturnType<T>,
  Parameters<T>
>;

const mockFetch: MockFunction<typeof fetch> = jest.fn();
```

## Comments & Documentation

```typescript
/**
 * Fetches user data from the API
 * 
 * @param userId - The unique identifier of the user
 * @returns Promise resolving to User object
 * @throws {NotFoundError} When user doesn't exist
 * @throws {ValidationError} When userId is invalid
 * 
 * @example
 * ```typescript
 * const user = await fetchUserData('123');
 * console.log(user.name);
 * ```
 */
async function fetchUserData(userId: string): Promise<User> {
  // Implementation
}
```

## Performance Tips

1. Use `const` assertions for literal types
2. Avoid excessive type assertions (`as`)
3. Prefer type inference over explicit types
4. Use discriminated unions for complex states
5. Enable `skipLibCheck` for faster compilation
6. Use project references for monorepos

---

*This configuration is part of SWO OneClick Skills - Created by SoftwareOne Team*
