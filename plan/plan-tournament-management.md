# Tournament Management Implementation Plan

## Feature Overview

Implement comprehensive tournament management system supporting multiple tournament formats, lifecycle management, and organizer controls.

## Objectives

- Tournament creation and configuration
- Support for Round Robin and Swiss System formats
- Tournament lifecycle management (Draft → Registration → In Progress → Completed)
- Organizer permissions and controls
- Tournament discovery and listing
- Registration management

## Technical Requirements

### Dependencies
- Authentication system (completed)
- User Profile Management (completed)
- Prisma ORM with PostgreSQL
- Zod for input validation
- Date manipulation libraries (date-fns)

### Constraints
- Only authenticated users with ORGANIZER or ADMIN roles can create tournaments
- Tournament dates must be in the future
- Tournament format cannot be changed after registration opens
- Maximum participants limit must be enforced

## Implementation Phases

### Phase 1: Database Schema & Models (2 days)

#### Tasks
1. **Create Tournament Schema**
   - Design tournament model with all required fields
   - Add tournament status enum
   - Create proper relationships

2. **Generate Database Migration**
   - Create Prisma migration for tournament tables
   - Add indexes for performance
   - Test constraints and relationships

#### Database Changes

```prisma
enum TournamentFormat {
  ROUND_ROBIN
  SWISS_SYSTEM
  KNOCKOUT
}

enum TournamentStatus {
  DRAFT
  REGISTRATION_OPEN
  REGISTRATION_CLOSED
  IN_PROGRESS
  COMPLETED
  CANCELLED
}

model Tournament {
  id          String     @id @default(cuid())
  name        String
  description String?    @db.Text
  format      TournamentFormat
  status      TournamentStatus @default(DRAFT)
  
  // Tournament dates
  startDate   DateTime
  endDate     DateTime?
  registrationStart DateTime?
  registrationEnd   DateTime?
  
  // Participant limits
  maxParticipants Int?
  minParticipants Int?     @default(4)
  
  // Tournament settings
  timeControl     String?  // e.g., "90+30", "5+3"
  rounds          Int?     // For Swiss system
  currentRound    Int      @default(0)
  
  // Location information
  venue           String?
  address         String?
  city            String?
  country         String?  @db.Char(2)
  
  // Tournament rules
  allowDraws      Boolean  @default(true)
  allowByes       Boolean  @default(true)
  pairingSystem   String?  @default("Swiss") // Swiss, Round Robin
  tiebreakRules   Json?    // Array of tiebreak methods
  
  // Organizer information
  createdBy       User     @relation("TournamentCreator", fields: [createdById], references: [id])
  createdById     String
  organizers      TournamentOrganizer[]
  
  // Relations
  participants    TournamentParticipant[]
  rounds          Round[]
  ratingHistory   RatingHistory[]
  
  // Metadata
  createdAt       DateTime @default(now())
  updatedAt       DateTime @updatedAt
  
  @@index([status])
  @@index([startDate])
  @@index([format])
  @@index([createdById])
  @@index([country])
  @@index([city])
}

model TournamentOrganizer {
  id           String     @id @default(cuid())
  tournamentId String
  userId       String
  tournament   Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  user         User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  role         OrganizerRole @default(ASSISTANT)
  addedAt      DateTime   @default(now())
  
  @@unique([tournamentId, userId])
}

model TournamentParticipant {
  id           String     @id @default(cuid())
  tournamentId String
  userId       String
  tournament   Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  user         User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  // Tournament-specific data
  seedRating   Int?       // Rating at time of registration
  score        Float      @default(0)
  tiebreaks    Json?      // Calculated tiebreak scores
  ranking      Int?
  
  // Registration data
  registeredAt DateTime   @default(now())
  status       ParticipantStatus @default(REGISTERED)
  
  // Withdrawal information
  withdrawnAt  DateTime?
  withdrawReason String?
  
  @@unique([tournamentId, userId])
  @@index([tournamentId, ranking])
  @@index([tournamentId, score])
}

enum OrganizerRole {
  CHIEF          // Full control
  ASSISTANT      // Limited control
  ARBITER        // Results only
}

enum ParticipantStatus {
  REGISTERED
  CONFIRMED
  WITHDRAWN
  DISQUALIFIED
}
```

#### Acceptance Criteria
- [ ] Tournament model created with all required fields
- [ ] Tournament status workflow implemented
- [ ] Organizer and participant relationships established
- [ ] Database constraints and indexes added
- [ ] Migration tested in development environment

### Phase 2: Validation Schemas & Business Rules (2 days)

#### Tasks
1. **Create Zod Validation Schemas**
   - Tournament creation and update schemas
   - Registration validation
   - Status transition validation

2. **Implement Business Rules**
   - Tournament date validation
   - Participant limit enforcement
   - Status transition rules

#### Implementation

```typescript
// src/lib/validations/tournament.ts
import { z } from "zod";
import { addDays, isFuture, isAfter } from "date-fns";

export const createTournamentSchema = z.object({
  name: z.string().min(3, "Name must be at least 3 characters").max(100, "Name too long"),
  description: z.string().max(1000, "Description too long").optional(),
  format: z.enum(["ROUND_ROBIN", "SWISS_SYSTEM", "KNOCKOUT"]),
  
  // Dates
  startDate: z.date().refine(isFuture, "Start date must be in the future"),
  endDate: z.date().optional(),
  registrationStart: z.date().optional(),
  registrationEnd: z.date().optional(),
  
  // Participants
  maxParticipants: z.number().int().min(4).max(500).optional(),
  minParticipants: z.number().int().min(2).max(100).default(4),
  
  // Tournament settings
  timeControl: z.string().regex(/^\d+\+\d+$|^\d+ min$/, "Invalid time control format").optional(),
  rounds: z.number().int().min(1).max(20).optional(),
  
  // Location
  venue: z.string().max(200).optional(),
  address: z.string().max(300).optional(),
  city: z.string().max(100).optional(),
  country: z.string().length(2).optional(),
  
  // Rules
  allowDraws: z.boolean().default(true),
  allowByes: z.boolean().default(true),
  tiebreakRules: z.array(z.enum(["BUCHHOLZ", "DIRECT_ENCOUNTER", "WINS", "BLACK_GAMES"])).optional(),
}).refine((data) => {
  // End date must be after start date
  if (data.endDate && !isAfter(data.endDate, data.startDate)) {
    return false;
  }
  
  // Registration end must be before start date
  if (data.registrationEnd && !isAfter(data.startDate, data.registrationEnd)) {
    return false;
  }
  
  // Max participants must be >= min participants
  if (data.maxParticipants && data.maxParticipants < data.minParticipants) {
    return false;
  }
  
  return true;
}, {
  message: "Invalid date or participant configuration",
});

export const updateTournamentSchema = createTournamentSchema.partial().extend({
  id: z.string(),
});

export const tournamentFiltersSchema = z.object({
  status: z.enum(["DRAFT", "REGISTRATION_OPEN", "REGISTRATION_CLOSED", "IN_PROGRESS", "COMPLETED", "CANCELLED"]).optional(),
  format: z.enum(["ROUND_ROBIN", "SWISS_SYSTEM", "KNOCKOUT"]).optional(),
  country: z.string().length(2).optional(),
  city: z.string().optional(),
  startDateFrom: z.date().optional(),
  startDateTo: z.date().optional(),
  search: z.string().optional(),
  page: z.number().int().min(1).default(1),
  limit: z.number().int().min(1).max(100).default(20),
});

export const registerForTournamentSchema = z.object({
  tournamentId: z.string(),
});

export const updateTournamentStatusSchema = z.object({
  tournamentId: z.string(),
  status: z.enum(["REGISTRATION_OPEN", "REGISTRATION_CLOSED", "IN_PROGRESS", "COMPLETED", "CANCELLED"]),
});

// Type exports
export type CreateTournamentInput = z.infer<typeof createTournamentSchema>;
export type UpdateTournamentInput = z.infer<typeof updateTournamentSchema>;
export type TournamentFilters = z.infer<typeof tournamentFiltersSchema>;
export type RegisterForTournament = z.infer<typeof registerForTournamentSchema>;
export type UpdateTournamentStatus = z.infer<typeof updateTournamentStatusSchema>;

// Tournament status transition rules
export const TOURNAMENT_STATUS_TRANSITIONS = {
  DRAFT: ["REGISTRATION_OPEN", "CANCELLED"],
  REGISTRATION_OPEN: ["REGISTRATION_CLOSED", "CANCELLED"],
  REGISTRATION_CLOSED: ["IN_PROGRESS", "CANCELLED"],
  IN_PROGRESS: ["COMPLETED"],
  COMPLETED: [],
  CANCELLED: [],
} as const;

// Business rule validators
export const validateStatusTransition = (currentStatus: string, newStatus: string): boolean => {
  const allowedTransitions = TOURNAMENT_STATUS_TRANSITIONS[currentStatus as keyof typeof TOURNAMENT_STATUS_TRANSITIONS];
  return allowedTransitions.includes(newStatus as any);
};

export const calculateRequiredRounds = (participantCount: number, format: string): number => {
  switch (format) {
    case "ROUND_ROBIN":
      return participantCount - 1;
    case "SWISS_SYSTEM":
      return Math.ceil(Math.log2(participantCount));
    case "KNOCKOUT":
      return Math.ceil(Math.log2(participantCount));
    default:
      return 1;
  }
};
```

#### Acceptance Criteria
- [ ] All input validation schemas created
- [ ] Business rules implemented and tested
- [ ] Status transition validation working
- [ ] Date validation comprehensive
- [ ] Participant limit validation functional

### Phase 3: API Implementation (4 days)

#### Tasks
1. **Create Tournament tRPC Router**
   - CRUD operations for tournaments
   - Registration management
   - Status transitions

2. **Implement Business Logic**
   - Tournament creation workflow
   - Registration validation
   - Organizer permission checks

3. **Add Advanced Features**
   - Tournament search and filtering
   - Participant management
   - Bulk operations

#### Implementation

```typescript
// src/server/api/routers/tournament.ts
import { z } from "zod";
import { createTRPCRouter, protectedProcedure, publicProcedure, organizerProcedure, adminProcedure } from "@/server/api/trpc";
import {
  createTournamentSchema,
  updateTournamentSchema,
  tournamentFiltersSchema,
  registerForTournamentSchema,
  updateTournamentStatusSchema,
  validateStatusTransition,
  calculateRequiredRounds,
} from "@/lib/validations/tournament";
import { TRPCError } from "@trpc/server";

export const tournamentRouter = createTRPCRouter({
  // Get all tournaments with filtering
  getAll: publicProcedure
    .input(tournamentFiltersSchema)
    .query(async ({ ctx, input }) => {
      const {
        status,
        format,
        country,
        city,
        startDateFrom,
        startDateTo,
        search,
        page,
        limit,
      } = input;

      const skip = (page - 1) * limit;

      const where = {
        ...(status && { status }),
        ...(format && { format }),
        ...(country && { country }),
        ...(city && { city: { contains: city, mode: "insensitive" as const } }),
        ...(startDateFrom && { startDate: { gte: startDateFrom } }),
        ...(startDateTo && { startDate: { lte: startDateTo } }),
        ...(search && {
          OR: [
            { name: { contains: search, mode: "insensitive" as const } },
            { description: { contains: search, mode: "insensitive" as const } },
            { venue: { contains: search, mode: "insensitive" as const } },
          ],
        }),
      };

      const [tournaments, total] = await Promise.all([
        ctx.db.tournament.findMany({
          where,
          skip,
          take: limit,
          orderBy: { startDate: "asc" },
          include: {
            createdBy: {
              select: { name: true, profile: { select: { firstName: true, lastName: true } } },
            },
            _count: { select: { participants: true } },
          },
        }),
        ctx.db.tournament.count({ where }),
      ]);

      return {
        tournaments,
        total,
        pages: Math.ceil(total / limit),
        currentPage: page,
      };
    }),

  // Get tournament by ID
  getById: publicProcedure
    .input(z.object({ id: z.string() }))
    .query(async ({ ctx, input }) => {
      const tournament = await ctx.db.tournament.findUnique({
        where: { id: input.id },
        include: {
          createdBy: {
            select: { name: true, profile: { select: { firstName: true, lastName: true } } },
          },
          organizers: {
            include: {
              user: {
                select: { name: true, profile: { select: { firstName: true, lastName: true } } },
              },
            },
          },
          participants: {
            include: {
              user: {
                select: { name: true, profile: { select: { firstName: true, lastName: true, fideRating: true, localRating: true } } },
              },
            },
            orderBy: [{ ranking: "asc" }, { score: "desc" }],
          },
          rounds: {
            orderBy: { roundNumber: "asc" },
            include: {
              games: {
                include: {
                  white: { select: { name: true, profile: { select: { firstName: true, lastName: true } } } },
                  black: { select: { name: true, profile: { select: { firstName: true, lastName: true } } } },
                },
              },
            },
          },
        },
      });

      if (!tournament) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Tournament not found",
        });
      }

      return tournament;
    }),

  // Create new tournament
  create: organizerProcedure
    .input(createTournamentSchema)
    .mutation(async ({ ctx, input }) => {
      // Calculate required rounds for Swiss system
      let calculatedRounds = input.rounds;
      if (input.format === "SWISS_SYSTEM" && !calculatedRounds) {
        calculatedRounds = calculateRequiredRounds(input.maxParticipants || 50, input.format);
      }

      const tournament = await ctx.db.tournament.create({
        data: {
          ...input,
          rounds: calculatedRounds,
          createdById: ctx.session.user.id,
          // Set default registration period if not specified
          registrationStart: input.registrationStart || new Date(),
          registrationEnd: input.registrationEnd || new Date(input.startDate.getTime() - 24 * 60 * 60 * 1000), // 1 day before start
        },
      });

      // Add creator as chief organizer
      await ctx.db.tournamentOrganizer.create({
        data: {
          tournamentId: tournament.id,
          userId: ctx.session.user.id,
          role: "CHIEF",
        },
      });

      return tournament;
    }),

  // Update tournament
  update: protectedProcedure
    .input(updateTournamentSchema)
    .mutation(async ({ ctx, input }) => {
      const { id, ...updateData } = input;

      // Check if user is organizer of this tournament
      const tournament = await ctx.db.tournament.findUnique({
        where: { id },
        include: {
          organizers: { where: { userId: ctx.session.user.id } },
        },
      });

      if (!tournament) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Tournament not found",
        });
      }

      const isOrganizer = tournament.createdById === ctx.session.user.id || 
                         tournament.organizers.length > 0 ||
                         ctx.session.user.role === "ADMIN";

      if (!isOrganizer) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "You don't have permission to update this tournament",
        });
      }

      // Prevent certain updates after registration opens
      if (tournament.status !== "DRAFT") {
        const restrictedFields = ["format", "maxParticipants", "startDate"];
        const hasRestrictedChanges = restrictedFields.some(field => field in updateData);
        
        if (hasRestrictedChanges) {
          throw new TRPCError({
            code: "BAD_REQUEST",
            message: "Cannot modify format, participants, or start date after registration opens",
          });
        }
      }

      const updatedTournament = await ctx.db.tournament.update({
        where: { id },
        data: updateData,
      });

      return updatedTournament;
    }),

  // Register for tournament
  register: protectedProcedure
    .input(registerForTournamentSchema)
    .mutation(async ({ ctx, input }) => {
      const tournament = await ctx.db.tournament.findUnique({
        where: { id: input.tournamentId },
        include: {
          participants: true,
        },
      });

      if (!tournament) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Tournament not found",
        });
      }

      // Check if registration is open
      if (tournament.status !== "REGISTRATION_OPEN") {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: "Registration is not open for this tournament",
        });
      }

      // Check if user is already registered
      const existingParticipant = tournament.participants.find(
        p => p.userId === ctx.session.user.id
      );

      if (existingParticipant) {
        throw new TRPCError({
          code: "CONFLICT",
          message: "You are already registered for this tournament",
        });
      }

      // Check participant limit
      if (tournament.maxParticipants && tournament.participants.length >= tournament.maxParticipants) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: "Tournament is full",
        });
      }

      // Get user's current rating for seeding
      const userProfile = await ctx.db.profile.findUnique({
        where: { userId: ctx.session.user.id },
        select: { fideRating: true, localRating: true },
      });

      const seedRating = userProfile?.fideRating || userProfile?.localRating;

      const participant = await ctx.db.tournamentParticipant.create({
        data: {
          tournamentId: input.tournamentId,
          userId: ctx.session.user.id,
          seedRating,
          status: "REGISTERED",
        },
      });

      return participant;
    }),

  // Withdraw from tournament
  withdraw: protectedProcedure
    .input(z.object({
      tournamentId: z.string(),
      reason: z.string().max(200).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      const participant = await ctx.db.tournamentParticipant.findUnique({
        where: {
          tournamentId_userId: {
            tournamentId: input.tournamentId,
            userId: ctx.session.user.id,
          },
        },
        include: { tournament: true },
      });

      if (!participant) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "You are not registered for this tournament",
        });
      }

      // Prevent withdrawal after tournament starts
      if (participant.tournament.status === "IN_PROGRESS") {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: "Cannot withdraw after tournament has started",
        });
      }

      const updatedParticipant = await ctx.db.tournamentParticipant.update({
        where: { id: participant.id },
        data: {
          status: "WITHDRAWN",
          withdrawnAt: new Date(),
          withdrawReason: input.reason,
        },
      });

      return updatedParticipant;
    }),

  // Update tournament status (organizer only)
  updateStatus: protectedProcedure
    .input(updateTournamentStatusSchema)
    .mutation(async ({ ctx, input }) => {
      const tournament = await ctx.db.tournament.findUnique({
        where: { id: input.tournamentId },
        include: {
          organizers: { where: { userId: ctx.session.user.id } },
          participants: { where: { status: "REGISTERED" } },
        },
      });

      if (!tournament) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Tournament not found",
        });
      }

      // Check organizer permissions
      const isOrganizer = tournament.createdById === ctx.session.user.id || 
                         tournament.organizers.length > 0 ||
                         ctx.session.user.role === "ADMIN";

      if (!isOrganizer) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "You don't have permission to update this tournament",
        });
      }

      // Validate status transition
      if (!validateStatusTransition(tournament.status, input.status)) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: `Cannot change status from ${tournament.status} to ${input.status}`,
        });
      }

      // Check minimum participants before starting
      if (input.status === "IN_PROGRESS") {
        if (tournament.participants.length < tournament.minParticipants) {
          throw new TRPCError({
            code: "BAD_REQUEST",
            message: `Need at least ${tournament.minParticipants} participants to start tournament`,
          });
        }
      }

      const updatedTournament = await ctx.db.tournament.update({
        where: { id: input.tournamentId },
        data: { status: input.status },
      });

      return updatedTournament;
    }),

  // Delete tournament (organizer only, only if not started)
  delete: protectedProcedure
    .input(z.object({ id: z.string() }))
    .mutation(async ({ ctx, input }) => {
      const tournament = await ctx.db.tournament.findUnique({
        where: { id: input.id },
        include: {
          organizers: { where: { userId: ctx.session.user.id } },
        },
      });

      if (!tournament) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Tournament not found",
        });
      }

      // Check permissions
      const isOrganizer = tournament.createdById === ctx.session.user.id || 
                         tournament.organizers.length > 0 ||
                         ctx.session.user.role === "ADMIN";

      if (!isOrganizer) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "You don't have permission to delete this tournament",
        });
      }

      // Prevent deletion after tournament starts
      if (["IN_PROGRESS", "COMPLETED"].includes(tournament.status)) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: "Cannot delete tournament that has started or completed",
        });
      }

      await ctx.db.tournament.delete({
        where: { id: input.id },
      });

      return { success: true };
    }),

  // Get user's tournaments
  getMyTournaments: protectedProcedure
    .input(z.object({
      type: z.enum(["CREATED", "PARTICIPATING", "ORGANIZING"]).default("PARTICIPATING"),
      page: z.number().int().min(1).default(1),
      limit: z.number().int().min(1).max(50).default(10),
    }))
    .query(async ({ ctx, input }) => {
      const { type, page, limit } = input;
      const skip = (page - 1) * limit;

      let where = {};
      let include = {};

      switch (type) {
        case "CREATED":
          where = { createdById: ctx.session.user.id };
          include = { _count: { select: { participants: true } } };
          break;
        
        case "PARTICIPATING":
          where = { participants: { some: { userId: ctx.session.user.id } } };
          include = {
            participants: {
              where: { userId: ctx.session.user.id },
              select: { status: true, registeredAt: true, score: true, ranking: true },
            },
          };
          break;
        
        case "ORGANIZING":
          where = { organizers: { some: { userId: ctx.session.user.id } } };
          include = { _count: { select: { participants: true } } };
          break;
      }

      const [tournaments, total] = await Promise.all([
        ctx.db.tournament.findMany({
          where,
          skip,
          take: limit,
          orderBy: { startDate: "desc" },
          include,
        }),
        ctx.db.tournament.count({ where }),
      ]);

      return {
        tournaments,
        total,
        pages: Math.ceil(total / limit),
        currentPage: page,
      };
    }),
});
```

#### Acceptance Criteria
- [ ] All CRUD operations implemented
- [ ] Registration workflow functional
- [ ] Status transitions validated
- [ ] Organizer permissions enforced
- [ ] Tournament search and filtering working
- [ ] Error handling comprehensive

### Phase 4: Frontend Components (4 days)

#### Tasks
1. **Create Tournament Forms**
   - Tournament creation form
   - Tournament editing form
   - Registration actions

2. **Create Tournament Display Components**
   - Tournament card component
   - Tournament list/grid
   - Tournament details page

3. **Create Tournament Management UI**
   - Organizer dashboard
   - Participant management
   - Status controls

#### Implementation

```typescript
// src/components/tournament/tournament-form.tsx
"use client";

import { useState } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { format } from "date-fns";
import { CalendarIcon } from "lucide-react";

import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Calendar } from "@/components/ui/calendar";
import { Popover, PopoverContent, PopoverTrigger } from "@/components/ui/popover";
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/components/ui/form";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Switch } from "@/components/ui/switch";
import { toast } from "@/components/ui/use-toast";
import { cn } from "@/lib/utils";
import { api } from "@/trpc/react";
import { createTournamentSchema, type CreateTournamentInput } from "@/lib/validations/tournament";
import { COUNTRIES } from "@/lib/constants/countries";

interface TournamentFormProps {
  defaultValues?: Partial<CreateTournamentInput>;
  onSuccess?: (tournament: any) => void;
  isEditing?: boolean;
}

export function TournamentForm({ defaultValues, onSuccess, isEditing = false }: TournamentFormProps) {
  const [isSubmitting, setIsSubmitting] = useState(false);

  const form = useForm<CreateTournamentInput>({
    resolver: zodResolver(createTournamentSchema),
    defaultValues: {
      format: "SWISS_SYSTEM",
      minParticipants: 4,
      allowDraws: true,
      allowByes: true,
      ...defaultValues,
    },
  });

  const createTournament = api.tournament.create.useMutation({
    onSuccess: (tournament) => {
      toast({ title: "Tournament created successfully" });
      onSuccess?.(tournament);
    },
    onError: (error) => {
      toast({ title: "Error", description: error.message, variant: "destructive" });
    },
  });

  const updateTournament = api.tournament.update.useMutation({
    onSuccess: (tournament) => {
      toast({ title: "Tournament updated successfully" });
      onSuccess?.(tournament);
    },
    onError: (error) => {
      toast({ title: "Error", description: error.message, variant: "destructive" });
    },
  });

  const onSubmit = async (data: CreateTournamentInput) => {
    setIsSubmitting(true);
    try {
      if (isEditing) {
        await updateTournament.mutateAsync({ ...data, id: defaultValues?.id } as any);
      } else {
        await createTournament.mutateAsync(data);
      }
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-8">
        {/* Basic Information */}
        <div className="space-y-4">
          <h3 className="text-lg font-medium">Tournament Information</h3>
          
          <FormField
            control={form.control}
            name="name"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Tournament Name</FormLabel>
                <FormControl>
                  <Input placeholder="Enter tournament name" {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />

          <FormField
            control={form.control}
            name="description"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Description</FormLabel>
                <FormControl>
                  <Textarea 
                    placeholder="Describe your tournament..." 
                    className="resize-none" 
                    {...field} 
                  />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />

          <FormField
            control={form.control}
            name="format"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Tournament Format</FormLabel>
                <Select onValueChange={field.onChange} defaultValue={field.value}>
                  <FormControl>
                    <SelectTrigger>
                      <SelectValue placeholder="Select tournament format" />
                    </SelectTrigger>
                  </FormControl>
                  <SelectContent>
                    <SelectItem value="ROUND_ROBIN">Round Robin</SelectItem>
                    <SelectItem value="SWISS_SYSTEM">Swiss System</SelectItem>
                    <SelectItem value="KNOCKOUT">Knockout</SelectItem>
                  </SelectContent>
                </Select>
                <FormMessage />
              </FormItem>
            )}
          />
        </div>

        {/* Dates */}
        <div className="space-y-4">
          <h3 className="text-lg font-medium">Tournament Dates</h3>
          
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <FormField
              control={form.control}
              name="startDate"
              render={({ field }) => (
                <FormItem className="flex flex-col">
                  <FormLabel>Start Date</FormLabel>
                  <Popover>
                    <PopoverTrigger asChild>
                      <FormControl>
                        <Button
                          variant={"outline"}
                          className={cn(
                            "w-full pl-3 text-left font-normal",
                            !field.value && "text-muted-foreground"
                          )}
                        >
                          {field.value ? (
                            format(field.value, "PPP")
                          ) : (
                            <span>Pick a date</span>
                          )}
                          <CalendarIcon className="ml-auto h-4 w-4 opacity-50" />
                        </Button>
                      </FormControl>
                    </PopoverTrigger>
                    <PopoverContent className="w-auto p-0" align="start">
                      <Calendar
                        mode="single"
                        selected={field.value}
                        onSelect={field.onChange}
                        disabled={(date) => date < new Date()}
                        initialFocus
                      />
                    </PopoverContent>
                  </Popover>
                  <FormMessage />
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="endDate"
              render={({ field }) => (
                <FormItem className="flex flex-col">
                  <FormLabel>End Date (Optional)</FormLabel>
                  <Popover>
                    <PopoverTrigger asChild>
                      <FormControl>
                        <Button
                          variant={"outline"}
                          className={cn(
                            "w-full pl-3 text-left font-normal",
                            !field.value && "text-muted-foreground"
                          )}
                        >
                          {field.value ? (
                            format(field.value, "PPP")
                          ) : (
                            <span>Pick a date</span>
                          )}
                          <CalendarIcon className="ml-auto h-4 w-4 opacity-50" />
                        </Button>
                      </FormControl>
                    </PopoverTrigger>
                    <PopoverContent className="w-auto p-0" align="start">
                      <Calendar
                        mode="single"
                        selected={field.value}
                        onSelect={field.onChange}
                        disabled={(date) => date < new Date()}
                        initialFocus
                      />
                    </PopoverContent>
                  </Popover>
                  <FormMessage />
                </FormItem>
              )}
            />
          </div>
        </div>

        {/* Participants */}
        <div className="space-y-4">
          <h3 className="text-lg font-medium">Participants</h3>
          
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <FormField
              control={form.control}
              name="minParticipants"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Minimum Participants</FormLabel>
                  <FormControl>
                    <Input 
                      type="number" 
                      {...field}
                      onChange={(e) => field.onChange(parseInt(e.target.value) || 4)}
                    />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="maxParticipants"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Maximum Participants (Optional)</FormLabel>
                  <FormControl>
                    <Input 
                      type="number" 
                      {...field}
                      onChange={(e) => field.onChange(e.target.value ? parseInt(e.target.value) : undefined)}
                    />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
          </div>
        </div>

        {/* Location */}
        <div className="space-y-4">
          <h3 className="text-lg font-medium">Location</h3>
          
          <FormField
            control={form.control}
            name="venue"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Venue</FormLabel>
                <FormControl>
                  <Input placeholder="Tournament venue" {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />

          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <FormField
              control={form.control}
              name="city"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>City</FormLabel>
                  <FormControl>
                    <Input placeholder="City" {...field} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="country"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Country</FormLabel>
                  <Select onValueChange={field.onChange} defaultValue={field.value}>
                    <FormControl>
                      <SelectTrigger>
                        <SelectValue placeholder="Select country" />
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
          </div>
        </div>

        {/* Tournament Settings */}
        <div className="space-y-4">
          <h3 className="text-lg font-medium">Tournament Settings</h3>
          
          <FormField
            control={form.control}
            name="timeControl"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Time Control</FormLabel>
                <FormControl>
                  <Input placeholder="e.g., 90+30, 5+3" {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />

          <div className="space-y-3">
            <FormField
              control={form.control}
              name="allowDraws"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between">
                  <FormLabel>Allow draws</FormLabel>
                  <FormControl>
                    <Switch checked={field.value} onCheckedChange={field.onChange} />
                  </FormControl>
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="allowByes"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between">
                  <FormLabel>Allow byes</FormLabel>
                  <FormControl>
                    <Switch checked={field.value} onCheckedChange={field.onChange} />
                  </FormControl>
                </FormItem>
              )}
            />
          </div>
        </div>

        <Button type="submit" disabled={isSubmitting} className="w-full">
          {isSubmitting ? "Creating..." : isEditing ? "Update Tournament" : "Create Tournament"}
        </Button>
      </form>
    </Form>
  );
}
```

#### Acceptance Criteria
- [ ] Tournament forms with comprehensive validation
- [ ] Tournament listing with filtering and search
- [ ] Tournament details page with all information
- [ ] Registration/withdrawal functionality
- [ ] Organizer management interface
- [ ] Status transition controls working

### Phase 5: Testing & Integration (2 days)

#### Tasks
1. **Unit Tests**
   - Tournament validation functions
   - Business rule logic
   - Status transition validation

2. **Integration Tests**
   - Tournament CRUD operations
   - Registration workflow
   - Organizer permissions

3. **Performance Testing**
   - Tournament search performance
   - Large participant list handling
   - Database query optimization

#### Test Implementation

```typescript
// __tests__/tournament/tournament-api.test.ts
import { describe, expect, it, beforeEach } from '@jest/globals';
import { createTournamentSchema, validateStatusTransition } from '@/lib/validations/tournament';

describe('Tournament API', () => {
  describe('Tournament Creation', () => {
    it('should validate tournament data correctly', () => {
      const tournamentData = {
        name: 'Test Tournament',
        format: 'SWISS_SYSTEM' as const,
        startDate: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000), // 1 week from now
        minParticipants: 8,
        maxParticipants: 32,
      };

      const result = createTournamentSchema.safeParse(tournamentData);
      expect(result.success).toBe(true);
    });

    it('should reject tournaments with start date in the past', () => {
      const tournamentData = {
        name: 'Test Tournament',
        format: 'SWISS_SYSTEM' as const,
        startDate: new Date(Date.now() - 24 * 60 * 60 * 1000), // Yesterday
      };

      const result = createTournamentSchema.safeParse(tournamentData);
      expect(result.success).toBe(false);
    });
  });

  describe('Status Transitions', () => {
    it('should allow valid status transitions', () => {
      expect(validateStatusTransition('DRAFT', 'REGISTRATION_OPEN')).toBe(true);
      expect(validateStatusTransition('REGISTRATION_OPEN', 'REGISTRATION_CLOSED')).toBe(true);
      expect(validateStatusTransition('REGISTRATION_CLOSED', 'IN_PROGRESS')).toBe(true);
    });

    it('should reject invalid status transitions', () => {
      expect(validateStatusTransition('DRAFT', 'IN_PROGRESS')).toBe(false);
      expect(validateStatusTransition('COMPLETED', 'DRAFT')).toBe(false);
    });
  });
});
```

#### Acceptance Criteria
- [ ] All tournament operations tested
- [ ] Performance benchmarks met
- [ ] Error scenarios covered
- [ ] Status transitions validated
- [ ] Registration workflow tested

## Risk Assessment

### High Risk
- **Complex business rules**: Tournament format validation and status transitions
- **Data integrity**: Prevent invalid state changes during tournament lifecycle
- **Performance**: Tournament search and filtering with large datasets

### Medium Risk
- **Permission management**: Organizer role validation and access control
- **Date/time handling**: Timezone considerations for international tournaments

### Low Risk
- **UI/UX consistency**: Follow established design patterns
- **Form validation**: Standard input validation patterns

## Success Metrics

- Tournament creation success rate > 95%
- Tournament search response time < 500ms
- Registration success rate > 98%
- Zero invalid status transitions
- 85% test coverage for tournament operations

## Dependencies for Next Phase

This tournament management system enables:
- Participant registration and management
- Game scheduling and pairing generation
- Tournament-specific scoreboard display
- Standings calculation based on tournament results
