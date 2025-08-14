# Authentication & Authorization System Implementation Plan

## Feature Overview

Implement a comprehensive authentication and authorization system using NextAuth with email/password provider, supporting multiple user roles and protecting API routes.

## Objectives

- Secure user authentication with email/password
- Role-based access control (Admin, Organizer, Participant, Viewer)
- Protected API routes and middleware
- Session management with secure cookies
- User registration and login flows

## Technical Requirements

### Dependencies
- NextAuth 5.0.0-beta.25
- Prisma ORM with PostgreSQL
- Zod for input validation
- bcryptjs for password hashing

### Constraints
- Must use NextAuth with Prisma adapter
- All passwords must be hashed
- Sessions must use HTTP-only cookies
- API routes must be protected with middleware

## Implementation Phases

### Phase 1: Database Schema & Setup (2 days)

#### Tasks
1. **Update Prisma Schema**
   - Add user roles enum
   - Update User model with role field
   - Ensure NextAuth compatibility

2. **Create Database Migration**
   - Generate Prisma migration for schema changes
   - Add indexes for performance optimization

3. **Setup Environment Variables**
   - Configure NextAuth secret
   - Database connection string
   - Email provider settings (for future use)

#### Database Changes

```prisma
enum UserRole {
  ADMIN
  ORGANIZER
  PARTICIPANT
  VIEWER
}

model User {
  id            String    @id @default(cuid())
  email         String    @unique
  name          String?
  role          UserRole  @default(PARTICIPANT)
  emailVerified DateTime?
  image         String?
  password      String?   // For email/password auth
  createdAt     DateTime  @default(now())
  updatedAt     DateTime  @updatedAt
  
  // NextAuth relations
  accounts      Account[]
  sessions      Session[]
  
  // Chess platform relations
  profile       Profile?
  tournaments   TournamentParticipant[]
  games         Game[]    @relation("PlayerGames")
  whiteGames    Game[]    @relation("WhitePlayer")
  blackGames    Game[]    @relation("BlackPlayer")
  createdTournaments Tournament[] @relation("TournamentCreator")
  
  @@index([email])
  @@index([role])
}
```

#### Acceptance Criteria
- [ ] Prisma schema updated with user roles
- [ ] Database migration created and tested
- [ ] Environment variables configured
- [ ] No breaking changes to existing auth tables

### Phase 2: NextAuth Configuration (3 days)

#### Tasks
1. **Configure NextAuth Providers**
   - Setup Credentials provider for email/password
   - Configure Prisma adapter
   - Setup session strategy

2. **Implement Password Hashing**
   - Create password hashing utilities
   - Secure password comparison functions

3. **Create Auth API Routes**
   - Configure NextAuth API routes
   - Custom signin/signup pages

#### Implementation

```typescript
// src/server/auth/config.ts
import { PrismaAdapter } from "@auth/prisma-adapter";
import { type DefaultSession, type NextAuthConfig } from "next-auth";
import CredentialsProvider from "next-auth/providers/credentials";
import bcrypt from "bcryptjs";
import { db } from "@/server/db";
import { loginSchema } from "@/lib/validations/auth";

declare module "next-auth" {
  interface Session extends DefaultSession {
    user: {
      id: string;
      role: "ADMIN" | "ORGANIZER" | "PARTICIPANT" | "VIEWER";
    } & DefaultSession["user"];
  }

  interface User {
    role: "ADMIN" | "ORGANIZER" | "PARTICIPANT" | "VIEWER";
  }
}

export const authConfig = {
  adapter: PrismaAdapter(db),
  providers: [
    CredentialsProvider({
      name: "credentials",
      credentials: {
        email: { label: "Email", type: "email" },
        password: { label: "Password", type: "password" }
      },
      async authorize(credentials) {
        try {
          const { email, password } = loginSchema.parse(credentials);
          
          const user = await db.user.findUnique({
            where: { email },
            select: {
              id: true,
              email: true,
              name: true,
              password: true,
              role: true,
            },
          });

          if (!user?.password) return null;

          const isValid = await bcrypt.compare(password, user.password);
          if (!isValid) return null;

          return {
            id: user.id,
            email: user.email,
            name: user.name,
            role: user.role,
          };
        } catch {
          return null;
        }
      },
    }),
  ],
  session: {
    strategy: "jwt",
  },
  callbacks: {
    jwt: ({ token, user }) => {
      if (user) {
        token.role = user.role;
      }
      return token;
    },
    session: ({ session, token }) => {
      if (token) {
        session.user.id = token.sub!;
        session.user.role = token.role as any;
      }
      return session;
    },
  },
  pages: {
    signIn: "/auth/signin",
    signUp: "/auth/signup",
  },
} satisfies NextAuthConfig;
```

#### Acceptance Criteria
- [ ] NextAuth configured with Credentials provider
- [ ] Password hashing implemented securely
- [ ] JWT session strategy working
- [ ] Custom auth pages created
- [ ] User role included in session

### Phase 3: Middleware & Route Protection (2 days)

#### Tasks
1. **Create Authentication Middleware**
   - Protect API routes based on authentication
   - Role-based access control implementation

2. **Create tRPC Procedures**
   - Protected procedure for authenticated users
   - Role-specific procedures (admin, organizer)

3. **Implement Route Guards**
   - Client-side route protection
   - Redirect logic for unauthorized access

#### Implementation

```typescript
// src/server/api/trpc.ts
import { TRPCError, initTRPC } from "@trpc/server";
import { type CreateNextContextOptions } from "@trpc/server/adapters/next";
import { type Session } from "next-auth";
import { getServerSession } from "next-auth/next";
import { authConfig } from "@/server/auth/config";
import { db } from "@/server/db";

interface CreateContextOptions {
  session: Session | null;
}

const createInnerTRPCContext = (opts: CreateContextOptions) => {
  return {
    session: opts.session,
    db,
  };
};

export const createTRPCContext = async (opts: CreateNextContextOptions) => {
  const { req, res } = opts;
  const session = await getServerSession(req, res, authConfig);
  return createInnerTRPCContext({
    session,
  });
};

const t = initTRPC.context<typeof createTRPCContext>().create();

export const createTRPCRouter = t.router;

// Base procedures
export const publicProcedure = t.procedure;

export const protectedProcedure = t.procedure.use(({ ctx, next }) => {
  if (!ctx.session || !ctx.session.user) {
    throw new TRPCError({ code: "UNAUTHORIZED" });
  }
  return next({
    ctx: {
      session: { ...ctx.session, user: ctx.session.user },
    },
  });
});

// Role-based procedures
export const adminProcedure = protectedProcedure.use(({ ctx, next }) => {
  if (ctx.session.user.role !== "ADMIN") {
    throw new TRPCError({ code: "FORBIDDEN" });
  }
  return next({ ctx });
});

export const organizerProcedure = protectedProcedure.use(({ ctx, next }) => {
  if (!["ADMIN", "ORGANIZER"].includes(ctx.session.user.role)) {
    throw new TRPCError({ code: "FORBIDDEN" });
  }
  return next({ ctx });
});
```

#### Acceptance Criteria
- [ ] API routes protected with authentication middleware
- [ ] Role-based access control implemented
- [ ] tRPC procedures support role checking
- [ ] Client-side route guards working
- [ ] Proper error handling for unauthorized access

### Phase 4: Registration & Authentication UI (3 days)

#### Tasks
1. **Create Registration Form**
   - User registration with email/password
   - Input validation and error handling
   - Role assignment (default: PARTICIPANT)

2. **Create Login Form**
   - Email/password authentication
   - Remember me functionality
   - Forgot password placeholder

3. **Create Authentication Components**
   - Sign-in/sign-out buttons
   - User profile dropdown
   - Role-based navigation

#### Implementation

```typescript
// src/lib/validations/auth.ts
import { z } from "zod";

export const loginSchema = z.object({
  email: z.string().email("Invalid email address"),
  password: z.string().min(6, "Password must be at least 6 characters"),
});

export const registerSchema = z.object({
  name: z.string().min(2, "Name must be at least 2 characters"),
  email: z.string().email("Invalid email address"),
  password: z.string().min(6, "Password must be at least 6 characters"),
  confirmPassword: z.string(),
}).refine((data) => data.password === data.confirmPassword, {
  message: "Passwords don't match",
  path: ["confirmPassword"],
});

export type LoginInput = z.infer<typeof loginSchema>;
export type RegisterInput = z.infer<typeof registerSchema>;
```

```typescript
// src/server/api/routers/auth.ts
import { z } from "zod";
import bcrypt from "bcryptjs";
import { createTRPCRouter, publicProcedure } from "@/server/api/trpc";
import { registerSchema } from "@/lib/validations/auth";
import { TRPCError } from "@trpc/server";

export const authRouter = createTRPCRouter({
  register: publicProcedure
    .input(registerSchema)
    .mutation(async ({ ctx, input }) => {
      const { name, email, password } = input;

      // Check if user already exists
      const existingUser = await ctx.db.user.findUnique({
        where: { email },
      });

      if (existingUser) {
        throw new TRPCError({
          code: "CONFLICT",
          message: "User with this email already exists",
        });
      }

      // Hash password
      const hashedPassword = await bcrypt.hash(password, 12);

      // Create user
      const user = await ctx.db.user.create({
        data: {
          name,
          email,
          password: hashedPassword,
          role: "PARTICIPANT", // Default role
        },
      });

      return {
        id: user.id,
        email: user.email,
        name: user.name,
      };
    }),
});
```

#### Acceptance Criteria
- [ ] Registration form with validation working
- [ ] Login form authenticates users correctly
- [ ] Password hashing implemented securely
- [ ] User roles assigned properly
- [ ] Authentication state managed correctly
- [ ] Error messages display appropriately

### Phase 5: Testing & Security (2 days)

#### Tasks
1. **Unit Tests**
   - Authentication utility functions
   - Password hashing/validation
   - Input validation schemas

2. **Integration Tests**
   - Registration flow end-to-end
   - Login/logout functionality
   - Role-based access control

3. **Security Audit**
   - Password security review
   - Session security validation
   - Input sanitization verification

#### Test Implementation

```typescript
// __tests__/auth/registration.test.ts
import { describe, expect, it } from '@jest/globals';
import { registerSchema } from '@/lib/validations/auth';
import bcrypt from 'bcryptjs';

describe('Authentication Validation', () => {
  it('should validate registration input correctly', () => {
    const validInput = {
      name: 'John Doe',
      email: 'john@example.com',
      password: 'password123',
      confirmPassword: 'password123',
    };

    const result = registerSchema.safeParse(validInput);
    expect(result.success).toBe(true);
  });

  it('should reject invalid email formats', () => {
    const invalidInput = {
      name: 'John Doe',
      email: 'invalid-email',
      password: 'password123',
      confirmPassword: 'password123',
    };

    const result = registerSchema.safeParse(invalidInput);
    expect(result.success).toBe(false);
  });
});

describe('Password Security', () => {
  it('should hash passwords securely', async () => {
    const password = 'testpassword123';
    const hash = await bcrypt.hash(password, 12);
    
    expect(hash).not.toBe(password);
    expect(await bcrypt.compare(password, hash)).toBe(true);
  });
});
```

#### Acceptance Criteria
- [ ] All authentication flows tested
- [ ] Security vulnerabilities addressed
- [ ] Input validation comprehensive
- [ ] Error handling robust
- [ ] Performance benchmarks met

## Risk Assessment

### High Risk
- **Security vulnerabilities**: Implement comprehensive security reviews
- **Session management issues**: Use NextAuth best practices
- **Password security**: Follow industry standards for hashing

### Medium Risk
- **Role permission complexity**: Create clear role hierarchy
- **Authentication state sync**: Ensure consistent state management

### Low Risk
- **UI/UX consistency**: Follow established design patterns
- **Performance optimization**: Monitor and optimize as needed

## Success Metrics

- User registration success rate > 95%
- Authentication response time < 200ms
- Zero security vulnerabilities in audit
- 100% test coverage for critical auth paths
- All role-based access controls working correctly

## Dependencies for Next Phase

This authentication system is required before implementing:
- User Profile Management
- Tournament Management
- Any protected API endpoints
- Dashboard functionality
