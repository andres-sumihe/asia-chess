# Game Management Implementation Plan

## Feature Overview

Implement comprehensive game management system for handling chess game creation, result submission, validation, and PGN storage within tournaments.

## Objectives

- Game creation and pairing generation
- Game result submission and validation
- PGN (Portable Game Notation) storage and management
- Game status tracking and updates
- Result validation and conflict resolution
- Game history and analytics

## Technical Requirements

### Dependencies
- Tournament Management (completed)
- Participant Management (completed)
- Authentication system (completed)
- Chess.js library for PGN validation
- Date/time utilities for game timing

### Constraints
- Games must belong to valid tournament rounds
- Results can only be submitted by participants or organizers
- PGN validation must be performed before storage
- Game timing must be tracked accurately

## Implementation Phases

### Phase 1: Enhanced Database Schema (2 days)

#### Tasks
1. **Extend Game and Round Models**
   - Add comprehensive game tracking fields
   - Enhance result validation
   - Add timing and clock management

2. **Create Game History Schema**
   - Move tracking and position analysis
   - Game annotation storage
   - Result verification audit trail

#### Database Changes

```prisma
model Round {
  id           String     @id @default(cuid())
  tournamentId String
  roundNumber  Int
  tournament   Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  games        Game[]
  
  // Round timing
  startTime    DateTime?
  endTime      DateTime?
  plannedStart DateTime?
  timeControl  String?    // Override tournament time control
  
  // Round status
  status       RoundStatus @default(PENDING)
  pairingsGenerated Boolean @default(false)
  allGamesCompleted Boolean @default(false)
  
  // Pairing settings
  pairingMethod String?   @default("Swiss") // Swiss, Manual, Random
  
  createdAt    DateTime   @default(now())
  updatedAt    DateTime   @updatedAt
  
  @@unique([tournamentId, roundNumber])
  @@index([tournamentId, status])
}

model Game {
  id          String      @id @default(cuid())
  roundId     String
  round       Round       @relation(fields: [roundId], references: [id], onDelete: Cascade)
  
  // Players
  whiteId     String
  blackId     String
  white       User        @relation("WhitePlayer", fields: [whiteId], references: [id])
  black       User        @relation("BlackPlayer", fields: [blackId], references: [id])
  
  // Game result
  result      GameResult?
  resultBy    ResultMethod?
  resultConfirmed Boolean  @default(false)
  
  // Game data
  pgn         String?     @db.Text
  fen         String?     // Current position
  moves       Json?       // Array of moves with timestamps
  moveCount   Int         @default(0)
  
  // Timing
  startTime   DateTime?
  endTime     DateTime?
  duration    Int?        // Game duration in seconds
  whiteTime   Int?        // White's remaining time in seconds
  blackTime   Int?        // Black's remaining time in seconds
  
  // Game status
  status      GameStatus  @default(SCHEDULED)
  board       Int?        // Board number for the round
  
  // Result submission tracking
  whiteResult GameResult?
  blackResult GameResult?
  whiteSubmittedAt DateTime?
  blackSubmittedAt DateTime?
  
  // Organizer verification
  verifiedBy  String?     // User ID of organizer who verified
  verifiedAt  DateTime?
  notes       String?     @db.Text
  
  // Relations
  resultHistory GameResultHistory[]
  
  createdAt   DateTime    @default(now())
  updatedAt   DateTime    @updatedAt
  
  @@unique([roundId, whiteId, blackId])
  @@index([roundId])
  @@index([whiteId])
  @@index([blackId])
  @@index([status])
  @@index([result])
}

enum RoundStatus {
  PENDING      // Not started yet
  ACTIVE       // In progress
  COMPLETED    // All games finished
  CANCELLED    // Round cancelled
}

enum GameStatus {
  SCHEDULED    // Game scheduled but not started
  IN_PROGRESS  // Game currently being played
  COMPLETED    // Game finished
  ADJOURNED    // Game paused (rare in modern tournaments)
  CANCELLED    // Game cancelled
  FORFEIT      // One player forfeited
}

enum GameResult {
  WHITE_WINS
  BLACK_WINS
  DRAW
  WHITE_FORFEIT
  BLACK_FORFEIT
  DOUBLE_FORFEIT
  CANCELLED
}

enum ResultMethod {
  CHECKMATE
  RESIGNATION
  TIME_FORFEIT
  DRAW_AGREEMENT
  STALEMATE
  THREEFOLD_REPETITION
  FIFTY_MOVE_RULE
  INSUFFICIENT_MATERIAL
  ARBITER_DECISION
  NO_SHOW
}

// Game result submission history and conflicts
model GameResultHistory {
  id          String      @id @default(cuid())
  gameId      String
  game        Game        @relation(fields: [gameId], references: [id], onDelete: Cascade)
  
  submittedBy String      // User ID
  submitter   User        @relation(fields: [submittedBy], references: [id])
  
  result      GameResult
  resultBy    ResultMethod?
  pgn         String?     @db.Text
  notes       String?
  
  submittedAt DateTime    @default(now())
  isActive    Boolean     @default(false) // Current active result
  
  @@index([gameId])
  @@index([submittedBy])
  @@index([submittedAt])
}

// Add new relations to existing models
model User {
  // Existing fields...
  
  // Game relations
  whiteGames     Game[]    @relation("WhitePlayer")
  blackGames     Game[]    @relation("BlackPlayer")
  resultHistory  GameResultHistory[]
  
  // Tournament organizer relation
  organizedTournaments TournamentOrganizer[]
}

model Tournament {
  // Existing fields...
  
  // Add organizers relation
  organizers     TournamentOrganizer[]
  
  // Add waiting list relation
  waitingList    TournamentWaitingList[]
}
```

#### Acceptance Criteria
- [ ] Enhanced game model with timing and status tracking
- [ ] Result history and conflict resolution system
- [ ] Round management with pairing generation
- [ ] Database constraints and indexes optimized
- [ ] Migration tested successfully

### Phase 2: Validation & Business Logic (2 days)

#### Tasks
1. **Create Game Validation Schemas**
   - Game result submission validation
   - PGN format validation
   - Move validation and timing

2. **Implement Pairing Algorithms**
   - Swiss system pairing logic
   - Round robin pairing generation
   - Manual pairing support

#### Implementation

```typescript
// src/lib/validations/game.ts
import { z } from "zod";

export const submitGameResultSchema = z.object({
  gameId: z.string(),
  result: z.enum([
    "WHITE_WINS", 
    "BLACK_WINS", 
    "DRAW", 
    "WHITE_FORFEIT", 
    "BLACK_FORFEIT", 
    "DOUBLE_FORFEIT",
    "CANCELLED"
  ]),
  resultBy: z.enum([
    "CHECKMATE",
    "RESIGNATION", 
    "TIME_FORFEIT",
    "DRAW_AGREEMENT",
    "STALEMATE",
    "THREEFOLD_REPETITION",
    "FIFTY_MOVE_RULE",
    "INSUFFICIENT_MATERIAL",
    "ARBITER_DECISION",
    "NO_SHOW"
  ]).optional(),
  pgn: z.string().optional(),
  notes: z.string().max(500).optional(),
});

export const updateGameStatusSchema = z.object({
  gameId: z.string(),
  status: z.enum(["SCHEDULED", "IN_PROGRESS", "COMPLETED", "ADJOURNED", "CANCELLED", "FORFEIT"]),
  startTime: z.date().optional(),
  endTime: z.date().optional(),
});

export const createManualGameSchema = z.object({
  roundId: z.string(),
  whiteId: z.string(),
  blackId: z.string(),
  board: z.number().int().positive().optional(),
  notes: z.string().max(200).optional(),
});

export const gameSearchSchema = z.object({
  tournamentId: z.string().optional(),
  roundId: z.string().optional(),
  playerId: z.string().optional(),
  status: z.enum(["SCHEDULED", "IN_PROGRESS", "COMPLETED", "ADJOURNED", "CANCELLED", "FORFEIT"]).optional(),
  result: z.enum([
    "WHITE_WINS", 
    "BLACK_WINS", 
    "DRAW", 
    "WHITE_FORFEIT", 
    "BLACK_FORFEIT", 
    "DOUBLE_FORFEIT",
    "CANCELLED"
  ]).optional(),
  dateFrom: z.date().optional(),
  dateTo: z.date().optional(),
  page: z.number().int().min(1).default(1),
  limit: z.number().int().min(1).max(100).default(20),
});

export const generatePairingsSchema = z.object({
  roundId: z.string(),
  method: z.enum(["SWISS", "ROUND_ROBIN", "MANUAL"]).default("SWISS"),
  avoidRematch: z.boolean().default(true),
  balanceColors: z.boolean().default(true),
  manualPairings: z.array(z.object({
    whiteId: z.string(),
    blackId: z.string(),
    board: z.number().int().positive().optional(),
  })).optional(),
});

// Type exports
export type SubmitGameResult = z.infer<typeof submitGameResultSchema>;
export type UpdateGameStatus = z.infer<typeof updateGameStatusSchema>;
export type CreateManualGame = z.infer<typeof createManualGameSchema>;
export type GameSearch = z.infer<typeof gameSearchSchema>;
export type GeneratePairings = z.infer<typeof generatePairingsSchema>;

// PGN validation utilities
export const validatePGN = (pgn: string): { isValid: boolean; error?: string } => {
  try {
    // Basic PGN format validation
    if (!pgn.trim()) {
      return { isValid: true }; // Empty PGN is allowed
    }

    // Check for basic PGN structure
    const hasResult = /\s+(1-0|0-1|1\/2-1\/2|\*)\s*$/.test(pgn);
    const hasValidMoves = /^\s*(\[.*?\]\s*)*\s*(\d+\.\s*[a-zA-Z0-9+#=\-O]+\s*[a-zA-Z0-9+#=\-O]*\s*)*/.test(pgn);

    if (!hasValidMoves) {
      return { isValid: false, error: "Invalid PGN format" };
    }

    return { isValid: true };
  } catch (error) {
    return { isValid: false, error: "PGN validation failed" };
  }
};

// Game result validation rules
export const validateGameResult = (
  result: string, 
  submittedBy: string, 
  whiteId: string, 
  blackId: string
): { isValid: boolean; errors: string[] } => {
  const errors: string[] = [];

  // Check if submitter is a participant
  if (submittedBy !== whiteId && submittedBy !== blackId) {
    errors.push("Only game participants can submit results");
  }

  // Validate result consistency
  if (result === "WHITE_WINS" && submittedBy === blackId) {
    // Black player claiming white wins - likely accurate
  } else if (result === "BLACK_WINS" && submittedBy === whiteId) {
    // White player claiming black wins - likely accurate
  } else if (result === "DRAW") {
    // Draw claims are generally accepted
  } else if (result.includes("FORFEIT")) {
    // Forfeit claims need verification
  }

  return {
    isValid: errors.length === 0,
    errors,
  };
};

// Swiss pairing utilities
export class SwissPairingEngine {
  static generatePairings(participants: any[], round: number): any[] {
    // Sort participants by score (descending), then by rating (descending)
    const sortedParticipants = participants
      .filter(p => p.status === "CONFIRMED" || p.status === "CHECKED_IN")
      .sort((a, b) => {
        if (a.score !== b.score) return b.score - a.score;
        return (b.seedRating || 0) - (a.seedRating || 0);
      });

    const pairings: any[] = [];
    const unpaired = [...sortedParticipants];
    const used = new Set<string>();

    while (unpaired.length >= 2) {
      const player1 = unpaired.find(p => !used.has(p.userId));
      if (!player1) break;

      used.add(player1.userId);
      
      // Find best opponent for player1
      const opponent = unpaired.find(p => 
        !used.has(p.userId) && 
        p.userId !== player1.userId &&
        !this.havePlayedBefore(player1.userId, p.userId, round)
      );

      if (opponent) {
        used.add(opponent.userId);
        
        // Determine colors (alternate or balance)
        const player1AsWhite = this.shouldPlayWhite(player1, opponent, round);
        
        pairings.push({
          whiteId: player1AsWhite ? player1.userId : opponent.userId,
          blackId: player1AsWhite ? opponent.userId : player1.userId,
          board: pairings.length + 1,
        });
      }
    }

    // Handle bye if odd number of players
    const remainingPlayer = unpaired.find(p => !used.has(p.userId));
    if (remainingPlayer) {
      pairings.push({
        whiteId: remainingPlayer.userId,
        blackId: null, // Bye
        board: pairings.length + 1,
      });
    }

    return pairings;
  }

  private static havePlayedBefore(player1Id: string, player2Id: string, currentRound: number): boolean {
    // This would check game history - simplified for now
    return false;
  }

  private static shouldPlayWhite(player1: any, player2: any, round: number): boolean {
    // Color balancing logic - simplified
    return Math.random() > 0.5;
  }
}
```

#### Acceptance Criteria
- [ ] Game validation schemas comprehensive
- [ ] PGN validation working correctly
- [ ] Swiss pairing algorithm implemented
- [ ] Result validation rules enforced
- [ ] Business logic tested thoroughly

### Phase 3: API Implementation (3 days)

#### Tasks
1. **Create Game tRPC Router**
   - Game CRUD operations
   - Result submission and verification
   - Pairing generation

2. **Implement Round Management**
   - Round creation and management
   - Automatic pairing generation
   - Round completion tracking

3. **Add Game Analytics**
   - Game statistics
   - Player performance tracking
   - Tournament progress monitoring

#### Implementation

```typescript
// src/server/api/routers/game.ts
import { z } from "zod";
import { createTRPCRouter, protectedProcedure, publicProcedure, organizerProcedure } from "@/server/api/trpc";
import {
  submitGameResultSchema,
  updateGameStatusSchema,
  createManualGameSchema,
  gameSearchSchema,
  generatePairingsSchema,
  validatePGN,
  validateGameResult,
  SwissPairingEngine,
} from "@/lib/validations/game";
import { TRPCError } from "@trpc/server";

export const gameRouter = createTRPCRouter({
  // Get games with filtering
  search: publicProcedure
    .input(gameSearchSchema)
    .query(async ({ ctx, input }) => {
      const {
        tournamentId,
        roundId,
        playerId,
        status,
        result,
        dateFrom,
        dateTo,
        page,
        limit,
      } = input;

      const skip = (page - 1) * limit;

      const where: any = {
        ...(roundId && { roundId }),
        ...(status && { status }),
        ...(result && { result }),
        ...(dateFrom && { startTime: { gte: dateFrom } }),
        ...(dateTo && { endTime: { lte: dateTo } }),
      };

      // Tournament filter
      if (tournamentId) {
        where.round = { tournamentId };
      }

      // Player filter
      if (playerId) {
        where.OR = [
          { whiteId: playerId },
          { blackId: playerId },
        ];
      }

      const [games, total] = await Promise.all([
        ctx.db.game.findMany({
          where,
          skip,
          take: limit,
          orderBy: [
            { round: { roundNumber: "asc" } },
            { board: "asc" },
          ],
          include: {
            white: {
              select: {
                id: true,
                name: true,
                profile: {
                  select: { firstName: true, lastName: true, fideRating: true, localRating: true },
                },
              },
            },
            black: {
              select: {
                id: true,
                name: true,
                profile: {
                  select: { firstName: true, lastName: true, fideRating: true, localRating: true },
                },
              },
            },
            round: {
              select: {
                id: true,
                roundNumber: true,
                tournament: {
                  select: { id: true, name: true },
                },
              },
            },
          },
        }),
        ctx.db.game.count({ where }),
      ]);

      return {
        games,
        total,
        pages: Math.ceil(total / limit),
        currentPage: page,
      };
    }),

  // Get game by ID with full details
  getById: publicProcedure
    .input(z.object({ id: z.string() }))
    .query(async ({ ctx, input }) => {
      const game = await ctx.db.game.findUnique({
        where: { id: input.id },
        include: {
          white: {
            select: {
              id: true,
              name: true,
              profile: {
                select: { firstName: true, lastName: true, fideRating: true, localRating: true },
              },
            },
          },
          black: {
            select: {
              id: true,
              name: true,
              profile: {
                select: { firstName: true, lastName: true, fideRating: true, localRating: true },
              },
            },
          },
          round: {
            include: {
              tournament: {
                select: { id: true, name: true, format: true, status: true },
              },
            },
          },
          resultHistory: {
            include: {
              submitter: {
                select: { name: true },
              },
            },
            orderBy: { submittedAt: "desc" },
          },
        },
      });

      if (!game) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Game not found",
        });
      }

      return game;
    }),

  // Submit game result
  submitResult: protectedProcedure
    .input(submitGameResultSchema)
    .mutation(async ({ ctx, input }) => {
      const { gameId, result, resultBy, pgn, notes } = input;

      // Get game with tournament info
      const game = await ctx.db.game.findUnique({
        where: { id: gameId },
        include: {
          round: {
            include: { tournament: true },
          },
        },
      });

      if (!game) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Game not found",
        });
      }

      // Validate user can submit result
      const canSubmit = game.whiteId === ctx.session.user.id ||
                       game.blackId === ctx.session.user.id ||
                       game.round.tournament.createdById === ctx.session.user.id ||
                       ctx.session.user.role === "ADMIN";

      if (!canSubmit) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "You don't have permission to submit this result",
        });
      }

      // Validate game can receive results
      if (game.status === "COMPLETED") {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: "Game is already completed",
        });
      }

      // Validate PGN if provided
      if (pgn) {
        const pgnValidation = validatePGN(pgn);
        if (!pgnValidation.isValid) {
          throw new TRPCError({
            code: "BAD_REQUEST",
            message: `Invalid PGN: ${pgnValidation.error}`,
          });
        }
      }

      // Validate result
      const resultValidation = validateGameResult(
        result,
        ctx.session.user.id,
        game.whiteId,
        game.blackId
      );

      if (!resultValidation.isValid) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: resultValidation.errors.join(", "),
        });
      }

      // Check for existing result submissions
      const existingWhiteResult = game.whiteResult;
      const existingBlackResult = game.blackResult;
      const isWhitePlayer = game.whiteId === ctx.session.user.id;
      const isBlackPlayer = game.blackId === ctx.session.user.id;

      // Update game with result submission
      const updateData: any = {
        status: "COMPLETED",
        endTime: new Date(),
      };

      if (isWhitePlayer) {
        updateData.whiteResult = result;
        updateData.whiteSubmittedAt = new Date();
      } else if (isBlackPlayer) {
        updateData.blackResult = result;
        updateData.blackSubmittedAt = new Date();
      }

      // If both players have submitted and results match, confirm the result
      const otherPlayerResult = isWhitePlayer ? existingBlackResult : existingWhiteResult;
      if (otherPlayerResult === result) {
        updateData.result = result;
        updateData.resultBy = resultBy;
        updateData.resultConfirmed = true;
        updateData.verifiedAt = new Date();
        if (pgn) updateData.pgn = pgn;
      } else if (otherPlayerResult && otherPlayerResult !== result) {
        // Conflicting results - needs organizer verification
        updateData.status = "IN_PROGRESS"; // Keep as in progress
        updateData.notes = "Conflicting results submitted - organizer verification required";
      } else {
        // First result submission
        updateData.result = result;
        updateData.resultBy = resultBy;
        updateData.resultConfirmed = false; // Wait for confirmation
        if (pgn) updateData.pgn = pgn;
      }

      // Update game and create history entry in transaction
      const updatedGame = await ctx.db.$transaction(async (tx) => {
        // Update game
        const game = await tx.game.update({
          where: { id: gameId },
          data: updateData,
        });

        // Create result history entry
        await tx.gameResultHistory.create({
          data: {
            gameId,
            submittedBy: ctx.session.user.id,
            result,
            resultBy,
            pgn,
            notes,
            isActive: updateData.resultConfirmed || false,
          },
        });

        return game;
      });

      return updatedGame;
    }),

  // Verify/override game result (organizer only)
  verifyResult: organizerProcedure
    .input(z.object({
      gameId: z.string(),
      result: z.enum([
        "WHITE_WINS", 
        "BLACK_WINS", 
        "DRAW", 
        "WHITE_FORFEIT", 
        "BLACK_FORFEIT", 
        "DOUBLE_FORFEIT",
        "CANCELLED"
      ]),
      resultBy: z.enum([
        "CHECKMATE",
        "RESIGNATION", 
        "TIME_FORFEIT",
        "DRAW_AGREEMENT",
        "STALEMATE",
        "THREEFOLD_REPETITION",
        "FIFTY_MOVE_RULE",
        "INSUFFICIENT_MATERIAL",
        "ARBITER_DECISION",
        "NO_SHOW"
      ]).optional(),
      notes: z.string().max(500).optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      const { gameId, result, resultBy, notes } = input;

      // Verify organizer permissions
      const game = await ctx.db.game.findUnique({
        where: { id: gameId },
        include: {
          round: {
            include: {
              tournament: {
                include: {
                  organizers: { where: { userId: ctx.session.user.id } },
                },
              },
            },
          },
        },
      });

      if (!game) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Game not found",
        });
      }

      const isOrganizer = game.round.tournament.createdById === ctx.session.user.id ||
                         game.round.tournament.organizers.length > 0 ||
                         ctx.session.user.role === "ADMIN";

      if (!isOrganizer) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "You don't have permission to verify this result",
        });
      }

      // Update game with verified result
      const updatedGame = await ctx.db.game.update({
        where: { id: gameId },
        data: {
          result,
          resultBy,
          resultConfirmed: true,
          verifiedBy: ctx.session.user.id,
          verifiedAt: new Date(),
          status: "COMPLETED",
          notes,
        },
      });

      return updatedGame;
    }),

  // Generate pairings for a round
  generatePairings: organizerProcedure
    .input(generatePairingsSchema)
    .mutation(async ({ ctx, input }) => {
      const { roundId, method, avoidRematch, balanceColors, manualPairings } = input;

      // Get round and tournament info
      const round = await ctx.db.round.findUnique({
        where: { id: roundId },
        include: {
          tournament: {
            include: {
              participants: {
                where: {
                  status: { in: ["CONFIRMED", "CHECKED_IN"] },
                },
                include: {
                  user: {
                    include: { profile: true },
                  },
                },
              },
              organizers: { where: { userId: ctx.session.user.id } },
            },
          },
          games: true,
        },
      });

      if (!round) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Round not found",
        });
      }

      // Check organizer permissions
      const isOrganizer = round.tournament.createdById === ctx.session.user.id ||
                         round.tournament.organizers.length > 0 ||
                         ctx.session.user.role === "ADMIN";

      if (!isOrganizer) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "You don't have permission to generate pairings",
        });
      }

      // Check if pairings already exist
      if (round.games.length > 0) {
        throw new TRPCError({
          code: "BAD_REQUEST",
          message: "Pairings already exist for this round",
        });
      }

      let pairings: any[] = [];

      if (method === "MANUAL" && manualPairings) {
        pairings = manualPairings;
      } else if (method === "SWISS") {
        pairings = SwissPairingEngine.generatePairings(
          round.tournament.participants,
          round.roundNumber
        );
      } else if (method === "ROUND_ROBIN") {
        // Implement round-robin pairing logic
        pairings = generateRoundRobinPairings(
          round.tournament.participants,
          round.roundNumber
        );
      }

      // Create games from pairings
      const games = await Promise.all(
        pairings.map((pairing, index) =>
          ctx.db.game.create({
            data: {
              roundId,
              whiteId: pairing.whiteId,
              blackId: pairing.blackId,
              board: pairing.board || index + 1,
              status: "SCHEDULED",
            },
          })
        )
      );

      // Update round status
      await ctx.db.round.update({
        where: { id: roundId },
        data: {
          pairingsGenerated: true,
          status: "ACTIVE",
        },
      });

      return {
        gamesCreated: games.length,
        games,
      };
    }),

  // Get player's games
  getPlayerGames: protectedProcedure
    .input(z.object({
      tournamentId: z.string().optional(),
      limit: z.number().int().min(1).max(50).default(10),
    }))
    .query(async ({ ctx, input }) => {
      const { tournamentId, limit } = input;

      const where: any = {
        OR: [
          { whiteId: ctx.session.user.id },
          { blackId: ctx.session.user.id },
        ],
      };

      if (tournamentId) {
        where.round = { tournamentId };
      }

      const games = await ctx.db.game.findMany({
        where,
        take: limit,
        orderBy: { createdAt: "desc" },
        include: {
          white: {
            select: {
              id: true,
              name: true,
              profile: {
                select: { firstName: true, lastName: true },
              },
            },
          },
          black: {
            select: {
              id: true,
              name: true,
              profile: {
                select: { firstName: true, lastName: true },
              },
            },
          },
          round: {
            select: {
              roundNumber: true,
              tournament: {
                select: { name: true },
              },
            },
          },
        },
      });

      return games;
    }),

  // Get game statistics
  getStatistics: publicProcedure
    .input(z.object({
      tournamentId: z.string().optional(),
      playerId: z.string().optional(),
    }))
    .query(async ({ ctx, input }) => {
      const { tournamentId, playerId } = input;

      const where: any = {};

      if (tournamentId) {
        where.round = { tournamentId };
      }

      if (playerId) {
        where.OR = [
          { whiteId: playerId },
          { blackId: playerId },
        ];
      }

      const [
        totalGames,
        completedGames,
        whiteWins,
        blackWins,
        draws,
        avgGameDuration,
      ] = await Promise.all([
        ctx.db.game.count({ where }),
        ctx.db.game.count({ where: { ...where, status: "COMPLETED" } }),
        ctx.db.game.count({ where: { ...where, result: "WHITE_WINS" } }),
        ctx.db.game.count({ where: { ...where, result: "BLACK_WINS" } }),
        ctx.db.game.count({ where: { ...where, result: "DRAW" } }),
        ctx.db.game.aggregate({
          where: { ...where, duration: { not: null } },
          _avg: { duration: true },
        }),
      ]);

      return {
        totalGames,
        completedGames,
        inProgress: totalGames - completedGames,
        results: {
          whiteWins,
          blackWins,
          draws,
        },
        avgGameDuration: avgGameDuration._avg.duration,
      };
    }),
});

// Helper function for round-robin pairings
function generateRoundRobinPairings(participants: any[], roundNumber: number): any[] {
  // Implement round-robin pairing algorithm
  const pairings: any[] = [];
  // ... implementation details
  return pairings;
}
```

#### Acceptance Criteria
- [ ] All game operations implemented
- [ ] Result submission and verification working
- [ ] Pairing generation algorithms functional
- [ ] Game statistics accurate
- [ ] Conflict resolution system working

### Phase 4: Frontend Components (3 days)

#### Tasks
1. **Create Game Display Components**
   - Game card/tile component
   - Game details view
   - Live game board (optional)

2. **Create Result Submission Interface**
   - Result submission form
   - PGN input and validation
   - Conflict resolution interface

3. **Create Organizer Tools**
   - Pairing generation interface
   - Game management dashboard
   - Result verification tools

#### Implementation

```typescript
// src/components/game/game-result-form.tsx
"use client";

import { useState } from "react";
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { Button } from "@/components/ui/button";
import { Textarea } from "@/components/ui/textarea";
import { Form, FormControl, FormField, FormItem, FormLabel, FormMessage } from "@/components/ui/form";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { toast } from "@/components/ui/use-toast";
import { api } from "@/trpc/react";
import { submitGameResultSchema, type SubmitGameResult } from "@/lib/validations/game";

interface GameResultFormProps {
  gameId: string;
  whiteName: string;
  blackName: string;
  currentUserId: string;
  onSuccess?: () => void;
}

export function GameResultForm({ 
  gameId, 
  whiteName, 
  blackName, 
  currentUserId,
  onSuccess 
}: GameResultFormProps) {
  const [isSubmitting, setIsSubmitting] = useState(false);

  const form = useForm<SubmitGameResult>({
    resolver: zodResolver(submitGameResultSchema),
    defaultValues: {
      gameId,
    },
  });

  const submitResultMutation = api.game.submitResult.useMutation({
    onSuccess: () => {
      toast({ title: "Result submitted successfully" });
      onSuccess?.();
    },
    onError: (error) => {
      toast({ 
        title: "Failed to submit result", 
        description: error.message,
        variant: "destructive" 
      });
    },
  });

  const onSubmit = async (data: SubmitGameResult) => {
    setIsSubmitting(true);
    try {
      await submitResultMutation.mutateAsync(data);
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <Form {...form}>
      <form onSubmit={form.handleSubmit(onSubmit)} className="space-y-6">
        <div className="text-center">
          <h3 className="text-lg font-medium">Submit Game Result</h3>
          <p className="text-sm text-muted-foreground">
            {whiteName} (White) vs {blackName} (Black)
          </p>
        </div>

        <FormField
          control={form.control}
          name="result"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Game Result</FormLabel>
              <Select onValueChange={field.onChange} defaultValue={field.value}>
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="Select game result" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  <SelectItem value="WHITE_WINS">White Wins (1-0)</SelectItem>
                  <SelectItem value="BLACK_WINS">Black Wins (0-1)</SelectItem>
                  <SelectItem value="DRAW">Draw (½-½)</SelectItem>
                  <SelectItem value="WHITE_FORFEIT">White Forfeit</SelectItem>
                  <SelectItem value="BLACK_FORFEIT">Black Forfeit</SelectItem>
                  <SelectItem value="DOUBLE_FORFEIT">Double Forfeit</SelectItem>
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="resultBy"
          render={({ field }) => (
            <FormItem>
              <FormLabel>How the game ended</FormLabel>
              <Select onValueChange={field.onChange} defaultValue={field.value}>
                <FormControl>
                  <SelectTrigger>
                    <SelectValue placeholder="Select how the game ended" />
                  </SelectTrigger>
                </FormControl>
                <SelectContent>
                  <SelectItem value="CHECKMATE">Checkmate</SelectItem>
                  <SelectItem value="RESIGNATION">Resignation</SelectItem>
                  <SelectItem value="TIME_FORFEIT">Time Forfeit</SelectItem>
                  <SelectItem value="DRAW_AGREEMENT">Draw Agreement</SelectItem>
                  <SelectItem value="STALEMATE">Stalemate</SelectItem>
                  <SelectItem value="THREEFOLD_REPETITION">Threefold Repetition</SelectItem>
                  <SelectItem value="FIFTY_MOVE_RULE">Fifty Move Rule</SelectItem>
                  <SelectItem value="INSUFFICIENT_MATERIAL">Insufficient Material</SelectItem>
                  <SelectItem value="ARBITER_DECISION">Arbiter Decision</SelectItem>
                  <SelectItem value="NO_SHOW">No Show</SelectItem>
                </SelectContent>
              </Select>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="pgn"
          render={({ field }) => (
            <FormItem>
              <FormLabel>PGN (Optional)</FormLabel>
              <FormControl>
                <Textarea 
                  placeholder="Paste the game PGN here..."
                  className="resize-none font-mono text-sm"
                  rows={6}
                  {...field} 
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <FormField
          control={form.control}
          name="notes"
          render={({ field }) => (
            <FormItem>
              <FormLabel>Notes (Optional)</FormLabel>
              <FormControl>
                <Textarea 
                  placeholder="Any additional notes about the game..."
                  className="resize-none"
                  rows={3}
                  {...field} 
                />
              </FormControl>
              <FormMessage />
            </FormItem>
          )}
        />

        <Button type="submit" disabled={isSubmitting} className="w-full">
          {isSubmitting ? "Submitting..." : "Submit Result"}
        </Button>
      </form>
    </Form>
  );
}
```

#### Acceptance Criteria
- [ ] Game result submission form functional
- [ ] Game display components responsive
- [ ] Pairing generation interface working
- [ ] Organizer tools comprehensive
- [ ] PGN validation and display working

### Phase 5: Testing & Integration (1 day)

#### Tasks
1. **Unit Tests**
   - Game validation functions
   - Pairing algorithms
   - Result submission logic

2. **Integration Tests**
   - Game creation and management
   - Result submission workflow
   - Organizer operations

3. **Performance Testing**
   - Large tournament game handling
   - Pairing generation performance

#### Test Implementation

```typescript
// __tests__/game/game-api.test.ts
import { describe, expect, it } from '@jest/globals';
import { submitGameResultSchema, validatePGN, SwissPairingEngine } from '@/lib/validations/game';

describe('Game Management', () => {
  describe('Result Submission', () => {
    it('should validate game result data', () => {
      const resultData = {
        gameId: 'game-1',
        result: 'WHITE_WINS' as const,
        resultBy: 'CHECKMATE' as const,
        pgn: '1. e4 e5 2. Nf3 Nc6 3. Bb5 1-0',
      };

      const result = submitGameResultSchema.safeParse(resultData);
      expect(result.success).toBe(true);
    });

    it('should reject invalid result types', () => {
      const resultData = {
        gameId: 'game-1',
        result: 'INVALID_RESULT' as any,
      };

      const result = submitGameResultSchema.safeParse(resultData);
      expect(result.success).toBe(false);
    });
  });

  describe('PGN Validation', () => {
    it('should validate correct PGN format', () => {
      const pgn = '1. e4 e5 2. Nf3 Nc6 3. Bb5 a6 4. Ba4 Nf6 5. O-O Be7 1-0';
      const result = validatePGN(pgn);
      expect(result.isValid).toBe(true);
    });

    it('should reject invalid PGN format', () => {
      const pgn = 'invalid pgn format';
      const result = validatePGN(pgn);
      expect(result.isValid).toBe(false);
    });
  });

  describe('Swiss Pairing Engine', () => {
    it('should generate valid pairings', () => {
      const participants = [
        { userId: '1', score: 2, seedRating: 1800 },
        { userId: '2', score: 2, seedRating: 1750 },
        { userId: '3', score: 1, seedRating: 1700 },
        { userId: '4', score: 1, seedRating: 1650 },
      ];

      const pairings = SwissPairingEngine.generatePairings(participants, 2);
      expect(pairings).toHaveLength(2);
      expect(pairings[0]).toHaveProperty('whiteId');
      expect(pairings[0]).toHaveProperty('blackId');
    });
  });
});
```

#### Acceptance Criteria
- [ ] All game operations tested
- [ ] Pairing algorithms validated
- [ ] Result submission workflow tested
- [ ] Performance benchmarks met
- [ ] Error scenarios covered

## Risk Assessment

### High Risk
- **Complex pairing algorithms**: Swiss system and round-robin pairing accuracy
- **Result conflicts**: Handling disagreement between players
- **Data integrity**: Ensuring game results are consistent and accurate

### Medium Risk
- **PGN validation**: Comprehensive chess notation validation
- **Performance**: Large tournaments with many games

### Low Risk
- **UI/UX consistency**: Standard game display patterns
- **Basic CRUD operations**: Standard database operations

## Success Metrics

- Game creation success rate > 98%
- Result submission success rate > 95%
- Pairing generation time < 2 seconds for 100 players
- Zero data consistency issues
- 90% test coverage for game operations

## Dependencies for Next Phase

This game management system enables:
- Real-time scoreboard updates
- Tournament standings calculation
- Player performance tracking
- Tournament progression monitoring
