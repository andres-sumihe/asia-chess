# Scoreboard System Implementation Plan

## Feature Overview

Implement real-time scoreboard system for displaying live tournament progress, game statuses, and results with automatic updates and filtering capabilities.

## Objectives

- Real-time game status display
- Live score tracking and updates
- Tournament progress visualization
- Game filtering and search
- Round-by-round results display
- Mobile-responsive scoreboard interface

## Technical Requirements

### Dependencies
- Game Management (completed)
- Tournament Management (completed)
- Participant Management (completed)
- Authentication system (completed)
- WebSocket/SSE for real-time updates

### Constraints
- Real-time updates must be efficient
- Scoreboard must handle large tournaments (500+ participants)
- Mobile-first responsive design required
- Public viewing with optional authentication

## Implementation Phases

### Phase 1: Enhanced Database Views & Indexes (1 day)

#### Tasks
1. **Create Scoreboard Database Views**
   - Tournament scoreboard materialized view
   - Game status aggregation views
   - Performance-optimized indexes

2. **Add Scoreboard Configuration**
   - Display preferences and settings
   - Public/private scoreboard controls
   - Custom scoreboard layouts

#### Database Changes

```prisma
// Add to existing Tournament model
model Tournament {
  // Existing fields...
  
  // Scoreboard settings
  scoreboardConfig Json?     // Configuration for scoreboard display
  publicScoreboard Boolean   @default(true)
  showPlayerRatings Boolean  @default(true)
  showGameHistory Boolean    @default(true)
  refreshInterval Int        @default(30) // seconds
  
  // Display settings
  hideUnpairedPlayers Boolean @default(false)
  showOnlyActiveRounds Boolean @default(false)
  groupByScore Boolean       @default(true)
  
  // Real-time features
  enableLiveUpdates Boolean  @default(true)
  allowSpectators Boolean    @default(true)
}

// Scoreboard view for performance
model ScoreboardEntry {
  id              String    @id @default(cuid())
  tournamentId    String
  participantId   String
  userId          String
  
  // Current standings
  currentScore    Float     @default(0)
  gamesPlayed     Int       @default(0)
  wins            Int       @default(0)
  draws           Int       @default(0)
  losses          Int       @default(0)
  
  // Performance metrics
  performance     Float?    // Performance rating
  averageOpponent Float?    // Average opponent rating
  colorBalance    Json?     // { white: number, black: number }
  
  // Tie-break scores
  tiebreak1       Float?    // Sonneborn-Berger or Progressive
  tiebreak2       Float?    // Buchholz or Direct encounter
  tiebreak3       Float?    // Secondary tie-breaks
  
  // Current position
  currentRank     Int?
  previousRank    Int?
  rankChange      Int?      // +1, -1, 0
  
  // Round status
  currentRoundId  String?
  currentGameId   String?
  currentOpponent String?   // Opponent user ID
  currentColor    String?   // "WHITE", "BLACK", "BYE"
  gameStatus      String?   // Game status from current round
  
  // Relations
  tournament      Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  participant     TournamentParticipant @relation(fields: [participantId], references: [id], onDelete: Cascade)
  user            User      @relation(fields: [userId], references: [id])
  
  lastUpdated     DateTime  @updatedAt
  
  @@unique([tournamentId, participantId])
  @@index([tournamentId, currentScore, tiebreak1])
  @@index([tournamentId, currentRank])
  @@index([currentGameId])
}

// Live game status for scoreboard
model LiveGameStatus {
  id              String    @id @default(cuid())
  gameId          String    @unique
  tournamentId    String
  roundNumber     Int
  board           Int?
  
  // Players
  whiteId         String
  blackId         String
  whiteName       String
  blackName       String
  whiteRating     Int?
  blackRating     Int?
  
  // Game state
  status          GameStatus
  result          GameResult?
  currentMove     String?   // Last move in algebraic notation
  moveCount       Int       @default(0)
  
  // Timing
  startTime       DateTime?
  currentDuration Int?      // Game duration in seconds
  whiteTimeLeft   Int?      // White's remaining time
  blackTimeLeft   Int?      // Black's remaining time
  
  // Live status
  isLive          Boolean   @default(false)
  lastMoveTime    DateTime?
  spectatorCount  Int       @default(0)
  
  // Relations
  game            Game      @relation(fields: [gameId], references: [id], onDelete: Cascade)
  tournament      Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  
  updatedAt       DateTime  @updatedAt
  
  @@index([tournamentId, roundNumber])
  @@index([tournamentId, status])
  @@index([isLive])
}

// Scoreboard subscription for real-time updates
model ScoreboardSubscription {
  id           String    @id @default(cuid())
  sessionId    String    // WebSocket session ID
  tournamentId String
  userId       String?   // Optional for authenticated users
  
  // Subscription preferences
  roundUpdates Boolean   @default(true)
  gameUpdates  Boolean   @default(true)
  resultUpdates Boolean  @default(true)
  standingsUpdates Boolean @default(true)
  
  // Activity tracking
  isActive     Boolean   @default(true)
  lastSeen     DateTime  @default(now())
  
  createdAt    DateTime  @default(now())
  
  @@index([tournamentId, isActive])
  @@index([sessionId])
}

// Add relations to existing models
model User {
  // Existing fields...
  scoreboardEntries ScoreboardEntry[]
}

model TournamentParticipant {
  // Existing fields...
  scoreboardEntry ScoreboardEntry?
}

model Game {
  // Existing fields...
  liveStatus LiveGameStatus?
}
```

#### Acceptance Criteria
- [ ] Scoreboard database views created and optimized
- [ ] Performance indexes implemented
- [ ] Live game status tracking enabled
- [ ] Subscription management system functional
- [ ] Database migration tested successfully

### Phase 2: Scoreboard Data Processing (2 days)

#### Tasks
1. **Create Scoreboard Calculation Engine**
   - Real-time score calculation
   - Tie-break computation
   - Ranking algorithm implementation

2. **Implement Update Triggers**
   - Automatic scoreboard updates on game completion
   - Live status updates during games
   - Efficient batch processing

#### Implementation

```typescript
// src/lib/scoreboard/calculator.ts
export interface TieBreakConfig {
  method: 'SONNEBORN_BERGER' | 'BUCHHOLZ' | 'PROGRESSIVE' | 'DIRECT_ENCOUNTER' | 'RATING_PERFORMANCE';
  includeOpponentForfeits: boolean;
  useModifiedSystem: boolean;
}

export interface ScoreboardConfig {
  tieBreaks: TieBreakConfig[];
  showPerformanceRating: boolean;
  showColorBalance: boolean;
  groupByScore: boolean;
  highlightLeaders: boolean;
}

export class ScoreboardCalculator {
  private tournament: Tournament;
  private games: Game[];
  private participants: TournamentParticipant[];

  constructor(tournament: Tournament, games: Game[], participants: TournamentParticipant[]) {
    this.tournament = tournament;
    this.games = games;
    this.participants = participants;
  }

  // Calculate complete scoreboard for tournament
  async calculateScoreboard(): Promise<ScoreboardEntry[]> {
    const entries: ScoreboardEntry[] = [];

    for (const participant of this.participants) {
      const entry = await this.calculateParticipantEntry(participant);
      entries.push(entry);
    }

    // Sort by score and tie-breaks
    return this.sortEntries(entries);
  }

  // Calculate individual participant entry
  private async calculateParticipantEntry(participant: TournamentParticipant): Promise<ScoreboardEntry> {
    const userGames = this.getUserGames(participant.userId);
    
    // Basic score calculation
    const { score, wins, draws, losses } = this.calculateBasicScore(userGames, participant.userId);
    
    // Tie-break calculations
    const tiebreak1 = await this.calculateTieBreak1(participant.userId, userGames);
    const tiebreak2 = await this.calculateTieBreak2(participant.userId, userGames);
    const tiebreak3 = await this.calculateTieBreak3(participant.userId, userGames);
    
    // Performance metrics
    const performance = this.calculatePerformance(userGames, participant.userId, participant.seedRating);
    const averageOpponent = this.calculateAverageOpponentRating(userGames, participant.userId);
    const colorBalance = this.calculateColorBalance(userGames, participant.userId);
    
    // Current round information
    const currentRoundInfo = await this.getCurrentRoundInfo(participant.userId);

    return {
      tournamentId: this.tournament.id,
      participantId: participant.id,
      userId: participant.userId,
      currentScore: score,
      gamesPlayed: userGames.filter(g => g.status === 'COMPLETED').length,
      wins,
      draws,
      losses,
      tiebreak1,
      tiebreak2,
      tiebreak3,
      performance,
      averageOpponent,
      colorBalance: JSON.stringify(colorBalance),
      ...currentRoundInfo,
      lastUpdated: new Date(),
    };
  }

  // Calculate basic score (1 for win, 0.5 for draw, 0 for loss)
  private calculateBasicScore(games: Game[], userId: string): { 
    score: number; 
    wins: number; 
    draws: number; 
    losses: number; 
  } {
    let score = 0;
    let wins = 0;
    let draws = 0;
    let losses = 0;

    for (const game of games) {
      if (game.status !== 'COMPLETED' || !game.result) continue;

      const isWhite = game.whiteId === userId;
      
      switch (game.result) {
        case 'WHITE_WINS':
          if (isWhite) {
            score += 1;
            wins++;
          } else {
            losses++;
          }
          break;
        case 'BLACK_WINS':
          if (!isWhite) {
            score += 1;
            wins++;
          } else {
            losses++;
          }
          break;
        case 'DRAW':
          score += 0.5;
          draws++;
          break;
        case 'WHITE_FORFEIT':
          if (!isWhite) {
            score += 1;
            wins++;
          } else {
            losses++;
          }
          break;
        case 'BLACK_FORFEIT':
          if (isWhite) {
            score += 1;
            wins++;
          } else {
            losses++;
          }
          break;
      }
    }

    return { score, wins, draws, losses };
  }

  // Sonneborn-Berger tie-break calculation
  private async calculateTieBreak1(userId: string, userGames: Game[]): Promise<number> {
    let sonneborn = 0;

    for (const game of userGames) {
      if (game.status !== 'COMPLETED') continue;

      const opponentId = game.whiteId === userId ? game.blackId : game.whiteId;
      const opponentScore = await this.calculateOpponentTotalScore(opponentId);
      
      const gameResult = this.getGameResult(game, userId);
      sonneborn += gameResult * opponentScore;
    }

    return sonneborn;
  }

  // Buchholz tie-break calculation
  private async calculateTieBreak2(userId: string, userGames: Game[]): Promise<number> {
    let buchholz = 0;

    for (const game of userGames) {
      if (game.status !== 'COMPLETED') continue;

      const opponentId = game.whiteId === userId ? game.blackId : game.whiteId;
      const opponentScore = await this.calculateOpponentTotalScore(opponentId);
      buchholz += opponentScore;
    }

    return buchholz;
  }

  // Progressive score tie-break
  private async calculateTieBreak3(userId: string, userGames: Game[]): Promise<number> {
    const roundScores: number[] = [];
    let cumulativeScore = 0;

    // Sort games by round
    const sortedGames = userGames
      .filter(g => g.status === 'COMPLETED')
      .sort((a, b) => a.round.roundNumber - b.round.roundNumber);

    for (const game of sortedGames) {
      const gameResult = this.getGameResult(game, userId);
      cumulativeScore += gameResult;
      roundScores.push(cumulativeScore);
    }

    // Sum of progressive scores
    return roundScores.reduce((sum, score) => sum + score, 0);
  }

  // Performance rating calculation
  private calculatePerformance(games: Game[], userId: string, playerRating?: number): number | null {
    if (!playerRating) return null;

    const completedGames = games.filter(g => g.status === 'COMPLETED');
    if (completedGames.length === 0) return null;

    let totalOpponentRating = 0;
    let totalScore = 0;
    let gameCount = 0;

    for (const game of completedGames) {
      const isWhite = game.whiteId === userId;
      const opponent = isWhite ? game.black : game.white;
      const opponentRating = opponent?.profile?.fideRating || opponent?.profile?.localRating;
      
      if (!opponentRating) continue;

      const gameResult = this.getGameResult(game, userId);
      
      totalOpponentRating += opponentRating;
      totalScore += gameResult;
      gameCount++;
    }

    if (gameCount === 0) return null;

    const averageOpponentRating = totalOpponentRating / gameCount;
    const scorePercentage = totalScore / gameCount;
    
    // FIDE performance rating calculation approximation
    const performanceRating = averageOpponentRating + (scorePercentage - 0.5) * 400;
    
    return Math.round(performanceRating);
  }

  // Calculate average opponent rating
  private calculateAverageOpponentRating(games: Game[], userId: string): number | null {
    const completedGames = games.filter(g => g.status === 'COMPLETED');
    if (completedGames.length === 0) return null;

    let totalRating = 0;
    let count = 0;

    for (const game of completedGames) {
      const isWhite = game.whiteId === userId;
      const opponent = isWhite ? game.black : game.white;
      const opponentRating = opponent?.profile?.fideRating || opponent?.profile?.localRating;
      
      if (opponentRating) {
        totalRating += opponentRating;
        count++;
      }
    }

    return count > 0 ? Math.round(totalRating / count) : null;
  }

  // Calculate color balance
  private calculateColorBalance(games: Game[], userId: string): { white: number; black: number } {
    let whiteCount = 0;
    let blackCount = 0;

    for (const game of games) {
      if (game.status === 'COMPLETED') {
        if (game.whiteId === userId) whiteCount++;
        if (game.blackId === userId) blackCount++;
      }
    }

    return { white: whiteCount, black: blackCount };
  }

  // Get current round information
  private async getCurrentRoundInfo(userId: string): Promise<{
    currentRoundId?: string;
    currentGameId?: string;
    currentOpponent?: string;
    currentColor?: string;
    gameStatus?: string;
  }> {
    // Find current active round
    const currentRound = await this.findCurrentRound();
    if (!currentRound) return {};

    // Find user's game in current round
    const currentGame = currentRound.games.find(
      g => g.whiteId === userId || g.blackId === userId
    );

    if (!currentGame) return { currentRoundId: currentRound.id };

    const isWhite = currentGame.whiteId === userId;
    const opponentId = isWhite ? currentGame.blackId : currentGame.whiteId;

    return {
      currentRoundId: currentRound.id,
      currentGameId: currentGame.id,
      currentOpponent: opponentId,
      currentColor: isWhite ? 'WHITE' : 'BLACK',
      gameStatus: currentGame.status,
    };
  }

  // Helper methods
  private getUserGames(userId: string): Game[] {
    return this.games.filter(g => g.whiteId === userId || g.blackId === userId);
  }

  private getGameResult(game: Game, userId: string): number {
    if (!game.result) return 0;

    const isWhite = game.whiteId === userId;
    
    switch (game.result) {
      case 'WHITE_WINS': return isWhite ? 1 : 0;
      case 'BLACK_WINS': return isWhite ? 0 : 1;
      case 'DRAW': return 0.5;
      case 'WHITE_FORFEIT': return isWhite ? 0 : 1;
      case 'BLACK_FORFEIT': return isWhite ? 1 : 0;
      default: return 0;
    }
  }

  private async calculateOpponentTotalScore(opponentId: string): Promise<number> {
    const opponentGames = this.getUserGames(opponentId);
    const { score } = this.calculateBasicScore(opponentGames, opponentId);
    return score;
  }

  private async findCurrentRound(): Promise<Round | null> {
    // Implementation to find current active round
    return null; // Placeholder
  }

  // Sort entries by score and tie-breaks
  private sortEntries(entries: ScoreboardEntry[]): ScoreboardEntry[] {
    return entries.sort((a, b) => {
      // Primary sort: current score (descending)
      if (a.currentScore !== b.currentScore) {
        return b.currentScore - a.currentScore;
      }
      
      // Secondary sort: tie-break 1 (descending)
      if (a.tiebreak1 !== b.tiebreak1) {
        return (b.tiebreak1 || 0) - (a.tiebreak1 || 0);
      }
      
      // Tertiary sort: tie-break 2 (descending)
      if (a.tiebreak2 !== b.tiebreak2) {
        return (b.tiebreak2 || 0) - (a.tiebreak2 || 0);
      }
      
      // Final sort: tie-break 3 (descending)
      return (b.tiebreak3 || 0) - (a.tiebreak3 || 0);
    }).map((entry, index) => ({
      ...entry,
      currentRank: index + 1,
    }));
  }
}

// Real-time update service
export class ScoreboardUpdateService {
  private db: PrismaClient;
  private wsServer: WebSocketServer; // WebSocket implementation

  constructor(db: PrismaClient, wsServer: WebSocketServer) {
    this.db = db;
    this.wsServer = wsServer;
  }

  // Update scoreboard when game result changes
  async updateScoreboardForGame(gameId: string): Promise<void> {
    const game = await this.db.game.findUnique({
      where: { id: gameId },
      include: {
        round: { include: { tournament: true } },
        white: { include: { profile: true } },
        black: { include: { profile: true } },
      },
    });

    if (!game) return;

    // Recalculate scoreboard for tournament
    await this.recalculateScoreboard(game.round.tournamentId);
    
    // Broadcast update to subscribers
    await this.broadcastScoreboardUpdate(game.round.tournamentId, {
      type: 'GAME_RESULT',
      gameId,
      roundNumber: game.round.roundNumber,
      result: game.result,
    });
  }

  // Recalculate complete scoreboard
  async recalculateScoreboard(tournamentId: string): Promise<void> {
    const tournament = await this.db.tournament.findUnique({
      where: { id: tournamentId },
      include: {
        participants: {
          where: { status: { in: ['CONFIRMED', 'CHECKED_IN'] } },
          include: { user: { include: { profile: true } } },
        },
        rounds: {
          include: {
            games: {
              include: {
                white: { include: { profile: true } },
                black: { include: { profile: true } },
              },
            },
          },
        },
      },
    });

    if (!tournament) return;

    const calculator = new ScoreboardCalculator(
      tournament,
      tournament.rounds.flatMap(r => r.games),
      tournament.participants
    );

    const entries = await calculator.calculateScoreboard();

    // Update database with new entries
    await this.db.$transaction(async (tx) => {
      // Delete existing entries
      await tx.scoreboardEntry.deleteMany({
        where: { tournamentId },
      });

      // Create new entries
      await tx.scoreboardEntry.createMany({
        data: entries,
      });
    });
  }

  // Broadcast update to WebSocket subscribers
  private async broadcastScoreboardUpdate(tournamentId: string, update: any): Promise<void> {
    const subscribers = await this.db.scoreboardSubscription.findMany({
      where: { 
        tournamentId,
        isActive: true,
      },
    });

    for (const subscriber of subscribers) {
      this.wsServer.send(subscriber.sessionId, {
        type: 'SCOREBOARD_UPDATE',
        data: update,
      });
    }
  }
}
```

#### Acceptance Criteria
- [ ] Scoreboard calculation engine implemented
- [ ] Tie-break algorithms accurate
- [ ] Real-time update triggers working
- [ ] Performance rating calculation functional
- [ ] Update service broadcasting correctly

### Phase 3: API Implementation (2 days)

#### Tasks
1. **Create Scoreboard tRPC Router**
   - Scoreboard data retrieval
   - Real-time subscription management
   - Live game status updates

2. **Implement WebSocket Integration**
   - Real-time scoreboard updates
   - Live game status broadcasting
   - Subscription management

#### Implementation

```typescript
// src/server/api/routers/scoreboard.ts
import { z } from "zod";
import { createTRPCRouter, protectedProcedure, publicProcedure } from "@/server/api/trpc";
import { ScoreboardCalculator, ScoreboardUpdateService } from "@/lib/scoreboard/calculator";
import { TRPCError } from "@trpc/server";

export const scoreboardRouter = createTRPCRouter({
  // Get tournament scoreboard
  getTournamentScoreboard: publicProcedure
    .input(z.object({
      tournamentId: z.string(),
      roundNumber: z.number().optional(),
      limit: z.number().int().min(1).max(500).default(100),
      sortBy: z.enum(['SCORE', 'RATING', 'NAME', 'RANK']).default('SCORE'),
      showOnlyActive: z.boolean().default(false),
    }))
    .query(async ({ ctx, input }) => {
      const { tournamentId, roundNumber, limit, sortBy, showOnlyActive } = input;

      // Get tournament and verify public access
      const tournament = await ctx.db.tournament.findUnique({
        where: { id: tournamentId },
        include: {
          organizers: true,
        },
      });

      if (!tournament) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Tournament not found",
        });
      }

      // Check if scoreboard is public or user has access
      const hasAccess = tournament.publicScoreboard ||
                       tournament.createdById === ctx.session?.user?.id ||
                       tournament.organizers.some(o => o.userId === ctx.session?.user?.id) ||
                       ctx.session?.user?.role === "ADMIN";

      if (!hasAccess) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "This scoreboard is private",
        });
      }

      // Get scoreboard entries
      const where: any = { tournamentId };
      
      if (showOnlyActive) {
        where.currentGameId = { not: null };
      }

      const entries = await ctx.db.scoreboardEntry.findMany({
        where,
        take: limit,
        orderBy: [
          { currentScore: 'desc' },
          { tiebreak1: 'desc' },
          { tiebreak2: 'desc' },
          { tiebreak3: 'desc' },
        ],
        include: {
          user: {
            select: {
              id: true,
              name: true,
              profile: {
                select: {
                  firstName: true,
                  lastName: true,
                  fideRating: true,
                  localRating: true,
                  country: true,
                  title: true,
                },
              },
            },
          },
          participant: {
            select: {
              seedNumber: true,
              seedRating: true,
            },
          },
        },
      });

      // Get tournament metadata
      const [
        totalParticipants,
        currentRound,
        totalRounds,
        completedGames,
        totalGames,
      ] = await Promise.all([
        ctx.db.tournamentParticipant.count({
          where: { 
            tournamentId,
            status: { in: ['CONFIRMED', 'CHECKED_IN'] },
          },
        }),
        ctx.db.round.findFirst({
          where: { 
            tournamentId,
            status: 'ACTIVE',
          },
          orderBy: { roundNumber: 'desc' },
        }),
        ctx.db.round.count({ where: { tournamentId } }),
        ctx.db.game.count({
          where: {
            round: { tournamentId },
            status: 'COMPLETED',
          },
        }),
        ctx.db.game.count({
          where: { round: { tournamentId } },
        }),
      ]);

      return {
        entries,
        metadata: {
          tournamentName: tournament.name,
          totalParticipants,
          currentRound: currentRound?.roundNumber || 0,
          totalRounds,
          completedGames,
          totalGames,
          progressPercentage: totalGames > 0 ? (completedGames / totalGames) * 100 : 0,
          lastUpdated: new Date(),
        },
        config: {
          showPlayerRatings: tournament.showPlayerRatings,
          showGameHistory: tournament.showGameHistory,
          refreshInterval: tournament.refreshInterval,
          groupByScore: tournament.groupByScore,
        },
      };
    }),

  // Get live games for tournament
  getLiveGames: publicProcedure
    .input(z.object({
      tournamentId: z.string(),
      roundNumber: z.number().optional(),
    }))
    .query(async ({ ctx, input }) => {
      const { tournamentId, roundNumber } = input;

      const where: any = { 
        tournamentId,
        isLive: true,
      };

      if (roundNumber) {
        where.roundNumber = roundNumber;
      }

      const liveGames = await ctx.db.liveGameStatus.findMany({
        where,
        orderBy: [
          { roundNumber: 'desc' },
          { board: 'asc' },
        ],
        include: {
          game: {
            select: {
              id: true,
              result: true,
              pgn: true,
              startTime: true,
              endTime: true,
            },
          },
        },
      });

      return liveGames;
    }),

  // Get player's tournament progress
  getPlayerProgress: protectedProcedure
    .input(z.object({
      tournamentId: z.string(),
      userId: z.string().optional(),
    }))
    .query(async ({ ctx, input }) => {
      const userId = input.userId || ctx.session.user.id;
      const { tournamentId } = input;

      // Get player's scoreboard entry
      const entry = await ctx.db.scoreboardEntry.findUnique({
        where: {
          tournamentId_participantId: {
            tournamentId,
            participantId: userId,
          },
        },
        include: {
          user: {
            select: {
              name: true,
              profile: {
                select: {
                  firstName: true,
                  lastName: true,
                  fideRating: true,
                  localRating: true,
                },
              },
            },
          },
        },
      });

      if (!entry) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Player not found in tournament",
        });
      }

      // Get player's games
      const games = await ctx.db.game.findMany({
        where: {
          round: { tournamentId },
          OR: [
            { whiteId: userId },
            { blackId: userId },
          ],
        },
        include: {
          white: {
            select: {
              name: true,
              profile: { select: { firstName: true, lastName: true } },
            },
          },
          black: {
            select: {
              name: true,
              profile: { select: { firstName: true, lastName: true } },
            },
          },
          round: {
            select: { roundNumber: true },
          },
        },
        orderBy: {
          round: { roundNumber: 'asc' },
        },
      });

      // Calculate round-by-round progress
      const roundProgress = games.map((game, index) => {
        const isWhite = game.whiteId === userId;
        const opponent = isWhite ? game.black : game.white;
        
        let gameScore = 0;
        if (game.result === 'WHITE_WINS') gameScore = isWhite ? 1 : 0;
        else if (game.result === 'BLACK_WINS') gameScore = isWhite ? 0 : 1;
        else if (game.result === 'DRAW') gameScore = 0.5;

        return {
          round: game.round.roundNumber,
          opponent: opponent.name || `${opponent.profile?.firstName} ${opponent.profile?.lastName}`,
          color: isWhite ? 'WHITE' : 'BLACK',
          result: game.result,
          score: gameScore,
          cumulativeScore: games.slice(0, index + 1).reduce((sum, g) => {
            const gIsWhite = g.whiteId === userId;
            let gScore = 0;
            if (g.result === 'WHITE_WINS') gScore = gIsWhite ? 1 : 0;
            else if (g.result === 'BLACK_WINS') gScore = gIsWhite ? 0 : 1;
            else if (g.result === 'DRAW') gScore = 0.5;
            return sum + gScore;
          }, 0),
        };
      });

      return {
        entry,
        roundProgress,
        statistics: {
          gamesPlayed: entry.gamesPlayed,
          currentScore: entry.currentScore,
          currentRank: entry.currentRank,
          performance: entry.performance,
          averageOpponent: entry.averageOpponent,
          colorBalance: JSON.parse(entry.colorBalance || '{"white":0,"black":0}'),
        },
      };
    }),

  // Subscribe to live scoreboard updates
  subscribeToUpdates: protectedProcedure
    .input(z.object({
      tournamentId: z.string(),
      sessionId: z.string(),
      preferences: z.object({
        roundUpdates: z.boolean().default(true),
        gameUpdates: z.boolean().default(true),
        resultUpdates: z.boolean().default(true),
        standingsUpdates: z.boolean().default(true),
      }).default({}),
    }))
    .mutation(async ({ ctx, input }) => {
      const { tournamentId, sessionId, preferences } = input;

      // Create or update subscription
      const subscription = await ctx.db.scoreboardSubscription.upsert({
        where: {
          sessionId_tournamentId: {
            sessionId,
            tournamentId,
          },
        },
        update: {
          userId: ctx.session.user.id,
          isActive: true,
          lastSeen: new Date(),
          ...preferences,
        },
        create: {
          sessionId,
          tournamentId,
          userId: ctx.session.user.id,
          isActive: true,
          ...preferences,
        },
      });

      return { subscribed: true, subscriptionId: subscription.id };
    }),

  // Unsubscribe from updates
  unsubscribeFromUpdates: protectedProcedure
    .input(z.object({
      sessionId: z.string(),
      tournamentId: z.string(),
    }))
    .mutation(async ({ ctx, input }) => {
      const { sessionId, tournamentId } = input;

      await ctx.db.scoreboardSubscription.updateMany({
        where: {
          sessionId,
          tournamentId,
        },
        data: {
          isActive: false,
        },
      });

      return { unsubscribed: true };
    }),

  // Get scoreboard statistics
  getStatistics: publicProcedure
    .input(z.object({
      tournamentId: z.string(),
    }))
    .query(async ({ ctx, input }) => {
      const { tournamentId } = input;

      const [
        totalPlayers,
        completedGames,
        totalGames,
        averageRating,
        topPerformers,
        currentRoundInfo,
      ] = await Promise.all([
        ctx.db.scoreboardEntry.count({ where: { tournamentId } }),
        ctx.db.game.count({
          where: {
            round: { tournamentId },
            status: 'COMPLETED',
          },
        }),
        ctx.db.game.count({
          where: { round: { tournamentId } },
        }),
        ctx.db.scoreboardEntry.aggregate({
          where: { tournamentId },
          _avg: {
            performance: true,
          },
        }),
        ctx.db.scoreboardEntry.findMany({
          where: { tournamentId },
          take: 3,
          orderBy: [
            { currentScore: 'desc' },
            { tiebreak1: 'desc' },
          ],
          include: {
            user: {
              select: {
                name: true,
                profile: {
                  select: { firstName: true, lastName: true },
                },
              },
            },
          },
        }),
        ctx.db.round.findFirst({
          where: {
            tournamentId,
            status: 'ACTIVE',
          },
          include: {
            games: {
              where: { status: 'IN_PROGRESS' },
            },
          },
        }),
      ]);

      return {
        totalPlayers,
        completedGames,
        totalGames,
        gamesInProgress: currentRoundInfo?.games.length || 0,
        completionPercentage: totalGames > 0 ? (completedGames / totalGames) * 100 : 0,
        averagePerformance: averageRating._avg.performance,
        topPerformers,
        currentRound: currentRoundInfo?.roundNumber || 0,
      };
    }),

  // Force scoreboard recalculation (organizer only)
  recalculateScoreboard: protectedProcedure
    .input(z.object({
      tournamentId: z.string(),
    }))
    .mutation(async ({ ctx, input }) => {
      const { tournamentId } = input;

      // Verify organizer permissions
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
          message: "You don't have permission to recalculate scoreboard",
        });
      }

      // Recalculate scoreboard
      const updateService = new ScoreboardUpdateService(ctx.db, ctx.wsServer);
      await updateService.recalculateScoreboard(tournamentId);

      return { success: true, recalculatedAt: new Date() };
    }),
});
```

#### Acceptance Criteria
- [ ] Scoreboard API endpoints functional
- [ ] Real-time subscription management working
- [ ] WebSocket integration implemented
- [ ] Player progress tracking accurate
- [ ] Statistics calculation correct

### Phase 4: Frontend Components (2 days)

#### Tasks
1. **Create Scoreboard Display Components**
   - Tournament scoreboard table
   - Live game status display
   - Player progress charts

2. **Implement Real-time Updates**
   - WebSocket connection management
   - Automatic scoreboard refresh
   - Live status indicators

#### Implementation

```typescript
// src/components/scoreboard/tournament-scoreboard.tsx
"use client";

import { useState, useEffect, useMemo } from "react";
import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from "@/components/ui/table";
import { Badge } from "@/components/ui/badge";
import { Button } from "@/components/ui/button";
import { Input } from "@/components/ui/input";
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from "@/components/ui/select";
import { Tabs, TabsContent, TabsList, TabsTrigger } from "@/components/ui/tabs";
import { RefreshCw, Search, Filter, TrendingUp, TrendingDown } from "lucide-react";
import { api } from "@/trpc/react";
import { useWebSocket } from "@/hooks/use-websocket";

interface TournamentScoreboardProps {
  tournamentId: string;
  showControls?: boolean;
  compact?: boolean;
  height?: string;
}

export function TournamentScoreboard({ 
  tournamentId, 
  showControls = true, 
  compact = false,
  height = "600px" 
}: TournamentScoreboardProps) {
  const [searchTerm, setSearchTerm] = useState("");
  const [sortBy, setSortBy] = useState<"SCORE" | "RATING" | "NAME" | "RANK">("SCORE");
  const [showOnlyActive, setShowOnlyActive] = useState(false);
  const [autoRefresh, setAutoRefresh] = useState(true);

  // Fetch scoreboard data
  const {
    data: scoreboardData,
    isLoading,
    refetch,
  } = api.scoreboard.getTournamentScoreboard.useQuery({
    tournamentId,
    sortBy,
    showOnlyActive,
  }, {
    refetchInterval: autoRefresh ? 30000 : false, // Refresh every 30 seconds
  });

  // WebSocket for real-time updates
  const { isConnected, lastMessage } = useWebSocket(
    `ws://localhost:3000/ws/scoreboard/${tournamentId}`,
    {
      onMessage: (message) => {
        const data = JSON.parse(message.data);
        if (data.type === 'SCOREBOARD_UPDATE') {
          refetch();
        }
      },
    }
  );

  // Filter entries based on search term
  const filteredEntries = useMemo(() => {
    if (!scoreboardData?.entries) return [];
    
    if (!searchTerm) return scoreboardData.entries;

    return scoreboardData.entries.filter((entry) => {
      const name = entry.user.name || 
                  `${entry.user.profile?.firstName} ${entry.user.profile?.lastName}`;
      return name.toLowerCase().includes(searchTerm.toLowerCase());
    });
  }, [scoreboardData?.entries, searchTerm]);

  const handleRefresh = () => {
    refetch();
  };

  if (isLoading) {
    return (
      <Card>
        <CardContent className="p-6">
          <div className="flex items-center justify-center h-32">
            <RefreshCw className="w-6 h-6 animate-spin" />
            <span className="ml-2">Loading scoreboard...</span>
          </div>
        </CardContent>
      </Card>
    );
  }

  if (!scoreboardData) {
    return (
      <Card>
        <CardContent className="p-6">
          <div className="text-center text-muted-foreground">
            Scoreboard not available
          </div>
        </CardContent>
      </Card>
    );
  }

  return (
    <Card>
      <CardHeader>
        <div className="flex items-center justify-between">
          <div>
            <CardTitle>{scoreboardData.metadata.tournamentName} - Scoreboard</CardTitle>
            <div className="text-sm text-muted-foreground">
              Round {scoreboardData.metadata.currentRound} of {scoreboardData.metadata.totalRounds} • 
              {scoreboardData.metadata.completedGames}/{scoreboardData.metadata.totalGames} games completed
              {isConnected && (
                <Badge variant="outline" className="ml-2">
                  Live
                </Badge>
              )}
            </div>
          </div>
          
          {showControls && (
            <div className="flex items-center gap-2">
              <Button
                variant="outline"
                size="sm"
                onClick={handleRefresh}
                disabled={isLoading}
              >
                <RefreshCw className={`w-4 h-4 ${isLoading ? 'animate-spin' : ''}`} />
              </Button>
              
              <Select value={sortBy} onValueChange={(value: any) => setSortBy(value)}>
                <SelectTrigger className="w-32">
                  <SelectValue />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="SCORE">Score</SelectItem>
                  <SelectItem value="RATING">Rating</SelectItem>
                  <SelectItem value="NAME">Name</SelectItem>
                  <SelectItem value="RANK">Rank</SelectItem>
                </SelectContent>
              </Select>
            </div>
          )}
        </div>

        {showControls && (
          <div className="flex items-center gap-4">
            <div className="relative flex-1 max-w-sm">
              <Search className="absolute left-2 top-2.5 h-4 w-4 text-muted-foreground" />
              <Input
                placeholder="Search players..."
                className="pl-8"
                value={searchTerm}
                onChange={(e) => setSearchTerm(e.target.value)}
              />
            </div>
            
            <Button
              variant={showOnlyActive ? "default" : "outline"}
              size="sm"
              onClick={() => setShowOnlyActive(!showOnlyActive)}
            >
              <Filter className="w-4 h-4 mr-1" />
              Active Games
            </Button>
          </div>
        )}
      </CardHeader>

      <CardContent>
        <div className="rounded-md border" style={{ height }}>
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead className="w-12">Rank</TableHead>
                <TableHead>Player</TableHead>
                {scoreboardData.config.showPlayerRatings && (
                  <TableHead className="text-center">Rating</TableHead>
                )}
                <TableHead className="text-center">Score</TableHead>
                <TableHead className="text-center">Games</TableHead>
                <TableHead className="text-center">TB1</TableHead>
                <TableHead className="text-center">TB2</TableHead>
                {!compact && (
                  <>
                    <TableHead className="text-center">Perf</TableHead>
                    <TableHead className="text-center">Status</TableHead>
                  </>
                )}
              </TableRow>
            </TableHeader>
            <TableBody>
              {filteredEntries.map((entry, index) => {
                const rankChange = entry.previousRank 
                  ? entry.currentRank! - entry.previousRank 
                  : 0;

                return (
                  <TableRow key={entry.id} className={index < 3 ? "bg-muted/50" : ""}>
                    <TableCell className="font-medium">
                      <div className="flex items-center">
                        {entry.currentRank}
                        {rankChange !== 0 && (
                          <span className="ml-1">
                            {rankChange > 0 ? (
                              <TrendingDown className="w-3 h-3 text-red-500" />
                            ) : (
                              <TrendingUp className="w-3 h-3 text-green-500" />
                            )}
                          </span>
                        )}
                      </div>
                    </TableCell>
                    
                    <TableCell>
                      <div>
                        <div className="font-medium">
                          {entry.user.name || 
                           `${entry.user.profile?.firstName} ${entry.user.profile?.lastName}`}
                        </div>
                        {entry.user.profile?.title && (
                          <Badge variant="secondary" className="text-xs">
                            {entry.user.profile.title}
                          </Badge>
                        )}
                      </div>
                    </TableCell>

                    {scoreboardData.config.showPlayerRatings && (
                      <TableCell className="text-center">
                        {entry.user.profile?.fideRating || 
                         entry.user.profile?.localRating || 
                         entry.participant.seedRating || "-"}
                      </TableCell>
                    )}

                    <TableCell className="text-center font-bold">
                      {entry.currentScore}
                    </TableCell>

                    <TableCell className="text-center">
                      <div className="text-sm">
                        <div>{entry.gamesPlayed}</div>
                        <div className="text-xs text-muted-foreground">
                          {entry.wins}W {entry.draws}D {entry.losses}L
                        </div>
                      </div>
                    </TableCell>

                    <TableCell className="text-center">
                      {entry.tiebreak1?.toFixed(1) || "-"}
                    </TableCell>

                    <TableCell className="text-center">
                      {entry.tiebreak2?.toFixed(1) || "-"}
                    </TableCell>

                    {!compact && (
                      <>
                        <TableCell className="text-center">
                          {entry.performance || "-"}
                        </TableCell>

                        <TableCell className="text-center">
                          {entry.currentGameId ? (
                            <Badge variant="outline" className="text-xs">
                              Playing
                            </Badge>
                          ) : (
                            <span className="text-xs text-muted-foreground">
                              Waiting
                            </span>
                          )}
                        </TableCell>
                      </>
                    )}
                  </TableRow>
                );
              })}
            </TableBody>
          </Table>
        </div>

        {/* Tournament progress bar */}
        <div className="mt-4 p-4 bg-muted rounded-lg">
          <div className="flex items-center justify-between text-sm mb-2">
            <span>Tournament Progress</span>
            <span>{scoreboardData.metadata.progressPercentage.toFixed(1)}%</span>
          </div>
          <div className="w-full bg-background rounded-full h-2">
            <div 
              className="bg-primary h-2 rounded-full transition-all duration-300" 
              style={{ width: `${scoreboardData.metadata.progressPercentage}%` }}
            />
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

#### Acceptance Criteria
- [ ] Scoreboard display responsive and functional
- [ ] Real-time updates working correctly
- [ ] Player search and filtering operational
- [ ] Live game status indicators visible
- [ ] Tournament progress tracking accurate

### Phase 5: Testing & Performance (1 day)

#### Tasks
1. **Performance Testing**
   - Large scoreboard rendering
   - Real-time update efficiency
   - Database query optimization

2. **Integration Testing**
   - Real-time WebSocket connectivity
   - Scoreboard calculation accuracy
   - Cross-browser compatibility

#### Acceptance Criteria
- [ ] Scoreboard loads in <2 seconds for 500 players
- [ ] Real-time updates delivered in <1 second
- [ ] Calculation accuracy validated
- [ ] WebSocket connections stable
- [ ] Mobile responsiveness verified

## Risk Assessment

### High Risk
- **Real-time performance**: Ensuring efficient updates for large tournaments
- **Calculation accuracy**: Complex tie-break and ranking algorithms
- **WebSocket stability**: Maintaining live connections reliably

### Medium Risk
- **Database performance**: Optimizing queries for large datasets
- **Mobile responsiveness**: Complex table layouts on small screens

### Low Risk
- **Basic display functionality**: Standard table and card components
- **Authentication integration**: Leveraging existing auth system

## Success Metrics

- Scoreboard load time < 2 seconds for 500 players
- Real-time update latency < 1 second
- 99.9% uptime for live updates
- Zero calculation errors in production
- Mobile usability score > 90%

## Dependencies for Next Phase

This scoreboard system enables:
- Real-time standings calculation
- Tournament progress monitoring
- Dashboard integration
- Live tournament broadcasting
