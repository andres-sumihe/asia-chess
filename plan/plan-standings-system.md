# Standings System Implementation Plan

## Feature Overview

Implement comprehensive tournament standings calculation system with real-time ranking updates, multiple tiebreak methods, and flexible scoring systems for different tournament formats.

## Objectives

- Real-time standings calculation and updates
- Multiple tiebreak method support (Buchholz, Sonneborn-Berger, Progressive)
- Tournament format-specific ranking algorithms
- Historical standings tracking and progression
- Performance rating calculations
- Automated ranking updates on game completion

## Technical Requirements

### Dependencies
- Scoreboard System (completed)
- Game Management (completed)
- Tournament Management (completed)
- Authentication system (completed)
- Real-time update infrastructure

### Constraints
- Calculations must be accurate and consistent
- Rankings must update in real-time (<1 second)
- Support large tournaments (500+ participants)
- Maintain historical ranking data
- Handle complex tiebreak scenarios

## Implementation Phases

### Phase 1: Enhanced Database Schema (2 days)

#### Tasks
1. **Create Standings Calculation Tables**
   - Tournament standings snapshots
   - Historical ranking progression
   - Tiebreak calculation cache

2. **Add Standings Configuration**
   - Tournament-specific tiebreak rules
   - Scoring system preferences
   - Performance rating settings

#### Database Changes

```prisma
// Enhanced Tournament model
model Tournament {
  // Existing fields...
  
  // Standings configuration
  standingsConfig Json?        // Tiebreak rules and scoring settings
  tiebreakMethods String[]     @default(["BUCHHOLZ", "SONNEBORN_BERGER", "PROGRESSIVE"])
  scoringSystem   ScoringSystem @default(STANDARD)
  calculatePerformance Boolean @default(true)
  
  // Historical tracking
  standingsSnapshots StandingsSnapshot[]
  
  // Display preferences
  showTiebreaks   Boolean     @default(true)
  showPerformance Boolean     @default(true)
  showProgression Boolean     @default(false)
}

// Tournament standings snapshots for history
model StandingsSnapshot {
  id           String    @id @default(cuid())
  tournamentId String
  roundNumber  Int
  timestamp    DateTime  @default(now())
  
  // Snapshot data
  rankings     Json      // Complete rankings at this point
  metadata     Json?     // Additional calculation metadata
  
  // Relations
  tournament   Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  
  @@unique([tournamentId, roundNumber])
  @@index([tournamentId, timestamp])
}

// Enhanced standings entries with full calculation data
model StandingsEntry {
  id              String    @id @default(cuid())
  tournamentId    String
  participantId   String
  userId          String
  
  // Current position
  currentRank     Int
  previousRank    Int?
  rankChange      Int?      // +1, -1, 0
  
  // Scoring data
  totalScore      Float     @default(0)
  gamePoints      Float     @default(0)  // Actual points from wins/draws
  bonusPoints     Float     @default(0)  // Additional scoring adjustments
  
  // Game statistics
  gamesPlayed     Int       @default(0)
  wins            Int       @default(0)
  draws           Int       @default(0)
  losses          Int       @default(0)
  forfeits        Int       @default(0)
  byes            Int       @default(0)
  
  // Tiebreak calculations
  buchholzScore   Float?    // Sum of opponents' scores
  buchholzCut1    Float?    // Buchholz minus lowest opponent
  sonnebornBerger Float?    // Performance-weighted score
  progressiveScore Float?   // Cumulative round-by-round
  directEncounter Float?    // Head-to-head results
  mostWins        Int?      // Win count tiebreak
  colorBalance    Json?     // White/black game distribution
  
  // Performance metrics
  performanceRating Float?  // Tournament performance rating
  averageOpponent Float?    // Average rating of opponents
  strengthOfSchedule Float? // Difficulty metric
  expectedScore   Float?    // Rating-based expected performance
  
  // Tournament context
  seedNumber      Int?      // Starting seed position
  seedRating      Int?      // Rating at tournament start
  ratingChange    Int?      // Projected rating change
  
  // Calculation metadata
  lastCalculated  DateTime  @updatedAt
  calculationVersion String @default("1.0")
  
  // Relations
  tournament      Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  participant     TournamentParticipant @relation(fields: [participantId], references: [id], onDelete: Cascade)
  user            User      @relation(fields: [userId], references: [id])
  
  @@unique([tournamentId, participantId])
  @@index([tournamentId, currentRank])
  @@index([tournamentId, totalScore, buchholzScore])
  @@index([userId, tournamentId])
}

// Historical rank progression tracking
model RankProgression {
  id             String    @id @default(cuid())
  standingsId    String
  roundNumber    Int
  rank           Int
  score          Float
  tiebreakData   Json?     // Tiebreak values at this round
  timestamp      DateTime  @default(now())
  
  // Relations  
  standings      StandingsEntry @relation(fields: [standingsId], references: [id], onDelete: Cascade)
  
  @@index([standingsId, roundNumber])
}

enum ScoringSystem {
  STANDARD        // 1-0.5-0 (win-draw-loss)
  FOOTBALL        // 3-1-0 (win-draw-loss)
  KASHDAN         // 4-2-1-0 (win-draw-loss-bye)
  CUSTOM          // User-defined
}
```

#### Acceptance Criteria
- [ ] Enhanced database schema created
- [ ] Standings calculation tables optimized
- [ ] Historical tracking system functional
- [ ] Performance indexes implemented
- [ ] Migration scripts tested

### Phase 2: Standings Calculation Engine (3 days)

#### Tasks
1. **Create Calculation Algorithms**
   - Multiple tiebreak method implementations
   - Scoring system variations
   - Performance rating calculations

2. **Implement Ranking Logic**
   - Real-time ranking updates
   - Historical comparison tracking
   - Efficient batch calculations

#### Implementation

```typescript
// src/lib/standings/calculator.ts
export interface TiebreakConfig {
  method: 'BUCHHOLZ' | 'BUCHHOLZ_CUT_1' | 'SONNEBORN_BERGER' | 'PROGRESSIVE' | 'DIRECT_ENCOUNTER' | 'MOST_WINS' | 'RATING_PERFORMANCE';
  weight: number;
  ascending?: boolean;
}

export interface StandingsConfig {
  scoringSystem: ScoringSystem;
  tiebreakMethods: TiebreakConfig[];
  calculatePerformance: boolean;
  includeForfeits: boolean;
  includeByeScore: boolean;
  byePointValue: number;
}

export class StandingsCalculator {
  private tournament: Tournament;
  private games: Game[];
  private participants: TournamentParticipant[];
  private config: StandingsConfig;

  constructor(
    tournament: Tournament, 
    games: Game[], 
    participants: TournamentParticipant[],
    config: StandingsConfig
  ) {
    this.tournament = tournament;
    this.games = games;
    this.participants = participants;
    this.config = config;
  }

  // Calculate complete standings for tournament
  async calculateStandings(): Promise<StandingsEntry[]> {
    const entries: StandingsEntry[] = [];

    for (const participant of this.participants) {
      const entry = await this.calculateParticipantStandings(participant);
      entries.push(entry);
    }

    // Sort by scoring and tiebreaks
    return this.rankEntries(entries);
  }

  // Calculate individual participant standings
  private async calculateParticipantStandings(participant: TournamentParticipant): Promise<StandingsEntry> {
    const userGames = this.getUserGames(participant.userId);
    
    // Basic scoring
    const scoringData = this.calculateScore(userGames, participant.userId);
    
    // Tiebreak calculations
    const tiebreakData = await this.calculateTiebreaks(participant.userId, userGames);
    
    // Performance metrics
    const performanceData = await this.calculatePerformanceMetrics(participant.userId, userGames);
    
    return {
      tournamentId: this.tournament.id,
      participantId: participant.id,
      userId: participant.userId,
      ...scoringData,
      ...tiebreakData,
      ...performanceData,
      lastCalculated: new Date(),
      calculationVersion: "1.0",
    };
  }

  // Calculate basic scoring based on game results
  private calculateScore(games: Game[], userId: string): {
    totalScore: number;
    gamePoints: number;
    bonusPoints: number;
    gamesPlayed: number;
    wins: number;
    draws: number;
    losses: number;
    forfeits: number;
    byes: number;
  } {
    let gamePoints = 0;
    let bonusPoints = 0;
    let wins = 0;
    let draws = 0;
    let losses = 0;
    let forfeits = 0;
    let byes = 0;

    for (const game of games) {
      if (game.status !== 'COMPLETED' || !game.result) continue;

      const isWhite = game.whiteId === userId;
      const isBye = game.whiteId === game.blackId; // Bye round indicator

      if (isBye) {
        byes++;
        if (this.config.includeByeScore) {
          bonusPoints += this.config.byePointValue;
        }
        continue;
      }

      switch (game.result) {
        case 'WHITE_WINS':
          if (isWhite) {
            gamePoints += this.getWinPoints();
            wins++;
          } else {
            gamePoints += this.getLossPoints();
            losses++;
          }
          break;
        case 'BLACK_WINS':
          if (!isWhite) {
            gamePoints += this.getWinPoints();
            wins++;
          } else {
            gamePoints += this.getLossPoints();
            losses++;
          }
          break;
        case 'DRAW':
          gamePoints += this.getDrawPoints();
          draws++;
          break;
        case 'WHITE_FORFEIT':
          if (!isWhite) {
            gamePoints += this.getWinPoints();
            wins++;
          } else {
            gamePoints += this.getLossPoints();
            forfeits++;
          }
          break;
        case 'BLACK_FORFEIT':
          if (isWhite) {
            gamePoints += this.getWinPoints();
            wins++;
          } else {
            gamePoints += this.getLossPoints();
            forfeits++;
          }
          break;
      }
    }

    const gamesPlayed = wins + draws + losses + forfeits;
    const totalScore = gamePoints + bonusPoints;

    return {
      totalScore,
      gamePoints,
      bonusPoints,
      gamesPlayed,
      wins,
      draws,
      losses,
      forfeits,
      byes,
    };
  }

  // Get point values based on scoring system
  private getWinPoints(): number {
    switch (this.config.scoringSystem) {
      case 'STANDARD': return 1;
      case 'FOOTBALL': return 3;
      case 'KASHDAN': return 4;
      default: return 1;
    }
  }

  private getDrawPoints(): number {
    switch (this.config.scoringSystem) {
      case 'STANDARD': return 0.5;
      case 'FOOTBALL': return 1;
      case 'KASHDAN': return 2;
      default: return 0.5;
    }
  }

  private getLossPoints(): number {
    switch (this.config.scoringSystem) {
      case 'KASHDAN': return 1;
      default: return 0;
    }
  }

  // Calculate all tiebreak methods
  private async calculateTiebreaks(userId: string, userGames: Game[]): Promise<{
    buchholzScore?: number;
    buchholzCut1?: number;
    sonnebornBerger?: number;
    progressiveScore?: number;
    directEncounter?: number;
    mostWins?: number;
    colorBalance?: any;
  }> {
    const tiebreakData: any = {};

    for (const tiebreakConfig of this.config.tiebreakMethods) {
      switch (tiebreakConfig.method) {
        case 'BUCHHOLZ':
          tiebreakData.buchholzScore = await this.calculateBuchholz(userId, userGames);
          break;
        case 'BUCHHOLZ_CUT_1':
          tiebreakData.buchholzCut1 = await this.calculateBuchholzCut1(userId, userGames);
          break;
        case 'SONNEBORN_BERGER':
          tiebreakData.sonnebornBerger = await this.calculateSonnebornBerger(userId, userGames);
          break;
        case 'PROGRESSIVE':
          tiebreakData.progressiveScore = await this.calculateProgressive(userId, userGames);
          break;
        case 'DIRECT_ENCOUNTER':
          tiebreakData.directEncounter = await this.calculateDirectEncounter(userId, userGames);
          break;
        case 'MOST_WINS':
          tiebreakData.mostWins = this.calculateMostWins(userId, userGames);
          break;
      }
    }

    // Color balance calculation
    tiebreakData.colorBalance = this.calculateColorBalance(userId, userGames);

    return tiebreakData;
  }

  // Buchholz system: sum of opponents' scores
  private async calculateBuchholz(userId: string, userGames: Game[]): Promise<number> {
    let buchholzSum = 0;

    for (const game of userGames) {
      if (game.status !== 'COMPLETED') continue;

      const opponentId = game.whiteId === userId ? game.blackId : game.whiteId;
      const opponentScore = await this.getOpponentTotalScore(opponentId);
      buchholzSum += opponentScore;
    }

    return buchholzSum;
  }

  // Buchholz Cut 1: Buchholz minus lowest opponent score
  private async calculateBuchholzCut1(userId: string, userGames: Game[]): Promise<number> {
    const opponentScores: number[] = [];

    for (const game of userGames) {
      if (game.status !== 'COMPLETED') continue;

      const opponentId = game.whiteId === userId ? game.blackId : game.whiteId;
      const opponentScore = await this.getOpponentTotalScore(opponentId);
      opponentScores.push(opponentScore);
    }

    if (opponentScores.length <= 1) {
      return opponentScores.reduce((sum, score) => sum + score, 0);
    }

    // Remove lowest score
    opponentScores.sort((a, b) => a - b);
    opponentScores.shift();

    return opponentScores.reduce((sum, score) => sum + score, 0);
  }

  // Sonneborn-Berger: weighted by game results
  private async calculateSonnebornBerger(userId: string, userGames: Game[]): Promise<number> {
    let sonnebornSum = 0;

    for (const game of userGames) {
      if (game.status !== 'COMPLETED' || !game.result) continue;

      const opponentId = game.whiteId === userId ? game.blackId : game.whiteId;
      const opponentScore = await this.getOpponentTotalScore(opponentId);
      const gameResult = this.getGameResult(game, userId);

      sonnebornSum += gameResult * opponentScore;
    }

    return sonnebornSum;
  }

  // Progressive (cumulative) scoring
  private async calculateProgressive(userId: string, userGames: Game[]): Promise<number> {
    let progressiveSum = 0;
    let cumulativeScore = 0;

    // Sort games by round
    const sortedGames = userGames
      .filter(g => g.status === 'COMPLETED')
      .sort((a, b) => a.round.roundNumber - b.round.roundNumber);

    for (const game of sortedGames) {
      const gameResult = this.getGameResult(game, userId);
      cumulativeScore += gameResult;
      progressiveSum += cumulativeScore;
    }

    return progressiveSum;
  }

  // Direct encounter between tied players
  private async calculateDirectEncounter(userId: string, userGames: Game[]): Promise<number> {
    // This would be calculated when comparing specific tied players
    // For now, return 0 as it's contextual
    return 0;
  }

  // Most wins tiebreak
  private calculateMostWins(userId: string, userGames: Game[]): number {
    let wins = 0;

    for (const game of userGames) {
      if (game.status !== 'COMPLETED' || !game.result) continue;

      const isWhite = game.whiteId === userId;
      
      if ((game.result === 'WHITE_WINS' && isWhite) || 
          (game.result === 'BLACK_WINS' && !isWhite)) {
        wins++;
      }
    }

    return wins;
  }

  // Color balance calculation
  private calculateColorBalance(userId: string, userGames: Game[]): { white: number; black: number } {
    let whiteGames = 0;
    let blackGames = 0;

    for (const game of userGames) {
      if (game.status !== 'COMPLETED') continue;

      if (game.whiteId === userId) {
        whiteGames++;
      } else {
        blackGames++;
      }
    }

    return { white: whiteGames, black: blackGames };
  }

  // Calculate performance metrics
  private async calculatePerformanceMetrics(userId: string, userGames: Game[]): Promise<{
    performanceRating?: number;
    averageOpponent?: number;
    strengthOfSchedule?: number;
    expectedScore?: number;
  }> {
    if (!this.config.calculatePerformance) {
      return {};
    }

    const opponentRatings: number[] = [];
    let totalScore = 0;

    for (const game of userGames) {
      if (game.status !== 'COMPLETED') continue;

      const opponentId = game.whiteId === userId ? game.blackId : game.whiteId;
      const opponentRating = await this.getOpponentRating(opponentId);
      
      if (opponentRating) {
        opponentRatings.push(opponentRating);
        totalScore += this.getGameResult(game, userId);
      }
    }

    if (opponentRatings.length === 0) {
      return {};
    }

    const averageOpponent = opponentRatings.reduce((sum, rating) => sum + rating, 0) / opponentRatings.length;
    const scorePercentage = totalScore / opponentRatings.length;
    const performanceRating = this.calculatePerformanceRating(averageOpponent, scorePercentage);
    const strengthOfSchedule = this.calculateStrengthOfSchedule(opponentRatings);
    const expectedScore = this.calculateExpectedScore(userId, opponentRatings);

    return {
      performanceRating,
      averageOpponent,
      strengthOfSchedule,
      expectedScore,
    };
  }

  // Rank entries by score and tiebreaks
  private rankEntries(entries: StandingsEntry[]): StandingsEntry[] {
    return entries.sort((a, b) => {
      // Primary: total score
      if (a.totalScore !== b.totalScore) {
        return b.totalScore - a.totalScore;
      }

      // Apply tiebreak methods in order
      for (const tiebreakConfig of this.config.tiebreakMethods) {
        const comparison = this.compareTiebreak(a, b, tiebreakConfig);
        if (comparison !== 0) {
          return comparison;
        }
      }

      return 0;
    }).map((entry, index) => ({
      ...entry,
      currentRank: index + 1,
    }));
  }

  // Compare two entries using specific tiebreak method
  private compareTiebreak(a: StandingsEntry, b: StandingsEntry, config: TiebreakConfig): number {
    let aValue: number | undefined;
    let bValue: number | undefined;

    switch (config.method) {
      case 'BUCHHOLZ':
        aValue = a.buchholzScore;
        bValue = b.buchholzScore;
        break;
      case 'BUCHHOLZ_CUT_1':
        aValue = a.buchholzCut1;
        bValue = b.buchholzCut1;
        break;
      case 'SONNEBORN_BERGER':
        aValue = a.sonnebornBerger;
        bValue = b.sonnebornBerger;
        break;
      case 'PROGRESSIVE':
        aValue = a.progressiveScore;
        bValue = b.progressiveScore;
        break;
      case 'MOST_WINS':
        aValue = a.mostWins;
        bValue = b.mostWins;
        break;
      case 'RATING_PERFORMANCE':
        aValue = a.performanceRating;
        bValue = b.performanceRating;
        break;
    }

    if (aValue === undefined || bValue === undefined) {
      return 0;
    }

    const multiplier = config.ascending ? 1 : -1;
    return (bValue - aValue) * multiplier * config.weight;
  }

  // Helper methods
  private getUserGames(userId: string): Game[] {
    return this.games.filter(game => 
      game.whiteId === userId || game.blackId === userId
    );
  }

  private getGameResult(game: Game, userId: string): number {
    if (!game.result) return 0;

    const isWhite = game.whiteId === userId;
    
    switch (game.result) {
      case 'WHITE_WINS':
        return isWhite ? this.getWinPoints() : this.getLossPoints();
      case 'BLACK_WINS':
        return !isWhite ? this.getWinPoints() : this.getLossPoints();
      case 'DRAW':
        return this.getDrawPoints();
      case 'WHITE_FORFEIT':
        return !isWhite ? this.getWinPoints() : this.getLossPoints();
      case 'BLACK_FORFEIT':
        return isWhite ? this.getWinPoints() : this.getLossPoints();
      default:
        return 0;
    }
  }

  private async getOpponentTotalScore(opponentId: string): Promise<number> {
    // Implementation to get opponent's total score
    return 0; // Placeholder
  }

  private async getOpponentRating(opponentId: string): Promise<number | null> {
    // Implementation to get opponent's rating
    return null; // Placeholder
  }

  private calculatePerformanceRating(averageOpponent: number, scorePercentage: number): number {
    // Standard ELO performance rating calculation
    const expectedScore = 1 / (1 + Math.pow(10, 0 / 400));
    const performancePoints = 400 * Math.log10(scorePercentage / (1 - scorePercentage));
    return Math.round(averageOpponent + performancePoints);
  }

  private calculateStrengthOfSchedule(opponentRatings: number[]): number {
    const avgRating = opponentRatings.reduce((sum, rating) => sum + rating, 0) / opponentRatings.length;
    const variance = opponentRatings.reduce((sum, rating) => sum + Math.pow(rating - avgRating, 2), 0) / opponentRatings.length;
    return avgRating + Math.sqrt(variance) * 0.1; // Weighted by rating spread
  }

  private calculateExpectedScore(userId: string, opponentRatings: number[]): number {
    // Implementation for expected score calculation
    return 0; // Placeholder
  }
}

// Real-time standings update service
export class StandingsUpdateService {
  private db: PrismaClient;
  private wsServer: WebSocketServer;

  constructor(db: PrismaClient, wsServer: WebSocketServer) {
    this.db = db;
    this.wsServer = wsServer;
  }

  // Update standings when game result changes
  async updateStandingsForGame(gameId: string): Promise<void> {
    const game = await this.db.game.findUnique({
      where: { id: gameId },
      include: {
        round: { include: { tournament: true } },
        white: { include: { profile: true } },
        black: { include: { profile: true } },
      },
    });

    if (!game) return;

    // Recalculate standings for tournament
    await this.recalculateStandings(game.round.tournamentId);
    
    // Broadcast update to subscribers
    await this.broadcastStandingsUpdate(game.round.tournamentId, {
      type: 'STANDINGS_UPDATE',
      gameId,
      roundNumber: game.round.roundNumber,
      affectedPlayers: [game.whiteId, game.blackId],
    });
  }

  // Recalculate complete standings
  async recalculateStandings(tournamentId: string): Promise<void> {
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

    // Parse standings configuration
    const standingsConfig: StandingsConfig = {
      scoringSystem: tournament.scoringSystem || 'STANDARD',
      tiebreakMethods: (tournament.standingsConfig as any)?.tiebreakMethods || [
        { method: 'BUCHHOLZ_CUT_1', weight: 1 },
        { method: 'BUCHHOLZ', weight: 1 },
        { method: 'SONNEBORN_BERGER', weight: 1 },
        { method: 'PROGRESSIVE', weight: 1 },
      ],
      calculatePerformance: tournament.calculatePerformance,
      includeForfeits: true,
      includeByeScore: true,
      byePointValue: 0.5,
    };

    const calculator = new StandingsCalculator(
      tournament,
      tournament.rounds.flatMap(r => r.games),
      tournament.participants,
      standingsConfig
    );

    const newStandings = await calculator.calculateStandings();

    // Update database with new standings
    await this.db.$transaction(async (tx) => {
      // Get previous standings for rank change calculation
      const previousStandings = await tx.standingsEntry.findMany({
        where: { tournamentId },
        select: { participantId: true, currentRank: true },
      });

      const previousRankMap = new Map(
        previousStandings.map(s => [s.participantId, s.currentRank])
      );

      // Delete existing standings
      await tx.standingsEntry.deleteMany({
        where: { tournamentId },
      });

      // Create new standings with rank change calculation
      const standingsWithRankChange = newStandings.map(standing => {
        const previousRank = previousRankMap.get(standing.participantId);
        const rankChange = previousRank ? previousRank - standing.currentRank : 0;

        return {
          ...standing,
          previousRank,
          rankChange,
        };
      });

      await tx.standingsEntry.createMany({
        data: standingsWithRankChange,
      });

      // Create standings snapshot
      await tx.standingsSnapshot.create({
        data: {
          tournamentId,
          roundNumber: tournament.currentRound,
          rankings: newStandings,
          metadata: {
            calculatedAt: new Date(),
            totalParticipants: newStandings.length,
            config: standingsConfig,
          },
        },
      });
    });
  }

  // Broadcast update to WebSocket subscribers
  private async broadcastStandingsUpdate(tournamentId: string, update: any): Promise<void> {
    const subscribers = await this.db.scoreboardSubscription.findMany({
      where: { 
        tournamentId,
        isActive: true,
        standingsUpdates: true,
      },
    });

    for (const subscriber of subscribers) {
      this.wsServer.send(subscriber.sessionId, {
        type: 'STANDINGS_UPDATE',
        data: update,
      });
    }
  }
}
```

#### Acceptance Criteria
- [ ] Standings calculation engine implemented
- [ ] Multiple tiebreak methods functional
- [ ] Performance rating calculations accurate
- [ ] Real-time update triggers working
- [ ] Historical tracking operational

### Phase 3: API Implementation (2 days)

#### Tasks
1. **Create Standings tRPC Router**
   - Tournament standings retrieval
   - Historical progression data
   - Rankings comparison endpoints

2. **Implement Real-time Integration**
   - Automatic calculation triggers
   - WebSocket standings updates
   - Calculation status tracking

#### Implementation

```typescript
// src/server/api/routers/standings.ts
export const standingsRouter = createTRPCRouter({
  // Get tournament standings
  getByTournament: publicProcedure
    .input(z.object({
      tournamentId: z.string(),
      roundNumber: z.number().optional(),
      limit: z.number().min(1).max(1000).default(100),
      includeHistory: z.boolean().default(false),
    }))
    .query(async ({ ctx, input }) => {
      const { tournamentId, roundNumber, limit, includeHistory } = input;

      // Get tournament and verify access
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

      // Check if standings are public or user has access
      const hasAccess = tournament.publicStandings ||
                       tournament.createdById === ctx.session?.user?.id ||
                       tournament.organizers.some(o => o.userId === ctx.session?.user?.id) ||
                       ctx.session?.user?.role === "ADMIN";

      if (!hasAccess) {
        throw new TRPCError({
          code: "FORBIDDEN",
          message: "These standings are private",
        });
      }

      // Get current standings
      const standings = await ctx.db.standingsEntry.findMany({
        where: { tournamentId },
        take: limit,
        orderBy: [
          { currentRank: 'asc' },
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

      // Get historical data if requested
      let history: RankProgression[] = [];
      if (includeHistory) {
        history = await ctx.db.rankProgression.findMany({
          where: {
            standingsId: { in: standings.map(s => s.id) },
          },
          orderBy: [
            { roundNumber: 'asc' },
          ],
        });
      }

      // Get tournament metadata
      const [
        totalParticipants,
        currentRound,
        totalRounds,
        lastSnapshot,
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
        ctx.db.standingsSnapshot.findFirst({
          where: { tournamentId },
          orderBy: { timestamp: 'desc' },
        }),
      ]);

      return {
        standings,
        history: includeHistory ? history : undefined,
        metadata: {
          tournamentName: tournament.name,
          totalParticipants,
          currentRound: currentRound?.roundNumber || 0,
          totalRounds,
          lastCalculated: lastSnapshot?.timestamp,
          scoringSystem: tournament.scoringSystem,
          tiebreakMethods: tournament.tiebreakMethods,
        },
        config: {
          showTiebreaks: tournament.showTiebreaks,
          showPerformance: tournament.showPerformance,
          showProgression: tournament.showProgression,
        },
      };
    }),

  // Get player's ranking progression
  getPlayerProgression: publicProcedure
    .input(z.object({
      tournamentId: z.string(),
      userId: z.string().optional(),
    }))
    .query(async ({ ctx, input }) => {
      const userId = input.userId || ctx.session?.user?.id;
      const { tournamentId } = input;

      if (!userId) {
        throw new TRPCError({
          code: "UNAUTHORIZED",
          message: "User ID required",
        });
      }

      // Get player's standings entry
      const standingsEntry = await ctx.db.standingsEntry.findUnique({
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

      if (!standingsEntry) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Player not found in tournament standings",
        });
      }

      // Get rank progression history
      const progression = await ctx.db.rankProgression.findMany({
        where: {
          standingsId: standingsEntry.id,
        },
        orderBy: [
          { roundNumber: 'asc' },
        ],
      });

      // Get game-by-game results for context
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

      return {
        player: standingsEntry,
        progression,
        games,
        statistics: {
          currentRank: standingsEntry.currentRank,
          bestRank: Math.min(...progression.map(p => p.rank), standingsEntry.currentRank),
          worstRank: Math.max(...progression.map(p => p.rank), standingsEntry.currentRank),
          rankImprovement: standingsEntry.previousRank ? standingsEntry.previousRank - standingsEntry.currentRank : 0,
        },
      };
    }),

  // Get standings comparison between rounds
  getComparison: publicProcedure
    .input(z.object({
      tournamentId: z.string(),
      fromRound: z.number(),
      toRound: z.number(),
    }))
    .query(async ({ ctx, input }) => {
      const { tournamentId, fromRound, toRound } = input;

      const [fromSnapshot, toSnapshot] = await Promise.all([
        ctx.db.standingsSnapshot.findUnique({
          where: {
            tournamentId_roundNumber: {
              tournamentId,
              roundNumber: fromRound,
            },
          },
        }),
        ctx.db.standingsSnapshot.findUnique({
          where: {
            tournamentId_roundNumber: {
              tournamentId,
              roundNumber: toRound,
            },
          },
        }),
      ]);

      if (!fromSnapshot || !toSnapshot) {
        throw new TRPCError({
          code: "NOT_FOUND",
          message: "Standings snapshots not found for comparison",
        });
      }

      // Calculate changes between snapshots
      const fromRankings = fromSnapshot.rankings as StandingsEntry[];
      const toRankings = toSnapshot.rankings as StandingsEntry[];

      const comparison = toRankings.map(toEntry => {
        const fromEntry = fromRankings.find(f => f.participantId === toEntry.participantId);
        
        return {
          participant: toEntry,
          changes: {
            rankChange: fromEntry ? fromEntry.currentRank - toEntry.currentRank : 0,
            scoreChange: fromEntry ? toEntry.totalScore - fromEntry.totalScore : toEntry.totalScore,
            positionMovement: fromEntry ? toEntry.currentRank - fromEntry.currentRank : 0,
          },
          from: fromEntry,
          to: toEntry,
        };
      });

      return {
        fromRound,
        toRound,
        comparison,
        summary: {
          biggestRiser: comparison.reduce((max, curr) => 
            curr.changes.rankChange > max.changes.rankChange ? curr : max
          ),
          biggestFaller: comparison.reduce((min, curr) => 
            curr.changes.rankChange < min.changes.rankChange ? curr : min
          ),
          averageScoreChange: comparison.reduce((sum, curr) => sum + curr.changes.scoreChange, 0) / comparison.length,
        },
      };
    }),

  // Get tiebreak details for tied players
  getTiebreakDetails: publicProcedure
    .input(z.object({
      tournamentId: z.string(),
      score: z.number(),
    }))
    .query(async ({ ctx, input }) => {
      const { tournamentId, score } = input;

      // Get all players with the same score
      const tiedPlayers = await ctx.db.standingsEntry.findMany({
        where: {
          tournamentId,
          totalScore: score,
        },
        orderBy: [
          { currentRank: 'asc' },
        ],
        include: {
          user: {
            select: {
              name: true,
              profile: {
                select: {
                  firstName: true,
                  lastName: true,
                },
              },
            },
          },
        },
      });

      if (tiedPlayers.length <= 1) {
        return {
          players: tiedPlayers,
          tiebreakExplanation: "No tiebreak needed - only one player with this score",
        };
      }

      // Get tournament tiebreak configuration
      const tournament = await ctx.db.tournament.findUnique({
        where: { id: tournamentId },
        select: {
          tiebreakMethods: true,
          standingsConfig: true,
        },
      });

      return {
        players: tiedPlayers,
        tiebreakMethods: tournament?.tiebreakMethods || [],
        explanation: this.generateTiebreakExplanation(tiedPlayers, tournament?.tiebreakMethods || []),
      };
    }),

  // Force standings recalculation (organizer only)
  recalculateStandings: protectedProcedure
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
          message: "You don't have permission to recalculate standings",
        });
      }

      // Recalculate standings
      const updateService = new StandingsUpdateService(ctx.db, ctx.wsServer);
      await updateService.recalculateStandings(tournamentId);

      return { success: true, recalculatedAt: new Date() };
    }),

  // Get standings statistics
  getStatistics: publicProcedure
    .input(z.object({
      tournamentId: z.string(),
    }))
    .query(async ({ ctx, input }) => {
      const { tournamentId } = input;

      const [
        standings,
        snapshots,
        tournament,
      ] = await Promise.all([
        ctx.db.standingsEntry.findMany({
          where: { tournamentId },
          orderBy: { currentRank: 'asc' },
          include: {
            user: {
              select: {
                profile: {
                  select: { fideRating: true, localRating: true },
                },
              },
            },
          },
        }),
        ctx.db.standingsSnapshot.count({
          where: { tournamentId },
        }),
        ctx.db.tournament.findUnique({
          where: { id: tournamentId },
          select: {
            currentRound: true,
            format: true,
          },
        }),
      ]);

      const totalPlayers = standings.length;
      const averageScore = standings.reduce((sum, s) => sum + s.totalScore, 0) / totalPlayers;
      const scoreDistribution = this.calculateScoreDistribution(standings);
      const ratingDistribution = this.calculateRatingDistribution(standings);

      return {
        totalPlayers,
        averageScore,
        scoreDistribution,
        ratingDistribution,
        leaderInfo: standings.slice(0, 3),
        currentRound: tournament?.currentRound || 0,
        totalSnapshots: snapshots,
        performanceStats: {
          averagePerformance: standings.reduce((sum, s) => sum + (s.performanceRating || 0), 0) / totalPlayers,
          topPerformers: standings
            .filter(s => s.performanceRating)
            .sort((a, b) => (b.performanceRating || 0) - (a.performanceRating || 0))
            .slice(0, 5),
        },
      };
    }),
});
```

#### Acceptance Criteria
- [ ] Standings API endpoints functional
- [ ] Real-time calculation triggers working
- [ ] Historical progression tracking accurate
- [ ] Tiebreak details properly exposed
- [ ] Statistics calculations correct

### Phase 4: Frontend Components (2 days)

#### Tasks
1. **Create Standings Display Components**
   - Tournament standings table
   - Player progression charts
   - Tiebreak explanations

2. **Implement Interactive Features**
   - Historical comparison views
   - Player detail modals
   - Real-time rank updates

#### Implementation

```typescript
// src/app/_components/standings/tournament-standings.tsx
'use client';

import { useState, useMemo } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { Badge } from '@/components/ui/badge';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { TrendingUp, TrendingDown, Search, History, Trophy, Medal } from 'lucide-react';
import { api } from '@/trpc/react';
import { useWebSocket } from '@/hooks/use-websocket';

interface TournamentStandingsProps {
  tournamentId: string;
  showControls?: boolean;
  compact?: boolean;
  maxHeight?: string;
}

export function TournamentStandings({ 
  tournamentId, 
  showControls = true, 
  compact = false,
  maxHeight = "600px" 
}: TournamentStandingsProps) {
  const [searchTerm, setSearchTerm] = useState("");
  const [includeHistory, setIncludeHistory] = useState(false);
  const [selectedRound, setSelectedRound] = useState<number | undefined>();

  // Fetch standings data
  const {
    data: standingsData,
    isLoading,
    refetch,
  } = api.standings.getByTournament.useQuery({
    tournamentId,
    includeHistory,
    roundNumber: selectedRound,
  });

  // WebSocket for real-time updates
  const { isConnected } = useWebSocket(
    `ws://localhost:3000/ws/standings/${tournamentId}`,
    {
      onMessage: (message) => {
        const data = JSON.parse(message.data);
        if (data.type === 'STANDINGS_UPDATE') {
          refetch();
        }
      },
    }
  );

  // Filter standings based on search term
  const filteredStandings = useMemo(() => {
    if (!standingsData?.standings) return [];
    
    if (!searchTerm) return standingsData.standings;

    return standingsData.standings.filter((entry) => {
      const playerName = entry.user.name || 
        `${entry.user.profile?.firstName} ${entry.user.profile?.lastName}`;
      return playerName.toLowerCase().includes(searchTerm.toLowerCase());
    });
  }, [standingsData?.standings, searchTerm]);

  const handlePlayerClick = (playerId: string) => {
    // Open player progression modal
    console.log(`Show progression for player ${playerId}`);
  };

  const handleRefresh = () => {
    refetch();
  };

  if (isLoading) {
    return (
      <Card>
        <CardContent className="p-6">
          <div className="flex items-center justify-center">
            <div className="animate-spin rounded-full h-8 w-8 border-b-2 border-primary"></div>
            <span className="ml-2">Loading standings...</span>
          </div>
        </CardContent>
      </Card>
    );
  }

  if (!standingsData) {
    return (
      <Card>
        <CardContent className="p-6">
          <div className="text-center text-muted-foreground">
            No standings data available
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
            <CardTitle>{standingsData.metadata.tournamentName} - Standings</CardTitle>
            <div className="text-sm text-muted-foreground">
              Round {standingsData.metadata.currentRound} of {standingsData.metadata.totalRounds} • 
              {standingsData.standings.length} participants
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
              >
                Refresh
              </Button>
              
              <Button
                variant="outline"
                size="sm"
                onClick={() => setIncludeHistory(!includeHistory)}
              >
                <History className="w-4 h-4 mr-1" />
                {includeHistory ? 'Hide' : 'Show'} History
              </Button>
            </div>
          )}
        </div>

        {showControls && (
          <div className="flex items-center gap-2 mt-4">
            <div className="relative flex-1">
              <Search className="absolute left-2 top-2.5 h-4 w-4 text-muted-foreground" />
              <Input
                placeholder="Search players..."
                value={searchTerm}
                onChange={(e) => setSearchTerm(e.target.value)}
                className="pl-8"
              />
            </div>
            
            {standingsData.metadata.totalRounds > 0 && (
              <Select value={selectedRound?.toString()} onValueChange={(value) => setSelectedRound(value ? parseInt(value) : undefined)}>
                <SelectTrigger className="w-40">
                  <SelectValue placeholder="All rounds" />
                </SelectTrigger>
                <SelectContent>
                  <SelectItem value="">All rounds</SelectItem>
                  {Array.from({ length: standingsData.metadata.totalRounds }, (_, i) => (
                    <SelectItem key={i + 1} value={(i + 1).toString()}>
                      Round {i + 1}
                    </SelectItem>
                  ))}
                </SelectContent>
              </Select>
            )}
          </div>
        )}
      </CardHeader>

      <CardContent>
        <div style={{ maxHeight, overflow: 'auto' }}>
          <Table>
            <TableHeader>
              <TableRow>
                <TableHead className="w-16">Rank</TableHead>
                <TableHead>Player</TableHead>
                {standingsData.config.showPerformance && !compact && (
                  <TableHead className="text-center">Rating</TableHead>
                )}
                <TableHead className="text-center">Score</TableHead>
                <TableHead className="text-center">Games</TableHead>
                {standingsData.config.showTiebreaks && (
                  <>
                    <TableHead className="text-center">TB1</TableHead>
                    <TableHead className="text-center">TB2</TableHead>
                    {!compact && (
                      <TableHead className="text-center">TB3</TableHead>
                    )}
                  </>
                )}
                {standingsData.config.showPerformance && !compact && (
                  <TableHead className="text-center">Perf</TableHead>
                )}
              </TableRow>
            </TableHeader>
            <TableBody>
              {filteredStandings.map((entry, index) => {
                const rankChange = entry.previousRank 
                  ? entry.previousRank - entry.currentRank 
                  : 0;

                return (
                  <TableRow 
                    key={entry.id} 
                    className={`cursor-pointer hover:bg-muted/50 ${index < 3 ? "bg-muted/30" : ""}`}
                    onClick={() => handlePlayerClick(entry.userId)}
                  >
                    <TableCell className="font-medium">
                      <div className="flex items-center">
                        {entry.currentRank === 1 && <Trophy className="w-4 h-4 text-yellow-500 mr-1" />}
                        {entry.currentRank === 2 && <Medal className="w-4 h-4 text-gray-400 mr-1" />}
                        {entry.currentRank === 3 && <Medal className="w-4 h-4 text-orange-500 mr-1" />}
                        {entry.currentRank}
                        {rankChange !== 0 && (
                          <span className="ml-1">
                            {rankChange > 0 ? (
                              <TrendingUp className="w-3 h-3 text-green-500" />
                            ) : (
                              <TrendingDown className="w-3 h-3 text-red-500" />
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
                        <div className="text-xs text-muted-foreground">
                          {entry.user.profile?.country && (
                            <span className="mr-2">{entry.user.profile.country}</span>
                          )}
                          {entry.user.profile?.title && (
                            <Badge variant="secondary" className="text-xs">
                              {entry.user.profile.title}
                            </Badge>
                          )}
                        </div>
                      </div>
                    </TableCell>

                    {standingsData.config.showPerformance && !compact && (
                      <TableCell className="text-center">
                        {entry.user.profile?.fideRating || 
                         entry.user.profile?.localRating || 
                         entry.participant.seedRating || "-"}
                      </TableCell>
                    )}

                    <TableCell className="text-center font-bold">
                      {entry.totalScore}
                      {entry.bonusPoints > 0 && (
                        <span className="text-xs text-muted-foreground ml-1">
                          (+{entry.bonusPoints})
                        </span>
                      )}
                    </TableCell>

                    <TableCell className="text-center">
                      <div className="text-sm">
                        <div>{entry.gamesPlayed}</div>
                        <div className="text-xs text-muted-foreground">
                          {entry.wins}W {entry.draws}D {entry.losses}L
                          {entry.forfeits > 0 && ` ${entry.forfeits}F`}
                          {entry.byes > 0 && ` ${entry.byes}B`}
                        </div>
                      </div>
                    </TableCell>

                    {standingsData.config.showTiebreaks && (
                      <>
                        <TableCell className="text-center">
                          {entry.buchholzCut1?.toFixed(1) || "-"}
                        </TableCell>
                        <TableCell className="text-center">
                          {entry.buchholzScore?.toFixed(1) || "-"}
                        </TableCell>
                        {!compact && (
                          <TableCell className="text-center">
                            {entry.sonnebornBerger?.toFixed(1) || "-"}
                          </TableCell>
                        )}
                      </>
                    )}

                    {standingsData.config.showPerformance && !compact && (
                      <TableCell className="text-center">
                        {entry.performanceRating || "-"}
                      </TableCell>
                    )}
                  </TableRow>
                );
              })}
            </TableBody>
          </Table>
        </div>

        {/* Tournament info footer */}
        <div className="mt-4 p-4 bg-muted rounded-lg">
          <div className="grid grid-cols-2 md:grid-cols-4 gap-4 text-sm">
            <div>
              <span className="font-medium">Scoring System:</span>
              <div className="text-muted-foreground">{standingsData.metadata.scoringSystem}</div>
            </div>
            <div>
              <span className="font-medium">Tiebreaks:</span>
              <div className="text-muted-foreground">
                {standingsData.metadata.tiebreakMethods.slice(0, 2).join(", ")}
              </div>
            </div>
            <div>
              <span className="font-medium">Last Updated:</span>
              <div className="text-muted-foreground">
                {standingsData.metadata.lastCalculated 
                  ? new Date(standingsData.metadata.lastCalculated).toLocaleTimeString()
                  : "Never"
                }
              </div>
            </div>
            <div>
              <span className="font-medium">Total Players:</span>
              <div className="text-muted-foreground">{standingsData.metadata.totalParticipants}</div>
            </div>
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

#### Acceptance Criteria
- [ ] Standings display responsive and functional
- [ ] Real-time updates working correctly
- [ ] Player search and filtering operational
- [ ] Historical data visualization accurate
- [ ] Tiebreak information clearly presented

### Phase 5: Testing & Performance (1 day)

#### Tasks
1. **Calculation Accuracy Testing**
   - Tiebreak algorithm validation
   - Edge case scenario testing
   - Cross-reference with known results

2. **Performance Optimization**
   - Large tournament calculation efficiency
   - Database query optimization
   - Real-time update performance

#### Acceptance Criteria
- [ ] Calculation accuracy verified
- [ ] Performance benchmarks met
- [ ] Edge cases handled correctly
- [ ] Real-time updates efficient
- [ ] Database queries optimized

## Risk Assessment

### High Risk
- **Calculation complexity**: Multiple tiebreak methods and edge cases
- **Performance at scale**: Large tournament standings calculations
- **Real-time accuracy**: Ensuring immediate updates are correct

### Medium Risk
- **Tiebreak edge cases**: Handling complex tied scenarios
- **Historical data integrity**: Maintaining accurate progression tracking

### Low Risk
- **UI complexity**: Standard table and chart components
- **API integration**: Leveraging existing patterns

## Success Metrics

- Standings calculation accuracy: 100%
- Update latency: <1 second for real-time changes
- Large tournament support: 500+ participants
- Tiebreak calculation time: <500ms
- 95% test coverage for calculation algorithms

## Dependencies for Next Phase

This standings system enables:
- Dashboard tournament overview
- Advanced tournament analytics
- Historical performance tracking
- Real-time tournament monitoring
