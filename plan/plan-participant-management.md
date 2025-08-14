# Participant Management Implementation Plan

## Feature Overview

Implement comprehensive participant management system for tournaments, including registration, profile management, withdrawal, and participant communication.

## Objectives

- Tournament participant registration and validation
- Participant profile display and management
- Registration status tracking and updates
- Participant communication and notifications
- Withdrawal and disqualification handling
- Participant search and filtering

## Technical Requirements

### Dependencies
- Authentication system (completed)
- User Profile Management (completed)
- Tournament Management (completed)
- Email notification system (optional)

### Constraints
- Participants must have valid profiles to register
- Registration must respect tournament limits and dates
- Withdrawal rules must be enforced based on tournament status
- Rating information must be captured at registration time

## Implementation Phases

### Phase 1: Enhanced Database Schema (1 day)

#### Tasks
1. **Extend Participant Model**
   - Add additional participant-specific fields
   - Enhance status tracking
   - Add communication preferences

2. **Create Notification System Schema**
   - Participant communication tracking
   - Email preferences and history

#### Database Changes

```prisma
// Enhanced TournamentParticipant model
model TournamentParticipant {
  id           String     @id @default(cuid())
  tournamentId String
  userId       String
  tournament   Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  user         User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  // Tournament-specific data
  seedRating   Int?       // Rating at time of registration
  currentRating Int?      // Updated rating during tournament
  score        Float      @default(0)
  tiebreaks    Json?      // Calculated tiebreak scores
  ranking      Int?
  
  // Registration data
  registeredAt DateTime   @default(now())
  status       ParticipantStatus @default(REGISTERED)
  paymentStatus PaymentStatus @default(NOT_REQUIRED)
  
  // Withdrawal information
  withdrawnAt  DateTime?
  withdrawReason String?
  withdrawnBy  String?    // User ID who processed withdrawal
  
  // Communication preferences
  emailNotifications Boolean @default(true)
  smsNotifications   Boolean @default(false)
  
  // Emergency contact (optional)
  emergencyContact   String?
  emergencyPhone     String?
  
  // Additional participant data
  specialRequests    String? @db.Text
  dietaryRestrictions String?
  accommodationNeeds String?
  
  // Relations
  notifications ParticipantNotification[]
  
  @@unique([tournamentId, userId])
  @@index([tournamentId, ranking])
  @@index([tournamentId, score])
  @@index([status])
  @@index([registeredAt])
}

enum ParticipantStatus {
  REGISTERED        // Initial registration
  CONFIRMED         // Registration confirmed by organizer
  CHECKED_IN        // Arrived at tournament
  PLAYING           // Currently active in tournament
  WITHDRAWN         // Voluntarily withdrawn
  DISQUALIFIED     // Disqualified by organizer
  NO_SHOW          // Didn't show up
}

enum PaymentStatus {
  NOT_REQUIRED     // Free tournament
  PENDING          // Payment required but not received
  PAID             // Payment completed
  REFUNDED         // Payment refunded
  WAIVED           // Payment waived by organizer
}

// Participant communication tracking
model ParticipantNotification {
  id            String @id @default(cuid())
  participantId String
  participant   TournamentParticipant @relation(fields: [participantId], references: [id], onDelete: Cascade)
  
  type          NotificationType
  subject       String
  message       String @db.Text
  channel       NotificationChannel
  
  sentAt        DateTime?
  deliveredAt   DateTime?
  readAt        DateTime?
  
  // Metadata
  createdAt     DateTime @default(now())
  createdBy     String   // User ID who sent notification
  
  @@index([participantId])
  @@index([type])
  @@index([sentAt])
}

enum NotificationType {
  REGISTRATION_CONFIRMATION
  TOURNAMENT_UPDATE
  ROUND_PAIRING
  RESULT_CONFIRMATION
  WITHDRAWAL_CONFIRMATION
  GENERAL_ANNOUNCEMENT
}

enum NotificationChannel {
  EMAIL
  SMS
  IN_APP
  PUSH
}

// Waiting list for full tournaments
model TournamentWaitingList {
  id           String     @id @default(cuid())
  tournamentId String
  userId       String
  tournament   Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  user         User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  position     Int        // Position in waiting list
  joinedAt     DateTime   @default(now())
  notifiedAt   DateTime?  // When user was notified of available spot
  expires      DateTime?  // When notification expires
  
  @@unique([tournamentId, userId])
  @@index([tournamentId, position])
}
```

#### Acceptance Criteria
- [ ] Enhanced participant model with all required fields
- [ ] Notification system schema created
- [ ] Waiting list functionality implemented
- [ ] Database constraints and indexes added
- [ ] Migration tested successfully

### Phase 2: Validation & Business Logic (2 days)

#### Tasks
1. **Create Participant Validation Schemas**
   - Registration validation with additional fields
   - Status update validation
   - Bulk operations validation

2. **Implement Participant Business Rules**
   - Registration eligibility checks
   - Status transition rules
   - Waiting list management

#### Implementation

```typescript
// src/lib/validations/participant.ts
import { z } from "zod";

export const participantRegistrationSchema = z.object({
  tournamentId: z.string(),
  emergencyContact: z.string().max(100).optional(),
  emergencyPhone: z.string().regex(/^\+?[\d\s\-\(\)]{10,20}$/).optional(),
  specialRequests: z.string().max(500).optional(),
  dietaryRestrictions: z.string().max(200).optional(),
  accommodationNeeds: z.string().max(200).optional(),
  emailNotifications: z.boolean().default(true),
  smsNotifications: z.boolean().default(false),
});

export const updateParticipantStatusSchema = z.object({
  participantId: z.string(),
  status: z.enum([
    "REGISTERED",
    "CONFIRMED", 
    "CHECKED_IN",
    "PLAYING",
    "WITHDRAWN",
    "DISQUALIFIED",
    "NO_SHOW"
  ]),
  reason: z.string().max(200).optional(),
  notifyParticipant: z.boolean().default(true),
});

export const bulkParticipantActionSchema = z.object({
  tournamentId: z.string(),
  participantIds: z.array(z.string()).min(1),
  action: z.enum(["CONFIRM", "CHECK_IN", "WITHDRAW", "DISQUALIFY"]),
  reason: z.string().max(200).optional(),
});

export const participantSearchSchema = z.object({
  tournamentId: z.string().optional(),
  status: z.enum([
    "REGISTERED",
    "CONFIRMED", 
    "CHECKED_IN",
    "PLAYING",
    "WITHDRAWN",
    "DISQUALIFIED",
    "NO_SHOW"
  ]).optional(),
  search: z.string().optional(), // Name, email, rating search
  ratingMin: z.number().int().min(0).max(3000).optional(),
  ratingMax: z.number().int().min(0).max(3000).optional(),
  registeredAfter: z.date().optional(),
  registeredBefore: z.date().optional(),
  country: z.string().length(2).optional(),
  club: z.string().optional(),
  page: z.number().int().min(1).default(1),
  limit: z.number().int().min(1).max(100).default(20),
  sortBy: z.enum(["name", "rating", "registeredAt", "score", "ranking"]).default("registeredAt"),
  sortOrder: z.enum(["asc", "desc"]).default("desc"),
});

export const withdrawParticipantSchema = z.object({
  participantId: z.string(),
  reason: z.string().min(10, "Please provide a detailed reason").max(500),
  refundRequested: z.boolean().default(false),
  notifyOthers: z.boolean().default(false), // Notify other participants about withdrawal
});

export const waitingListSchema = z.object({
  tournamentId: z.string(),
  userId: z.string().optional(), // For admin adding someone to waiting list
});

// Type exports
export type ParticipantRegistration = z.infer<typeof participantRegistrationSchema>;
export type UpdateParticipantStatus = z.infer<typeof updateParticipantStatusSchema>;
export type BulkParticipantAction = z.infer<typeof bulkParticipantActionSchema>;
export type ParticipantSearch = z.infer<typeof participantSearchSchema>;
export type WithdrawParticipant = z.infer<typeof withdrawParticipantSchema>;
export type WaitingListEntry = z.infer<typeof waitingListSchema>;

// Business rule constants
export const PARTICIPANT_STATUS_TRANSITIONS = {
  REGISTERED: ["CONFIRMED", "WITHDRAWN", "NO_SHOW"],
  CONFIRMED: ["CHECKED_IN", "WITHDRAWN", "NO_SHOW"],
  CHECKED_IN: ["PLAYING", "WITHDRAWN", "DISQUALIFIED"],
  PLAYING: ["WITHDRAWN", "DISQUALIFIED"],
  WITHDRAWN: [], // Final state
  DISQUALIFIED: [], // Final state
  NO_SHOW: [], // Final state
} as const;

export const validateParticipantStatusTransition = (
  currentStatus: string, 
  newStatus: string
): boolean => {
  const allowedTransitions = PARTICIPANT_STATUS_TRANSITIONS[
    currentStatus as keyof typeof PARTICIPANT_STATUS_TRANSITIONS
  ];
  return allowedTransitions.includes(newStatus as any);
};

// Registration eligibility checks
export const checkRegistrationEligibility = (
  tournament: any,
  user: any,
  existingParticipants: any[]
) => {
  const errors: string[] = [];

  // Check tournament status
  if (tournament.status !== "REGISTRATION_OPEN") {
    errors.push("Registration is not open for this tournament");
  }

  // Check if already registered
  const existingParticipant = existingParticipants.find(p => p.userId === user.id);
  if (existingParticipant) {
    errors.push("Already registered for this tournament");
  }

  // Check participant limit
  const activeParticipants = existingParticipants.filter(
    p => !["WITHDRAWN", "DISQUALIFIED", "NO_SHOW"].includes(p.status)
  );
  
  if (tournament.maxParticipants && activeParticipants.length >= tournament.maxParticipants) {
    errors.push("Tournament is full");
  }

  // Check registration deadline
  if (tournament.registrationEnd && new Date() > tournament.registrationEnd) {
    errors.push("Registration deadline has passed");
  }

  // Check minimum rating requirement (if set)
  if (tournament.minRating && (!user.profile?.fideRating && !user.profile?.localRating)) {
    errors.push("Rating information required for registration");
  }

  return {
    isEligible: errors.length === 0,
    errors,
  };
};
```

#### Acceptance Criteria
- [ ] All participant validation schemas created
- [ ] Business rules implemented and tested
- [ ] Status transition validation working
- [ ] Registration eligibility checks comprehensive
- [ ] Waiting list logic functional

### Phase 3: API Implementation (3 days)

#### Tasks
1. **Create Participant tRPC Router**
   - Registration and withdrawal endpoints
   - Status management
   - Search and filtering

2. **Implement Advanced Features**
   - Bulk participant operations
   - Waiting list management
   - Notification system

3. **Add Organizer Tools**
   - Participant management dashboard
   - Communication tools
   - Registration oversight

#### Implementation

```typescript
// src/server/api/routers/participant.ts
import { z } from "zod";
import { createTRPCRouter, protectedProcedure, publicProcedure, organizerProcedure } from "@/server/api/trpc";
import {
  participantRegistrationSchema,
  updateParticipantStatusSchema,
  bulkParticipantActionSchema,
  participantSearchSchema,
  withdrawParticipantSchema,
  waitingListSchema,
  validateParticipantStatusTransition,
  checkRegistrationEligibility,
} from "@/lib/validations/participant";
import { TRPCError } from "@trpc/server";

export const participantRouter = createTRPCRouter({
  // Register for tournament with enhanced data
  register: protectedProcedure
    .input(participantRegistrationSchema)
    .mutation(async ({ ctx, input }) => {
      const { tournamentId, ...participantData } = input;

      // Get tournament and existing participants
      const [tournament, existingParticipants, userProfile] = await Promise.all([
        ctx.db.tournament.findUnique({
          where: { id: tournamentId },
          include: { participants: true },
        }),
        ctx.db.tournamentParticipant.findMany({
          where: { tournamentId },
          include: { user: { include: { profile: true } } },
        }),
        ctx.db.profile.findUnique({
          where: { userId: ctx.session.user.id },
        }),
      ]);

      if (!tournament) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Tournament not found",
        });
      }

      if (!userProfile) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: "Please complete your profile before registering",
        });
      }

      // Check registration eligibility
      const eligibility = checkRegistrationEligibility(
        tournament,
        { id: ctx.session.user.id, profile: userProfile },
        existingParticipants
      );

      if (!eligibility.isEligible) {
        // If tournament is full, add to waiting list
        if (eligibility.errors.includes("Tournament is full")) {
          const waitingListPosition = await ctx.db.tournamentWaitingList.count({
            where: { tournamentId },
          });

          const waitingListEntry = await ctx.db.tournamentWaitingList.create({
            data: {
              tournamentId,
              userId: ctx.session.user.id,
              position: waitingListPosition + 1,
            },
          });

          return {
            success: false,
            waitingList: true,
            position: waitingListEntry.position,
            message: "Tournament is full. You have been added to the waiting list.",
          };
        }

        throw new TRPCError({
          code: "BAD_REQUEST",
          message: eligibility.errors.join(", "),
        });
      }

      // Get seed rating
      const seedRating = userProfile.fideRating || userProfile.localRating;

      // Create participant registration
      const participant = await ctx.db.tournamentParticipant.create({
        data: {
          tournamentId,
          userId: ctx.session.user.id,
          seedRating,
          currentRating: seedRating,
          ...participantData,
          status: "REGISTERED",
        },
        include: {
          user: {
            include: { profile: true },
          },
          tournament: {
            select: { name: true, startDate: true },
          },
        },
      });

      // Send registration confirmation notification
      await ctx.db.participantNotification.create({
        data: {
          participantId: participant.id,
          type: "REGISTRATION_CONFIRMATION",
          subject: `Registration confirmed for ${participant.tournament.name}`,
          message: `Your registration for ${participant.tournament.name} has been confirmed.`,
          channel: "EMAIL",
          createdBy: ctx.session.user.id,
        },
      });

      return { success: true, participant };
    }),

  // Withdraw from tournament
  withdraw: protectedProcedure
    .input(withdrawParticipantSchema)
    .mutation(async ({ ctx, input }) => {
      const participant = await ctx.db.tournamentParticipant.findUnique({
        where: { id: input.participantId },
        include: {
          tournament: true,
          user: true,
        },
      });

      if (!participant) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Participant not found",
        });
      }

      // Check if user can withdraw this participant
      const canWithdraw = participant.userId === ctx.session.user.id ||
                         participant.tournament.createdById === ctx.session.user.id ||
                         ctx.session.user.role === "ADMIN";

      if (!canWithdraw) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "You don't have permission to withdraw this participant",
        });
      }

      // Check if withdrawal is allowed
      if (["WITHDRAWN", "DISQUALIFIED", "NO_SHOW"].includes(participant.status)) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: "Participant is already withdrawn or disqualified",
        });
      }

      // Prevent withdrawal after tournament starts (unless admin/organizer)
      if (participant.tournament.status === "IN_PROGRESS" && 
          participant.userId === ctx.session.user.id) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: "Cannot withdraw after tournament has started",
        });
      }

      // Update participant status
      const updatedParticipant = await ctx.db.tournamentParticipant.update({
        where: { id: input.participantId },
        data: {
          status: "WITHDRAWN",
          withdrawnAt: new Date(),
          withdrawReason: input.reason,
          withdrawnBy: ctx.session.user.id,
        },
      });

      // Send withdrawal confirmation
      await ctx.db.participantNotification.create({
        data: {
          participantId: participant.id,
          type: "WITHDRAWAL_CONFIRMATION",
          subject: `Withdrawal confirmed for ${participant.tournament.name}`,
          message: `Your withdrawal from ${participant.tournament.name} has been processed. Reason: ${input.reason}`,
          channel: "EMAIL",
          createdBy: ctx.session.user.id,
        },
      });

      // Check waiting list and promote next participant
      const nextInLine = await ctx.db.tournamentWaitingList.findFirst({
        where: { tournamentId: participant.tournamentId },
        orderBy: { position: "asc" },
        include: { user: true },
      });

      if (nextInLine) {
        // Notify next participant about available spot
        await ctx.db.participantNotification.create({
          data: {
            participantId: nextInLine.userId,
            type: "TOURNAMENT_UPDATE",
            subject: `Spot available in ${participant.tournament.name}`,
            message: `A spot has opened up in ${participant.tournament.name}. You have 24 hours to confirm your registration.`,
            channel: "EMAIL",
            createdBy: "system",
          },
        });

        // Update waiting list entry with notification time
        await ctx.db.tournamentWaitingList.update({
          where: { id: nextInLine.id },
          data: {
            notifiedAt: new Date(),
            expires: new Date(Date.now() + 24 * 60 * 60 * 1000), // 24 hours
          },
        });
      }

      return updatedParticipant;
    }),

  // Update participant status (organizer only)
  updateStatus: organizerProcedure
    .input(updateParticipantStatusSchema)
    .mutation(async ({ ctx, input }) => {
      const participant = await ctx.db.tournamentParticipant.findUnique({
        where: { id: input.participantId },
        include: {
          tournament: {
            include: {
              organizers: { where: { userId: ctx.session.user.id } },
            },
          },
        },
      });

      if (!participant) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Participant not found",
        });
      }

      // Check organizer permissions
      const isOrganizer = participant.tournament.createdById === ctx.session.user.id ||
                         participant.tournament.organizers.length > 0 ||
                         ctx.session.user.role === "ADMIN";

      if (!isOrganizer) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "You don't have permission to update this participant",
        });
      }

      // Validate status transition
      if (!validateParticipantStatusTransition(participant.status, input.status)) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: `Cannot change status from ${participant.status} to ${input.status}`,
        });
      }

      // Update participant
      const updatedParticipant = await ctx.db.tournamentParticipant.update({
        where: { id: input.participantId },
        data: {
          status: input.status,
          ...(input.status === "WITHDRAWN" && {
            withdrawnAt: new Date(),
            withdrawReason: input.reason,
            withdrawnBy: ctx.session.user.id,
          }),
        },
      });

      // Send notification if requested
      if (input.notifyParticipant) {
        await ctx.db.participantNotification.create({
          data: {
            participantId: participant.id,
            type: "TOURNAMENT_UPDATE",
            subject: `Status updated for ${participant.tournament.name}`,
            message: `Your status has been updated to ${input.status}${input.reason ? `. Reason: ${input.reason}` : ''}`,
            channel: "EMAIL",
            createdBy: ctx.session.user.id,
          },
        });
      }

      return updatedParticipant;
    }),

  // Bulk participant actions
  bulkAction: organizerProcedure
    .input(bulkParticipantActionSchema)
    .mutation(async ({ ctx, input }) => {
      const { tournamentId, participantIds, action, reason } = input;

      // Verify tournament and permissions
      const tournament = await ctx.db.tournament.findUnique({
        where: { id: tournamentId },
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
          message: "You don't have permission to perform bulk actions",
        });
      }

      // Map actions to status
      const statusMap = {
        CONFIRM: "CONFIRMED",
        CHECK_IN: "CHECKED_IN",
        WITHDRAW: "WITHDRAWN",
        DISQUALIFY: "DISQUALIFIED",
      };

      const newStatus = statusMap[action];

      // Perform bulk update
      const updateData: any = { status: newStatus };
      
      if (action === "WITHDRAW" || action === "DISQUALIFY") {
        updateData.withdrawnAt = new Date();
        updateData.withdrawReason = reason;
        updateData.withdrawnBy = ctx.session.user.id;
      }

      const result = await ctx.db.tournamentParticipant.updateMany({
        where: {
          id: { in: participantIds },
          tournamentId,
        },
        data: updateData,
      });

      // Send notifications to affected participants
      const participants = await ctx.db.tournamentParticipant.findMany({
        where: { id: { in: participantIds } },
        include: { user: true },
      });

      const notifications = participants.map(participant => ({
        participantId: participant.id,
        type: "TOURNAMENT_UPDATE" as const,
        subject: `Status updated for ${tournament.name}`,
        message: `Your status has been updated to ${newStatus}${reason ? `. Reason: ${reason}` : ''}`,
        channel: "EMAIL" as const,
        createdBy: ctx.session.user.id,
      }));

      await ctx.db.participantNotification.createMany({
        data: notifications,
      });

      return {
        updated: result.count,
        action,
        status: newStatus,
      };
    }),

  // Search and filter participants
  search: publicProcedure
    .input(participantSearchSchema)
    .query(async ({ ctx, input }) => {
      const {
        tournamentId,
        status,
        search,
        ratingMin,
        ratingMax,
        registeredAfter,
        registeredBefore,
        country,
        club,
        page,
        limit,
        sortBy,
        sortOrder,
      } = input;

      const skip = (page - 1) * limit;

      // Build where clause
      const where: any = {
        ...(tournamentId && { tournamentId }),
        ...(status && { status }),
        ...(registeredAfter && { registeredAt: { gte: registeredAfter } }),
        ...(registeredBefore && { registeredAt: { lte: registeredBefore } }),
      };

      // Add rating filters
      if (ratingMin || ratingMax) {
        where.OR = [
          {
            user: {
              profile: {
                fideRating: {
                  ...(ratingMin && { gte: ratingMin }),
                  ...(ratingMax && { lte: ratingMax }),
                },
              },
            },
          },
          {
            user: {
              profile: {
                localRating: {
                  ...(ratingMin && { gte: ratingMin }),
                  ...(ratingMax && { lte: ratingMax }),
                },
              },
            },
          },
        ];
      }

      // Add search filter
      if (search) {
        where.OR = [
          ...(where.OR || []),
          { user: { name: { contains: search, mode: "insensitive" } } },
          { user: { email: { contains: search, mode: "insensitive" } } },
          { user: { profile: { firstName: { contains: search, mode: "insensitive" } } } },
          { user: { profile: { lastName: { contains: search, mode: "insensitive" } } } },
        ];
      }

      // Add location filters
      if (country) {
        where.user = { ...where.user, profile: { ...where.user?.profile, country } };
      }

      if (club) {
        where.user = {
          ...where.user,
          profile: {
            ...where.user?.profile,
            club: { contains: club, mode: "insensitive" },
          },
        };
      }

      // Build order by
      const orderBy: any = {};
      switch (sortBy) {
        case "name":
          orderBy.user = { name: sortOrder };
          break;
        case "rating":
          orderBy.seedRating = sortOrder;
          break;
        case "score":
          orderBy.score = sortOrder;
          break;
        case "ranking":
          orderBy.ranking = sortOrder;
          break;
        default:
          orderBy.registeredAt = sortOrder;
      }

      const [participants, total] = await Promise.all([
        ctx.db.tournamentParticipant.findMany({
          where,
          skip,
          take: limit,
          orderBy,
          include: {
            user: {
              select: {
                id: true,
                name: true,
                email: true,
                profile: {
                  select: {
                    firstName: true,
                    lastName: true,
                    fideRating: true,
                    localRating: true,
                    country: true,
                    club: true,
                  },
                },
              },
            },
            tournament: {
              select: {
                id: true,
                name: true,
                startDate: true,
                status: true,
              },
            },
          },
        }),
        ctx.db.tournamentParticipant.count({ where }),
      ]);

      return {
        participants,
        total,
        pages: Math.ceil(total / limit),
        currentPage: page,
      };
    }),

  // Get participant details
  getById: publicProcedure
    .input(z.object({ id: z.string() }))
    .query(async ({ ctx, input }) => {
      const participant = await ctx.db.tournamentParticipant.findUnique({
        where: { id: input.id },
        include: {
          user: {
            include: { profile: true },
          },
          tournament: true,
          notifications: {
            orderBy: { createdAt: "desc" },
            take: 10,
          },
        },
      });

      if (!participant) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Participant not found",
        });
      }

      return participant;
    }),

  // Get waiting list
  getWaitingList: publicProcedure
    .input(z.object({ tournamentId: z.string() }))
    .query(async ({ ctx, input }) => {
      const waitingList = await ctx.db.tournamentWaitingList.findMany({
        where: { tournamentId: input.tournamentId },
        orderBy: { position: "asc" },
        include: {
          user: {
            include: { profile: true },
          },
        },
      });

      return waitingList;
    }),

  // Join waiting list
  joinWaitingList: protectedProcedure
    .input(waitingListSchema)
    .mutation(async ({ ctx, input }) => {
      const { tournamentId } = input;

      // Check if already in waiting list
      const existingEntry = await ctx.db.tournamentWaitingList.findUnique({
        where: {
          tournamentId_userId: {
            tournamentId,
            userId: ctx.session.user.id,
          },
        },
      });

      if (existingEntry) {
        throw new TRPCError({
          code: "CONFLICT",
          message: "Already in waiting list",
        });
      }

      // Get next position
      const position = await ctx.db.tournamentWaitingList.count({
        where: { tournamentId },
      });

      const waitingListEntry = await ctx.db.tournamentWaitingList.create({
        data: {
          tournamentId,
          userId: ctx.session.user.id,
          position: position + 1,
        },
      });

      return waitingListEntry;
    }),
});
```

#### Acceptance Criteria
- [ ] All participant management operations implemented
- [ ] Bulk operations working correctly
- [ ] Waiting list functionality complete
- [ ] Notification system integrated
- [ ] Search and filtering comprehensive
- [ ] Status transitions validated

### Phase 4: Frontend Components (3 days)

#### Tasks
1. **Create Participant Management Components**
   - Participant registration form
   - Participant list/table
   - Status management interface

2. **Create Organizer Tools**
   - Participant dashboard
   - Bulk action controls
   - Communication interface

3. **Create Participant Experience Components**
   - Registration confirmation
   - Withdrawal interface
   - Notification center

#### Implementation

```typescript
// src/components/participant/participant-registration-form.tsx
"use client";

import { useState } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { Switch } from "@/components/ui/switch";
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/components/ui/form";
import { toast } from "@/components/ui/use-toast";
import { api } from "@/trpc/react";
import { participantRegistrationSchema, type ParticipantRegistration } from "@/lib/validations/participant";

interface ParticipantRegistrationFormProps {
  tournamentId: string;
  onSuccess?: () => void;
}

export function ParticipantRegistrationForm({ 
  tournamentId, 
  onSuccess 
}: ParticipantRegistrationFormProps) {
  const [isSubmitting, setIsSubmitting] = useState(false);

  const form = useForm<ParticipantRegistration>({
    resolver: zodResolver(participantRegistrationSchema),
    defaultValues: {
      tournamentId,
      emailNotifications: true,
      smsNotifications: false,
    },
  });

  const registerMutation = api.participant.register.useMutation({
    onSuccess: (result) => {
      if (result.success) {
        toast({ title: "Registration successful!" });
        onSuccess?.();
      } else if (result.waitingList) {
        toast({
          title: "Added to waiting list",
          description: `You are #${result.position} on the waiting list.`,
        });
      }
    },
    onError: (error) => {
      toast({ 
        title: "Registration failed", 
        description: error.message,
        variant: "destructive" 
      });
    },
  });

  const onSubmit = async (data: ParticipantRegistration) => {
    setIsSubmitting(true);
    try {
      await registerMutation.mutateAsync(data);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        {/* Emergency Contact */}
        <div className="space-y-4">
          <h3 className="text-lg font-medium">Emergency Contact (Optional)</h3>
          
          <div className="grid grid-cols-1 md:grid-cols-2 gap-4">
            <FormField
              control={form.control}
              name="emergencyContact"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Emergency Contact Name</FormLabel>
                  <FormControl>
                    <Input placeholder="Contact name" {...field} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="emergencyPhone"
              render={({ field }) => (
                <FormItem>
                  <FormLabel>Emergency Phone</FormLabel>
                  <FormControl>
                    <Input placeholder="+1234567890" {...field} />
                  </FormControl>
                  <FormMessage />
                </FormItem>
              )}
            />
          </div>
        </div>

        {/* Additional Information */}
        <div className="space-y-4">
          <h3 className="text-lg font-medium">Additional Information</h3>
          
          <FormField
            control={form.control}
            name="specialRequests"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Special Requests</FormLabel>
                <FormControl>
                  <Textarea 
                    placeholder="Any special requests or requirements..." 
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
            name="dietaryRestrictions"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Dietary Restrictions</FormLabel>
                <FormControl>
                  <Input placeholder="e.g., Vegetarian, Gluten-free" {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />

          <FormField
            control={form.control}
            name="accommodationNeeds"
            render={({ field }) => (
              <FormItem>
                <FormLabel>Accommodation Needs</FormLabel>
                <FormControl>
                  <Input placeholder="Any accessibility requirements" {...field} />
                </FormControl>
                <FormMessage />
              </FormItem>
            )}
          />
        </div>

        {/* Communication Preferences */}
        <div className="space-y-4">
          <h3 className="text-lg font-medium">Communication Preferences</h3>
          
          <div className="space-y-3">
            <FormField
              control={form.control}
              name="emailNotifications"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between">
                  <FormLabel>Email notifications</FormLabel>
                  <FormControl>
                    <Switch checked={field.value} onCheckedChange={field.onChange} />
                  </FormControl>
                </FormItem>
              )}
            />

            <FormField
              control={form.control}
              name="smsNotifications"
              render={({ field }) => (
                <FormItem className="flex items-center justify-between">
                  <FormLabel>SMS notifications</FormLabel>
                  <FormControl>
                    <Switch checked={field.value} onCheckedChange={field.onChange} />
                  </FormControl>
                </FormItem>
              )}
            />
          </div>
        </div>

        <Button type="submit" disabled={isSubmitting} className="w-full">
          {isSubmitting ? "Registering..." : "Register for Tournament"}
        </Button>
      </form>
    </Form>
  );
}
```

```typescript
// src/components/participant/participant-management-table.tsx
"use client";

import { useState } from "react";
import { format } from "date-fns";
import { MoreHorizontal, Mail, UserCheck, UserX } from "lucide-react";

import { Button } from "@/components/ui/button";
import { Badge } from "@/components/ui/badge";
import { Checkbox } from "@/components/ui/checkbox";
import {
  DropdownMenu,
  DropdownMenuContent,
  DropdownMenuItem,
  DropdownMenuSeparator,
  DropdownMenuTrigger,
} from "@/components/ui/dropdown-menu";
import {
  Table,
  TableBody,
  TableCell,
  TableHead,
  TableHeader,
  TableRow,
} from "@/components/ui/table";
import { api } from "@/trpc/react";
import type { ParticipantSearch } from "@/lib/validations/participant";

interface ParticipantManagementTableProps {
  tournamentId: string;
  filters: ParticipantSearch;
  onRefresh?: () => void;
}

export function ParticipantManagementTable({ 
  tournamentId, 
  filters, 
  onRefresh 
}: ParticipantManagementTableProps) {
  const [selectedParticipants, setSelectedParticipants] = useState<string[]>([]);

  const { data, isLoading } = api.participant.search.useQuery({
    ...filters,
    tournamentId,
  });

  const updateStatusMutation = api.participant.updateStatus.useMutation({
    onSuccess: () => {
      onRefresh?.();
    },
  });

  const bulkActionMutation = api.participant.bulkAction.useMutation({
    onSuccess: () => {
      setSelectedParticipants([]);
      onRefresh?.();
    },
  });

  const getStatusBadge = (status: string) => {
    const variants: Record<string, "default" | "secondary" | "destructive" | "outline"> = {
      REGISTERED: "default",
      CONFIRMED: "secondary",
      CHECKED_IN: "outline",
      PLAYING: "secondary",
      WITHDRAWN: "destructive",
      DISQUALIFIED: "destructive",
      NO_SHOW: "destructive",
    };

    return (
      <Badge variant={variants[status] || "default"}>
        {status.replace('_', ' ')}
      </Badge>
    );
  };

  const handleSelectAll = (checked: boolean) => {
    if (checked) {
      setSelectedParticipants(data?.participants.map(p => p.id) || []);
    } else {
      setSelectedParticipants([]);
    }
  };

  const handleSelectParticipant = (participantId: string, checked: boolean) => {
    if (checked) {
      setSelectedParticipants(prev => [...prev, participantId]);
    } else {
      setSelectedParticipants(prev => prev.filter(id => id !== participantId));
    }
  };

  const handleBulkAction = async (action: string) => {
    if (selectedParticipants.length === 0) return;

    await bulkActionMutation.mutateAsync({
      tournamentId,
      participantIds: selectedParticipants,
      action: action as any,
    });
  };

  if (isLoading) {
    return <div>Loading participants...</div>;
  }

  if (!data?.participants.length) {
    return <div>No participants found.</div>;
  }

  return (
    <div className="space-y-4">
      {/* Bulk Actions */}
      {selectedParticipants.length > 0 && (
        <div className="flex items-center gap-2 p-4 bg-muted rounded-lg">
          <span className="text-sm font-medium">
            {selectedParticipants.length} selected
          </span>
          <Button
            size="sm"
            onClick={() => handleBulkAction("CONFIRM")}
          >
            <UserCheck className="mr-1 h-4 w-4" />
            Confirm
          </Button>
          <Button
            size="sm"
            variant="outline"
            onClick={() => handleBulkAction("CHECK_IN")}
          >
            Check In
          </Button>
          <Button
            size="sm"
            variant="destructive"
            onClick={() => handleBulkAction("WITHDRAW")}
          >
            <UserX className="mr-1 h-4 w-4" />
            Withdraw
          </Button>
        </div>
      )}

      {/* Participants Table */}
      <Table>
        <TableHeader>
          <TableRow>
            <TableHead className="w-12">
              <Checkbox
                checked={selectedParticipants.length === data.participants.length}
                onCheckedChange={handleSelectAll}
              />
            </TableHead>
            <TableHead>Name</TableHead>
            <TableHead>Rating</TableHead>
            <TableHead>Status</TableHead>
            <TableHead>Registered</TableHead>
            <TableHead>Score</TableHead>
            <TableHead>Ranking</TableHead>
            <TableHead className="w-12"></TableHead>
          </TableRow>
        </TableHeader>
        <TableBody>
          {data.participants.map((participant) => (
            <TableRow key={participant.id}>
              <TableCell>
                <Checkbox
                  checked={selectedParticipants.includes(participant.id)}
                  onCheckedChange={(checked) => 
                    handleSelectParticipant(participant.id, checked as boolean)
                  }
                />
              </TableCell>
              <TableCell>
                <div>
                  <div className="font-medium">
                    {participant.user.profile?.firstName} {participant.user.profile?.lastName}
                  </div>
                  <div className="text-sm text-muted-foreground">
                    {participant.user.email}
                  </div>
                </div>
              </TableCell>
              <TableCell>
                {participant.seedRating || 'Unrated'}
              </TableCell>
              <TableCell>
                {getStatusBadge(participant.status)}
              </TableCell>
              <TableCell>
                {format(new Date(participant.registeredAt), 'MMM dd, yyyy')}
              </TableCell>
              <TableCell>
                {participant.score}
              </TableCell>
              <TableCell>
                {participant.ranking || '-'}
              </TableCell>
              <TableCell>
                <DropdownMenu>
                  <DropdownMenuTrigger asChild>
                    <Button variant="ghost" size="sm">
                      <MoreHorizontal className="h-4 w-4" />
                    </Button>
                  </DropdownMenuTrigger>
                  <DropdownMenuContent align="end">
                    <DropdownMenuItem>
                      <Mail className="mr-2 h-4 w-4" />
                      Send Message
                    </DropdownMenuItem>
                    <DropdownMenuSeparator />
                    <DropdownMenuItem
                      onClick={() => updateStatusMutation.mutate({
                        participantId: participant.id,
                        status: "CONFIRMED",
                      })}
                    >
                      Confirm Registration
                    </DropdownMenuItem>
                    <DropdownMenuItem
                      onClick={() => updateStatusMutation.mutate({
                        participantId: participant.id,
                        status: "CHECKED_IN",
                      })}
                    >
                      Check In
                    </DropdownMenuItem>
                    <DropdownMenuSeparator />
                    <DropdownMenuItem
                      className="text-destructive"
                      onClick={() => updateStatusMutation.mutate({
                        participantId: participant.id,
                        status: "WITHDRAWN",
                        reason: "Withdrawn by organizer",
                      })}
                    >
                      Withdraw
                    </DropdownMenuItem>
                  </DropdownMenuContent>
                </DropdownMenu>
              </TableCell>
            </TableRow>
          ))}
        </TableBody>
      </Table>

      {/* Pagination */}
      {data.pages > 1 && (
        <div className="flex items-center justify-between">
          <div className="text-sm text-muted-foreground">
            Showing {(data.currentPage - 1) * filters.limit + 1} to{' '}
            {Math.min(data.currentPage * filters.limit, data.total)} of {data.total} participants
          </div>
          {/* Add pagination controls here */}
        </div>
      )}
    </div>
  );
}
```

#### Acceptance Criteria
- [ ] Registration form with all fields working
- [ ] Participant management table functional
- [ ] Bulk operations interface complete
- [ ] Status update controls working
- [ ] Search and filtering UI implemented
- [ ] Notification system integrated

### Phase 5: Testing & Integration (1 day)

#### Tasks
1. **Unit Tests**
   - Participant validation functions
   - Status transition logic
   - Registration eligibility checks

2. **Integration Tests**
   - Registration workflow
   - Withdrawal process
   - Bulk operations

3. **Performance Testing**
   - Large participant list handling
   - Search performance optimization

#### Test Implementation

```typescript
// __tests__/participant/participant-api.test.ts
import { describe, expect, it, beforeEach } from '@jest/globals';
import { participantRegistrationSchema, validateParticipantStatusTransition } from '@/lib/validations/participant';

describe('Participant Management', () => {
  describe('Registration Validation', () => {
    it('should validate participant registration data', () => {
      const registrationData = {
        tournamentId: 'tournament-1',
        emergencyContact: 'John Doe',
        emergencyPhone: '+1234567890',
        emailNotifications: true,
        smsNotifications: false,
      };

      const result = participantRegistrationSchema.safeParse(registrationData);
      expect(result.success).toBe(true);
    });

    it('should reject invalid phone numbers', () => {
      const registrationData = {
        tournamentId: 'tournament-1',
        emergencyPhone: 'invalid-phone',
      };

      const result = participantRegistrationSchema.safeParse(registrationData);
      expect(result.success).toBe(false);
    });
  });

  describe('Status Transitions', () => {
    it('should allow valid status transitions', () => {
      expect(validateParticipantStatusTransition('REGISTERED', 'CONFIRMED')).toBe(true);
      expect(validateParticipantStatusTransition('CONFIRMED', 'CHECKED_IN')).toBe(true);
      expect(validateParticipantStatusTransition('CHECKED_IN', 'PLAYING')).toBe(true);
    });

    it('should reject invalid status transitions', () => {
      expect(validateParticipantStatusTransition('REGISTERED', 'PLAYING')).toBe(false);
      expect(validateParticipantStatusTransition('WITHDRAWN', 'CONFIRMED')).toBe(false);
    });
  });
});
```

#### Acceptance Criteria
- [ ] All participant operations tested
- [ ] Status transitions validated
- [ ] Registration eligibility checked
- [ ] Performance benchmarks met
- [ ] Error scenarios covered

## Risk Assessment

### High Risk
- **Complex status workflows**: Ensure status transitions are properly validated
- **Data consistency**: Maintain participant data integrity across operations
- **Notification reliability**: Ensure critical notifications are delivered

### Medium Risk
- **Bulk operations performance**: Optimize for large participant counts
- **Waiting list management**: Handle edge cases in waiting list promotion

### Low Risk
- **UI/UX consistency**: Follow established design patterns
- **Form validation**: Standard input validation patterns

## Success Metrics

- Registration success rate > 95%
- Participant search response time < 400ms
- Status update success rate > 98%
- Zero data consistency issues
- 85% test coverage for participant operations

## Dependencies for Next Phase

This participant management system enables:
- Game scheduling with confirmed participants
- Tournament scoreboard with participant data
- Standings calculation with participant scores
- Communication system for tournament updates
