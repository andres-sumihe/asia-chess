# User Profile Management Implementation Plan

## Feature Overview

Implement comprehensive user profile management system allowing users to maintain their chess-related information, ratings, and personal details.

## Objectives

- User profile creation and editing
- Chess rating management (FIDE and local)
- Personal information storage
- Profile validation and data integrity
- Privacy controls for profile visibility

## Technical Requirements

### Dependencies
- Authentication system (must be completed first)
- Prisma ORM with PostgreSQL
- Zod for input validation
- React Hook Form for form management
- shadcn/ui components

### Constraints
- All profile operations must be authenticated
- Data validation on both client and server
- Optional fields must handle null values properly
- Rating updates should be tracked historically

## Implementation Phases

### Phase 1: Database Schema & Models (2 days)

#### Tasks
1. **Create Profile Schema**
   - Design comprehensive profile model
   - Add proper relationships and constraints
   - Include rating history tracking

2. **Generate Database Migration**
   - Create Prisma migration for profile tables
   - Add indexes for performance
   - Test migration rollback scenarios

#### Database Changes

```prisma
model Profile {
  id          String   @id @default(cuid())
  userId      String   @unique
  user        User     @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  // Chess-specific information
  fideRating  Int?     @db.SmallInt
  localRating Int?     @db.SmallInt
  fideId      String?  @unique // FIDE ID for official players
  
  // Personal information
  firstName   String?
  lastName    String?
  country     String?  @db.Char(2) // ISO country code
  club        String?
  birthDate   DateTime?
  phone       String?
  bio         String?  @db.Text
  
  // Privacy settings
  isPublic    Boolean  @default(true)
  showRating  Boolean  @default(true)
  showCountry Boolean  @default(true)
  showClub    Boolean  @default(true)
  
  // Metadata
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
  
  // Relations
  ratingHistory RatingHistory[]
  
  @@index([country])
  @@index([club])
  @@index([fideRating])
  @@index([localRating])
}

model RatingHistory {
  id        String      @id @default(cuid())
  profileId String
  profile   Profile     @relation(fields: [profileId], references: [id], onDelete: Cascade)
  
  ratingType RatingType
  oldRating  Int?
  newRating  Int
  change     Int         // Calculated field: newRating - oldRating
  reason     String?     // Tournament, manual adjustment, etc.
  date       DateTime    @default(now())
  
  // Optional tournament reference
  tournamentId String?
  tournament   Tournament? @relation(fields: [tournamentId], references: [id])
  
  @@index([profileId, date])
  @@index([ratingType])
}

enum RatingType {
  FIDE
  LOCAL
}
```

#### Acceptance Criteria
- [ ] Profile model created with all required fields
- [ ] Rating history tracking implemented
- [ ] Database constraints and indexes added
- [ ] Migration tested in development environment
- [ ] Data integrity rules enforced

### Phase 2: Validation Schemas & Types (1 day)

#### Tasks
1. **Create Zod Validation Schemas**
   - Profile creation and update schemas
   - Country code validation
   - Rating range validation

2. **Generate TypeScript Types**
   - Profile input/output types
   - Rating history types
   - Privacy settings types

#### Implementation

```typescript
// src/lib/validations/profile.ts
import { z } from "zod";

// Country codes validation (subset for common countries)
const countryCodeSchema = z.string().length(2).regex(/^[A-Z]{2}$/, "Invalid country code");

// Phone number validation (basic international format)
const phoneSchema = z.string().regex(/^\+?[\d\s\-\(\)]{10,20}$/, "Invalid phone number format");

export const createProfileSchema = z.object({
  firstName: z.string().min(1, "First name is required").max(50, "First name too long").optional(),
  lastName: z.string().min(1, "Last name is required").max(50, "Last name too long").optional(),
  fideRating: z.number().int().min(0).max(3000).optional(),
  localRating: z.number().int().min(0).max(3000).optional(),
  fideId: z.string().regex(/^\d{8,10}$/, "FIDE ID must be 8-10 digits").optional(),
  country: countryCodeSchema.optional(),
  club: z.string().max(100, "Club name too long").optional(),
  birthDate: z.date().max(new Date(), "Birth date cannot be in the future").optional(),
  phone: phoneSchema.optional(),
  bio: z.string().max(500, "Bio too long").optional(),
  isPublic: z.boolean().default(true),
  showRating: z.boolean().default(true),
  showCountry: z.boolean().default(true),
  showClub: z.boolean().default(true),
});

export const updateProfileSchema = createProfileSchema.partial();

export const ratingUpdateSchema = z.object({
  ratingType: z.enum(["FIDE", "LOCAL"]),
  newRating: z.number().int().min(0).max(3000),
  reason: z.string().max(200).optional(),
  tournamentId: z.string().optional(),
});

// Output types
export type ProfileInput = z.infer<typeof createProfileSchema>;
export type ProfileUpdate = z.infer<typeof updateProfileSchema>;
export type RatingUpdate = z.infer<typeof ratingUpdateSchema>;

// Privacy settings
export type ProfilePrivacy = {
  isPublic: boolean;
  showRating: boolean;
  showCountry: boolean;
  showClub: boolean;
};

// Full profile with relations
export type FullProfile = {
  id: string;
  userId: string;
  firstName?: string;
  lastName?: string;
  fideRating?: number;
  localRating?: number;
  fideId?: string;
  country?: string;
  club?: string;
  birthDate?: Date;
  phone?: string;
  bio?: string;
  privacy: ProfilePrivacy;
  createdAt: Date;
  updatedAt: Date;
  ratingHistory: RatingHistory[];
};
```

#### Acceptance Criteria
- [ ] All input validation schemas created
- [ ] TypeScript types properly generated
- [ ] Country code validation working
- [ ] Rating range validation implemented
- [ ] Privacy settings validation added

### Phase 3: API Implementation (3 days)

#### Tasks
1. **Create Profile tRPC Router**
   - CRUD operations for profiles
   - Rating history management
   - Privacy controls

2. **Implement Business Logic**
   - Profile creation workflow
   - Rating update calculations
   - Privacy filtering logic

3. **Add Error Handling**
   - Comprehensive error messages
   - Input validation errors
   - Database constraint violations

#### Implementation

```typescript
// src/server/api/routers/profile.ts
import { z } from "zod";
import { createTRPCRouter, protectedProcedure, publicProcedure } from "@/server/api/trpc";
import { createProfileSchema, updateProfileSchema, ratingUpdateSchema } from "@/lib/validations/profile";
import { TRPCError } from "@trpc/server";

export const profileRouter = createTRPCRouter({
  // Get current user's profile
  getCurrent: protectedProcedure.query(async ({ ctx }) => {
    const profile = await ctx.db.profile.findUnique({
      where: { userId: ctx.session.user.id },
      include: {
        ratingHistory: {
          orderBy: { date: "desc" },
          take: 10, // Last 10 rating changes
        },
      },
    });

    return profile;
  }),

  // Get public profile by user ID
  getByUserId: publicProcedure
    .input(z.object({ userId: z.string() }))
    .query(async ({ ctx, input }) => {
      const profile = await ctx.db.profile.findUnique({
        where: { userId: input.userId },
        include: {
          user: {
            select: { name: true, email: false }, // Don't expose email
          },
          ratingHistory: {
            where: { reason: { not: null } }, // Only show tournament-related changes
            orderBy: { date: "desc" },
            take: 5,
          },
        },
      });

      if (!profile || !profile.isPublic) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Profile not found or not public",
        });
      }

      // Filter based on privacy settings
      return {
        ...profile,
        fideRating: profile.showRating ? profile.fideRating : null,
        localRating: profile.showRating ? profile.localRating : null,
        country: profile.showCountry ? profile.country : null,
        club: profile.showClub ? profile.club : null,
        phone: null, // Never show phone publicly
        birthDate: null, // Never show birth date publicly
      };
    }),

  // Create profile
  create: protectedProcedure
    .input(createProfileSchema)
    .mutation(async ({ ctx, input }) => {
      // Check if profile already exists
      const existingProfile = await ctx.db.profile.findUnique({
        where: { userId: ctx.session.user.id },
      });

      if (existingProfile) {
        throw new TRPCError({
          code: "CONFLICT",
          message: "Profile already exists",
        });
      }

      // Check FIDE ID uniqueness if provided
      if (input.fideId) {
        const existingFideId = await ctx.db.profile.findUnique({
          where: { fideId: input.fideId },
        });

        if (existingFideId) {
          throw new TRPCError({
            code: "CONFLICT",
            message: "FIDE ID already registered",
          });
        }
      }

      const profile = await ctx.db.profile.create({
        data: {
          ...input,
          userId: ctx.session.user.id,
        },
      });

      return profile;
    }),

  // Update profile
  update: protectedProcedure
    .input(updateProfileSchema)
    .mutation(async ({ ctx, input }) => {
      const existingProfile = await ctx.db.profile.findUnique({
        where: { userId: ctx.session.user.id },
      });

      if (!existingProfile) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Profile not found",
        });
      }

      // Check FIDE ID uniqueness if being updated
      if (input.fideId && input.fideId !== existingProfile.fideId) {
        const existingFideId = await ctx.db.profile.findUnique({
          where: { fideId: input.fideId },
        });

        if (existingFideId) {
          throw new TRPCError({
            code: "CONFLICT",
            message: "FIDE ID already registered",
          });
        }
      }

      const updatedProfile = await ctx.db.profile.update({
        where: { userId: ctx.session.user.id },
        data: input,
      });

      return updatedProfile;
    }),

  // Update rating with history tracking
  updateRating: protectedProcedure
    .input(ratingUpdateSchema)
    .mutation(async ({ ctx, input }) => {
      const profile = await ctx.db.profile.findUnique({
        where: { userId: ctx.session.user.id },
      });

      if (!profile) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Profile not found",
        });
      }

      const currentRating = input.ratingType === "FIDE" ? profile.fideRating : profile.localRating;
      const change = input.newRating - (currentRating || 0);

      // Update rating and create history entry in transaction
      const result = await ctx.db.$transaction(async (tx) => {
        // Update profile rating
        const updatedProfile = await tx.profile.update({
          where: { id: profile.id },
          data: {
            [input.ratingType === "FIDE" ? "fideRating" : "localRating"]: input.newRating,
          },
        });

        // Create rating history entry
        const historyEntry = await tx.ratingHistory.create({
          data: {
            profileId: profile.id,
            ratingType: input.ratingType,
            oldRating: currentRating,
            newRating: input.newRating,
            change,
            reason: input.reason,
            tournamentId: input.tournamentId,
          },
        });

        return { profile: updatedProfile, history: historyEntry };
      });

      return result;
    }),

  // Get rating history
  getRatingHistory: protectedProcedure
    .input(z.object({
      ratingType: z.enum(["FIDE", "LOCAL"]).optional(),
      limit: z.number().min(1).max(100).default(20),
    }))
    .query(async ({ ctx, input }) => {
      const profile = await ctx.db.profile.findUnique({
        where: { userId: ctx.session.user.id },
      });

      if (!profile) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Profile not found",
        });
      }

      const history = await ctx.db.ratingHistory.findMany({
        where: {
          profileId: profile.id,
          ...(input.ratingType && { ratingType: input.ratingType }),
        },
        orderBy: { date: "desc" },
        take: input.limit,
        include: {
          tournament: {
            select: { name: true, id: true },
          },
        },
      });

      return history;
    }),

  // Delete profile
  delete: protectedProcedure.mutation(async ({ ctx }) => {
    const profile = await ctx.db.profile.findUnique({
      where: { userId: ctx.session.user.id },
    });

    if (!profile) {
      throw new TRPCError({
        code: "NOT_FOUND",
        message: "Profile not found",
      });
    }

    // Delete profile (cascade will handle rating history)
    await ctx.db.profile.delete({
      where: { id: profile.id },
    });

    return { success: true };
  }),
});
```

#### Acceptance Criteria
- [ ] All CRUD operations implemented
- [ ] Rating history tracking working
- [ ] Privacy controls enforced
- [ ] Error handling comprehensive
- [ ] Input validation on all endpoints
- [ ] Proper database transactions for rating updates

### Phase 4: Frontend Components (4 days)

#### Tasks
1. **Create Profile Forms**
   - Profile creation form
   - Profile editing form
   - Rating update form

2. **Create Profile Display Components**
   - Profile card component
   - Rating history display
   - Privacy settings interface

3. **Create Profile Pages**
   - My profile page
   - Public profile view
   - Profile settings page

#### Implementation

```typescript
// src/components/profile/profile-form.tsx
"use client";

import { useState } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Switch } from "@/components/ui/switch";
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/components/ui/form";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { toast } from "@/components/ui/use-toast";
import { api } from "@/trpc/react";
import { createProfileSchema, type ProfileInput } from "@/lib/validations/profile";
import { COUNTRIES } from "@/lib/constants/countries";

interface ProfileFormProps {
  defaultValues?: Partial<ProfileInput>;
  onSuccess?: () => void;
  isEditing?: boolean;
}

export function ProfileForm({ defaultValues, onSuccess, isEditing = false }: ProfileFormProps) {
  const [isSubmitting, setIsSubmitting] = useState(false);

  const form = useForm<ProfileInput>({
    resolver: zodResolver(createProfileSchema),
    defaultValues: {
      isPublic: true,
      showRating: true,
      showCountry: true,
      showClub: true,
      ...defaultValues,
    },
  });

  const createProfile = api.profile.create.useMutation({
    onSuccess: () => {
      toast({ title: "Profile created successfully" });
      onSuccess?.();
    },
    onError: (error) => {
      toast({ title: "Error", description: error.message, variant: "destructive" });
    },
  });

  const updateProfile = api.profile.update.useMutation({
    onSuccess: () => {
      toast({ title: "Profile updated successfully" });
      onSuccess?.();
    },
    onError: (error) => {
      toast({ title: "Error", description: error.message, variant: "destructive" });
    },
  });

  const onSubmit = async (data: ProfileInput) => {
    setIsSubmitting(true);
    try {
      if (isEditing) {
        await updateProfile.mutateAsync(data);
      } else {
        await createProfile.mutateAsync(data);
      }
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        {/* Personal Information */}
        <div className="space-y-4">
          <h3 className="text-lg font-medium">Personal Information</h3>
          
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <FormField
              control={form.control}
              name="firstName"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>First Name</FormLabel>
                  <FormControl>
                    <Input placeholder="Enter your first name" {...field} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="lastName"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Last Name</FormLabel>
                  <FormControl>
                    <Input placeholder="Enter your last name" {...field} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
          </div>

          <FormField
            control={form.control}
            name="country"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Country</FormLabel>
                <Select onValueChange={field.onChange} defaultValue={field.value}>
                  <FormControl>
                    <SelectTrigger>
                      <SelectValue placeholder="Select your country" />
                    </SelectTrigger>
                  </FormControl>
                  <SelectContent>
                    {COUNTRIES.map((country) => (
                      <SelectItem key={country.code} value={country.code}>
                        {country.flag} {country.name}
                      </SelectItem>
                    ))}
                  </SelectContent>
                </Select>
                <FormMessage />
              </FormItem>
            )}
          />

          <FormField
            control={form.control}
            name="club"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Chess Club</FormLabel>
                <FormControl>
                  <Input placeholder="Enter your chess club" {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />

          <FormField
            control={form.control}
            name="bio"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Bio</FormLabel>
                <FormControl>
                  <Textarea 
                    placeholder="Tell us about yourself..." 
                    className="resize-none" 
                    {...field} 
                  />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
        </div>

        {/* Chess Information */}
        <div className="space-y-4">
          <h3 className="text-lg font-medium">Chess Information</h3>
          
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <FormField
              control={form.control}
              name="fideRating"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>FIDE Rating</FormLabel>
                  <FormControl>
                    <Input 
                      type="number" 
                      placeholder="Enter your FIDE rating" 
                      {...field}
                      onChange={(e) => field.onChange(e.target.value ? parseInt(e.target.value) : undefined)}
                    />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="localRating"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Local Rating</FormLabel>
                  <FormControl>
                    <Input 
                      type="number" 
                      placeholder="Enter your local rating" 
                      {...field}
                      onChange={(e) => field.onChange(e.target.value ? parseInt(e.target.value) : undefined)}
                    />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
          </div>

          <FormField
            control={form.control}
            name="fideId"
            render={({ field }) => (
              <FormItem>
                <FormLabel>FIDE ID</FormLabel>
                <FormControl>
                  <Input placeholder="Enter your FIDE ID" {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
        </div>

        {/* Privacy Settings */}
        <div className="space-y-4">
          <h3 className="text-lg font-medium">Privacy Settings</h3>
          
          <div className="space-y-3">
            <FormField
              control={form.control}
              name="isPublic"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between">
                  <FormLabel>Make profile public</FormLabel>
                  <FormControl>
                    <Switch checked={field.value} onCheckedChange={field.onChange} />
                  </FormControl>
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="showRating"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between">
                  <FormLabel>Show ratings publicly</FormLabel>
                  <FormControl>
                    <Switch checked={field.value} onCheckedChange={field.onChange} />
                  </FormControl>
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="showCountry"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between">
                  <FormLabel>Show country publicly</FormLabel>
                  <FormControl>
                    <Switch checked={field.value} onCheckedChange={field.onChange} />
                  </FormControl>
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="showClub"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between">
                  <FormLabel>Show club publicly</FormLabel>
                  <FormControl>
                    <Switch checked={field.value} onCheckedChange={field.onChange} />
                  </FormControl>
                </FormItem>
              )}
            />
          </div>
        </div>

        <Button type="submit" disabled={isSubmitting}>
          {isSubmitting ? "Saving..." : isEditing ? "Update Profile" : "Create Profile"}
        </Button>
      </form>
    </Form>
  );
}
```

#### Acceptance Criteria
- [ ] Profile forms with validation working
- [ ] Profile display components responsive
- [ ] Rating history visualization implemented
- [ ] Privacy settings interface functional
- [ ] Error handling in UI components
- [ ] Loading states implemented

### Phase 5: Testing & Performance (2 days)

#### Tasks
1. **Unit Tests**
   - Profile validation functions
   - Rating calculation logic
   - Privacy filtering functions

2. **Integration Tests**
   - Profile CRUD operations
   - Rating history tracking
   - Privacy controls

3. **Performance Optimization**
   - Database query optimization
   - Component rendering optimization
   - Form validation performance

#### Test Implementation

```typescript
// __tests__/profile/profile-api.test.ts
import { describe, expect, it, beforeEach } from '@jest/globals';
import { createTRPCMsw } from 'msw-trpc';
import { profileRouter } from '@/server/api/routers/profile';
import { createProfileSchema } from '@/lib/validations/profile';

describe('Profile API', () => {
  const trpcMsw = createTRPCMsw(profileRouter);

  beforeEach(() => {
    // Setup test database state
  });

  it('should create profile with valid data', async () => {
    const profileData = {
      firstName: 'John',
      lastName: 'Doe',
      fideRating: 1500,
      country: 'US',
      isPublic: true,
    };

    const result = createProfileSchema.safeParse(profileData);
    expect(result.success).toBe(true);
  });

  it('should reject invalid FIDE rating', async () => {
    const profileData = {
      firstName: 'John',
      fideRating: 4000, // Invalid rating
    };

    const result = createProfileSchema.safeParse(profileData);
    expect(result.success).toBe(false);
  });

  it('should track rating history correctly', async () => {
    // Test rating history creation and calculation
  });
});
```

#### Acceptance Criteria
- [ ] All profile operations tested
- [ ] Performance benchmarks met
- [ ] Error scenarios covered
- [ ] Privacy controls validated
- [ ] Data integrity maintained

## Risk Assessment

### High Risk
- **Data privacy compliance**: Ensure GDPR compliance for EU users
- **Rating data integrity**: Prevent unauthorized rating modifications
- **FIDE ID conflicts**: Handle duplicate FIDE ID registrations

### Medium Risk
- **Profile data validation**: Comprehensive validation on all inputs
- **Performance with rating history**: Optimize queries for large datasets

### Low Risk
- **UI/UX consistency**: Follow established design patterns
- **Form submission handling**: Standard form validation patterns

## Success Metrics

- Profile creation success rate > 95%
- Profile update response time < 300ms
- Rating history queries < 200ms
- Zero privacy control failures
- 90% test coverage for profile operations

## Dependencies for Next Phase

This profile management system enables:
- Tournament participant registration
- Player identification in games
- Rating calculations after tournaments
- User directory and search functionality
