---
name: typescript-clean-architecture
description: Universal TypeScript clean code, strict typing, design patterns, and architecture skill. Directs the generation of production-ready, type-safe, and maintainable TypeScript code avoiding anti-patterns. Covers interface design, generics, discriminated unions, type narrowing, utility types, and strict TSConfig setups across any TS project. Triggers on "ts", "typescript", "type safety", "refactor ts", "create interface", "generics".
license: MIT
metadata:
  author: lizdev
  version: "1.0.0"
---

# ARCHITECTURE AND TYPE SYSTEM PRINCIPLES

## Rigorous Type System Design
Static type checking in TypeScript serves as a compile-time verification boundary to guarantee type safety before code execution. A rigorous type system treats types not merely as documentation, but as strict mathematical contracts that eliminate runtime exceptions. 
1. Declare contracts at the boundary: Explicitly annotate input and output types for all public interfaces, class methods, and exported functions. This prevents the compiler from performing unnecessary, expensive, and potentially inaccurate generic inferences.
2. Code to structural contracts: TypeScript's type system is structural, not nominal. An object matches a type if it satisfies its shape. Therefore, declare type contracts at the source of object literal creations rather than relying on late implicit checks at function call sites, preventing distant compile errors.

## Interfaces vs. Types Guidelines
Interfaces and type aliases are highly similar, but the official compiler specifications establish clear architectural boundaries:
1. Object Shapes and Classes: Always use `interface` declarations to define object structures, structural API contracts, and class boundaries. Interfaces are designed for open extension and support declaration merging, allowing them to be re-opened for adding new properties .
2. Compiler Performance: From a compilation speed and tracing perspective, extending interfaces via `interface extends` is faster and more performant than using type intersections (`&`). The compiler builds a single, flat, cached type representation for interfaces, detecting property conflicts early, whereas intersections recursively merge properties and must evaluate every constituent before checking against the flattened type.
3. Aliases, Unions, and Tuples: Use `type` aliases strictly for naming primitives, union types, tuple types, mapped types, or conditional type expressions. A type alias cannot be changed or re-opened for declaration merging after definition.

## Absolute Prohibitions
To prevent type-system bypasses (type leaks) at compile time, the following operations are strictly prohibited:
1. Absolute Prohibition of `any`: The `any` type completely disables type checking, acting as an implicit suppression directive. If a type is unknown or represents arbitrary input, you must use `unknown` . Values of type `unknown` must undergo explicit control flow narrowing or runtime type checks before property access.
2. Absolute Prohibition of Non-Null Assertions (`!`): Postfix non-null assertions override the compiler's safety checks without introducing runtime verifications. This introduces high risks of unhandled runtime errors. Enforce explicit truthiness narrowing, runtime assertion checks, or optional chaining (`?.`) instead of `!` .
3. Unsafe Type Casting (`as Type`): Avoid type assertions for object initialization. Creating objects using `as Type` hides missing properties, which leads to silent refactoring bugs. Use strict variable annotations (`const x: Type = ...`) instead.
4. Unsafe Unary Coercion: Do not use the unary plus (`+`) operator to convert strings to numbers. Unary plus conversions fail silently, have unexpected corner cases, and are easily missed during code review. Use `Number()` and perform explicit check validations using `isFinite()`.

---

# WORKFLOW: FROM CONCEPT TO TYPED CODE

This sequential five-step algorithm must be followed when creating new code files or refactoring existing modules:

## Step 1: Boundary Modeling
Identify the domain objects and represent their structural contracts using explicit `interface` declarations. For fields that represent alternative states, declare literal types to define finite values.
```typescript
interface UserProfile {
  readonly id: string;
  readonly name: string;
  readonly role: "admin" | "member" | "guest";
}
```

## Step 2: Contract Definition
Annotate all input parameters and explicit return types of your module's functions and class methods. Never rely on return type inference for exported functions.
```typescript
function fetchProfile(userId: string): Promise<UserProfile> {
  // implementation
}
```

## Step 3: Truthiness and Control Flow Narrowing
Apply explicit type guards (e.g., `typeof`, `instanceof`, `in`, or equality checks) to narrow union or optional properties before accessing their fields.
```typescript
function processRole(profile: UserProfile | null): string {
  if (profile === null) {
    return "No profile available";
  }
  return profile.role;
}
```

## Step 4: Generic Abstraction
When a component must operate over multiple types, abstract it using generics with structural constraints (`extends`). Keep type parameters to a minimum, and ensure each type parameter relates multiple values.
```typescript
interface Identifiable {
  readonly id: string;
}

function findById<T extends Identifiable>(items: T[], targetId: string): T | undefined {
  return items.find((item) => item.id === targetId);
}
```

## Step 5: Exhaustiveness Enforcement
Ensure complete coverage of union cases using discriminated unions and compile-time exhaustiveness checking with the `never` type.
```typescript
function handleRole(role: "admin" | "member" | "guest"): string {
  switch (role) {
    case "admin": return "Full Access";
    case "member": return "Standard Access";
    case "guest": return "Read Only";
    default: {
      const _exhaustiveCheck: never = role;
      return _exhaustiveCheck;
    }
  }
}
```

---

# ADVANCED PATTERNS AND CODE EXAMPLES

## a) Discriminated Unions for State Management
Discriminated unions group distinct structural object types under a common literal field (the discriminant), enabling safe runtime branches without unsafe boolean flags.

### INCORRECT / ANTIPATTERN
Multiple boolean flags allow invalid states (e.g., loading and error being true simultaneously) and lead to unsafe optional chaining or assertions.
```typescript
interface NetworkState {
  isLoading: boolean;
  isError: boolean;
  data?: string;
  error?: Error;
}

function handleState(state: NetworkState): string {
  if (state.isLoading) {
    return "Loading...";
  }
  if (state.isError) {
    // Unsafe: state.error could be undefined despite isError flag
    return `Error: ${state.error!.message}`;
  }
  // Unsafe: state.data could be undefined
  return `Data: ${state.data!}`;
}
```

### CORRECT / CLEAN
Clean separation of concerns with a strict literal discriminant and complete, compile-time exhaustiveness checking.
```typescript
interface LoadingState {
  readonly status: "loading";
}

interface ErrorState {
  readonly status: "error";
  readonly error: Error;
}

interface SuccessState {
  readonly status: "success";
  readonly data: string;
}

type NetworkState = LoadingState | ErrorState | SuccessState;

function handleState(state: NetworkState): string {
  switch (state.status) {
    case "loading":
      return "Loading...";
    case "error":
      return `Error: ${state.error.message}`;
    case "success":
      return `Data: ${state.data}`;
    default: {
      const _exhaustiveCheck: never = state;
      return _exhaustiveCheck;
    }
  }
}
```

## b) Generics with Constraints and Utility Types
Apply structural constraints via the `extends` keyword and leverage built-in type transformations to avoid repetitive definitions.

### INCORRECT / ANTIPATTERN
Writing unconstrained generics makes it impossible to safely access properties inside the generic function, leading to unsafe type casts or unconstrained properties.
```typescript
// Unconstrained Type parameter allows any value, including numbers which have no properties
function updateProperty<Type, Key>(obj: Type, key: Key, value: any): Type {
  // Compile error: Key cannot be used to index type Type
  obj[key] = value;
  return obj;
}
```

### CORRECT / CLEAN
Enforce type parameter mapping using `extends keyof` and leverage `Omit`, `Pick`, `Record`, or `ReturnType` to build safe transformations.
```typescript
interface UserEntity {
  id: string;
  username: string;
  email: string;
  role: string;
}

// Ensure Key is strictly a property of Type
function updateProperty<Type, Key extends keyof Type>(
  obj: Type,
  key: Key,
  value: Type[Key]
): Type {
  const updated = { ...obj, [key]: value };
  return updated;
}

// Utility Types consumption
type UserCredentials = Pick<UserEntity, "username" | "email">;
type ReadonlyUser = Readonly<UserEntity>;
type UserUpdates = Omit<UserEntity, "id">;
type UserRegistry = Record<string, UserEntity>;

// Capturing return types dynamically
declare function getRegistry(): UserRegistry;
type RegistryResponse = ReturnType<typeof getRegistry>;
```

## c) Custom Type Guards and Type Predicates
Custom type guards use type predicates to enforce narrow and verified type scopes down nested paths.

### INCORRECT / ANTIPATTERN
Inline type assertions or unsafe conditional checks force developers to repeatedly assert types down call stacks, masking runtime exceptions.
```typescript
interface AdminUser {
  id: string;
  permissions: string[];
}

function processUser(user: unknown) {
  // Unsafe: checking field existence without narrowing leaves type as unknown
  if (user && typeof user === "object" && "permissions" in user) {
    // Forced unsafe casting
    const admin = user as unknown as AdminUser;
    console.log(admin.permissions.join(", "));
  }
}
```

### CORRECT / CLEAN
Isolate runtime safety checks within a structured type guard function returning a strict `parameterName is Type` predicate.
```typescript
interface AdminUser {
  readonly id: string;
  readonly permissions: readonly string[];
}

// Custom Type Guard with type predicate
function isAdminUser(user: unknown): user is AdminUser {
  if (user === null || typeof user !== "object") {
    return false;
  }
  
  const candidate = user as Record<string, unknown>;
  return (
    typeof candidate["id"] === "string" &&
    Array.isArray(candidate["permissions"]) &&
    candidate["permissions"].every((perm) => typeof perm === "string")
  );
}

function processUser(user: unknown): void {
  if (isAdminUser(user)) {
    // Type is successfully narrowed to AdminUser
    console.log(user.permissions.map((p) => p.toUpperCase()));
  }
}
```

## d) Replacing Enums with const Assertions or String Literal Unions
Standard TypeScript `enum` and `const enum` structures generate runtime footprints or suffer from critical transpilation issues under isolated modules compiling. String literal unions or constant object assertions are the cleanest alternatives.

### INCORRECT / ANTIPATTERN
Traditional numeric or heterogeneous enums generate complex reverse-mapped objects at runtime and introduce unsafe boolean coercion behavior (where value 0 is falsy).
```typescript
enum UserRole {
  Admin,  // 0 (falsy!)
  Member, // 1
  Guest   // 2
}

function checkAccess(role: UserRole): boolean {
  // Bug-prone: if role is Admin (0), Boolean(role) or implicit coercion evaluates to false
  if (!role) {
    return false;
  }
  return true;
}
```

### CORRECT / CLEAN
String literal unions and `as const` assertions provide zero runtime overhead, preserve absolute literal type safety, and support isolated transpilation.
```typescript
// Pattern A: String Literal Union
type AppRole = "admin" | "member" | "guest";

function checkAccess(role: AppRole): boolean {
  return role !== "guest";
}

// Pattern B: Constant Object Assertion (as const)
const APP_ROLES = {
  ADMIN: "admin",
  MEMBER: "member",
  GUEST: "guest",
} as const;

type AppRoleType = typeof APP_ROLES[keyof typeof APP_ROLES];

function handleConstantRole(role: AppRoleType): string {
  if (role === APP_ROLES.ADMIN) {
    return "Administrator Access";
  }
  return "Standard Access";
}
```

---

# ANTI-PATTERNS AND PREVENTION CHECKLIST

Avoid these common compilation and code structure anti-patterns:

## 1. Boxed Primitives
* Anti-pattern: Instantiating or typing with primitive wrapper objects `String`, `Number`, `Boolean`, `Symbol`, or `Object`. Boxed wrappers have different meanings from lowercased primitives and exhibit surprising behavior.
* Corrective action: Always use lowercased primitives `string`, `number`, `boolean`, `symbol`, and `object`.
```typescript
/* INCORRECT */
const key: String = new String("id");

/* CORRECT */
const key: string = "id";
```

## 2. Callback Return Any
* Anti-pattern: Typing the return value of a callback as `any` when its return value is meant to be ignored. This leaves returned objects completely unchecked.
* Corrective action: Use the `void` return type, which safely prevents accidental usage of discarded return values.
```typescript
/* INCORRECT */
function execCallback(cb: () => any): void { cb(); }

/* CORRECT */
function execCallback(cb: () => void): void { cb(); }
```

## 3. Optional Callback Parameters
* Anti-pattern: Declaring optional parameters inside callback definitions when they might not be passed. This forces the callback implementation to handle `undefined` values unnecessarily.
* Corrective action: Declare callback parameters as required. JavaScript always permits passing a callback that accepts fewer arguments.
```typescript
/* INCORRECT */
function fetchItem(cb: (data: unknown, delay?: number) => void): void { cb({}, 100); }

/* CORRECT */
function fetchItem(cb: (data: unknown, delay: number) => void): void { cb({}, 100); }
```

## 4. General Overloads First
* Anti-pattern: Declaring general, catch-all overloads before specific ones. TypeScript selects the first matching signature, which hides more specific signatures.
* Corrective action: Always order overloads from most specific to most general. Better yet, prefer union parameters over function overloads when possible.
```typescript
/* INCORRECT */
declare function parseValue(x: unknown): unknown;
declare function parseValue(x: string): number;

/* CORRECT */
declare function parseValue(x: string): number;
declare function parseValue(x: unknown): unknown;
```

## 5. Unfiltered for...in Object Iteration
* Anti-pattern: Using unfiltered `for...in` loops to iterate over object keys. `for...in` includes inherited enumerable properties from prototype chains.
* Corrective action: Use `for...of` with `Object.keys()`, `Object.values()`, or `Object.entries()` to guarantee only own properties are iterated.
```typescript
/* INCORRECT */
for (const key in obj) { console.log(obj[key]); }

/* CORRECT */
for (const [key, value] of Object.entries(obj)) { console.log(key, value); }
```

---

# UNIVERSAL TSCONFIG PRESET

This is the production-ready `tsconfig.json` template. It enables strict typing constraints to block runtime type leaks:

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    /* Target Environment */
    "target": "es2022",                          /* Support modern runtime features and standard APIs */
    "module": "nodenext",                        /* Support modern module compilation matching Node.JS ESM */
    "moduleResolution": "nodenext",              /* Resolve imports based on Node.JS package.json export conditions */
    
    /* Strict Type-Checking Options */
    "strict": true,                              /* Enable all strict type-checking flags at once */
    "noImplicitAny": true,                       /* Error on expressions and declarations with implied 'any' type */
    "strictNullChecks": true,                    /* Null and undefined get their own distinct types, blocking null crashes */
    "strictFunctionTypes": true,                 /* Enforce contravariant checking for function parameters */
    "strictPropertyInitialization": true,        /* Ensure class fields are initialized inside constructors */
    "noImplicitThis": true,                      /* Error on 'this' expressions with an implied 'any' type */
    "useUnknownInCatchVariables": true,          /* Set variables inside catch blocks to 'unknown' instead of 'any' */

    /* Additional Architecture constraints */
    "exactOptionalPropertyTypes": true,          /* Disallow assigning 'undefined' to optional fields explicitly */
    "noUncheckedIndexedAccess": true,            /* Add 'undefined' to index signatures lookup to force safety checks */
    "noPropertyAccessFromIndexSignature": true,  /* Force using index accessors (obj["key"]) for index-declared fields */
    
    /* Code Cleanliness Checks */
    "noUnusedLocals": true,                      /* Error when local variables are declared but never read */
    "noUnusedParameters": true,                  /* Error when function parameters are declared but never read */
    "noFallthroughCasesInSwitch": true,          /* Error when non-empty case branches fall through without break/return */
    "noImplicitReturns": true,                   /* Ensure all code paths inside a function return a value */

    /* Interop & Compiling Settings */
    "esModuleInterop": true,                     /* Emit helper shims for CommonJS and ESM module interop */
    "forceConsistentCasingInFileNames": true,    /* Enforce identical casing imports to avoid case-insensitive FS bugs */
    "isolatedModules": true,                     /* Enforce code safety under single-file transpilation tools */
    "verbatimModuleSyntax": true,                /* Preserve standard import/export formatting, dropping only type-only statements */
    
    /* Optimization & Emit Configuration */
    "noEmit": true,                              /* Delegate transpilation to build tools, using tsc only for type checking */
    "skipLibCheck": true,                        /* Skip full type checking of declaration files (.d.ts) to speed up compilation */
    "incremental": true                          /* Enable incremental compilations via cache files */
  },
  "include": ["src/**/*"],
  "exclude": ["**/node_modules", "**/.*/"]       /* Exclude dependency directories and hidden dot directories */
}
```