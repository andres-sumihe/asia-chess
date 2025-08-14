# Reports & Analytics Implementation Plan

## Feature Overview

Implement comprehensive reporting and analytics system providing tournament statistics, player performance insights, platform usage metrics, and data visualization capabilities for tournament organizers and administrators.

## Objectives

- Tournament performance analytics and statistics
- Player ranking and rating trend analysis
- Platform usage metrics and insights
- Custom report generation and scheduling
- Data visualization with interactive charts
- Export capabilities for external analysis

## Technical Requirements

### Dependencies
- All previous systems (tournaments, games, standings, dashboard, real-time)
- Authentication and authorization system
- Data aggregation and calculation engines
- Chart visualization libraries (Chart.js/Recharts)

### Constraints
- Large dataset query optimization required
- Historical data retention policies
- Export format standardization
- Performance optimization for complex analytics
- Secure access to sensitive analytics data

## Implementation Phases

### Phase 1: Analytics Data Model & Aggregation (2 days)

#### Tasks
1. **Create Analytics Database Schema**
   - Tournament statistics aggregation tables
   - Player performance metrics storage
   - System usage tracking models

2. **Implement Data Aggregation Engine**
   - Scheduled data processing jobs
   - Real-time metrics calculation
   - Historical data analysis pipelines

#### Database Changes

```prisma
// Tournament analytics and statistics
model TournamentAnalytics {
  id                String    @id @default(cuid())
  tournamentId      String    @unique
  tournament        Tournament @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  
  // Basic statistics
  totalParticipants Int
  totalGames        Int
  completedGames    Int
  averageGameDuration Float?  // in minutes
  totalDuration     Int?      // in minutes
  
  // Performance metrics
  avgRatingParticipants Float?
  ratingSpread      Float?    // difference between highest and lowest rated
  upsetCount        Int       @default(0) // lower rated beating higher rated
  drawPercentage    Float?
  
  // Engagement metrics
  registrationRate  Float?    // registrations per day leading up
  withdrawalRate    Float?    // percentage who withdrew
  completionRate    Float?    // percentage who completed all games
  
  // Technical metrics
  avgResponseTime   Float?    // average API response time during tournament
  errorCount        Int       @default(0)
  systemDowntime    Int       @default(0) // in minutes
  
  // Timestamps
  calculatedAt      DateTime  @default(now())
  updatedAt         DateTime  @updatedAt
  
  @@index([tournamentId])
  @@index([calculatedAt])
}

// Player performance analytics
model PlayerAnalytics {
  id              String    @id @default(cuid())
  userId          String    @unique
  user            User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  // Overall statistics
  totalTournaments Int      @default(0)
  totalGames      Int       @default(0)
  wins            Int       @default(0)
  losses          Int       @default(0)
  draws           Int       @default(0)
  
  // Performance metrics
  winRate         Float?    // wins / total games
  averageOpponentRating Float?
  performanceRating Float?  // calculated performance rating
  ratingTrend     Float?    // recent rating change trend
  
  // Playstyle analysis
  averageGameLength Float?  // in moves
  favoriteOpenings Json?    // most played openings
  strengthsByPhase Json?    // opening/middlegame/endgame strengths
  
  // Engagement metrics
  lastActive      DateTime?
  streakCurrent   Int       @default(0) // current win/loss streak
  streakBest      Int       @default(0) // best win streak
  
  // Recent performance (last 30 days)
  recentGames     Int       @default(0)
  recentWinRate   Float?
  recentRatingChange Float?
  
  // Calculated timestamps
  calculatedAt    DateTime  @default(now())
  updatedAt       DateTime  @updatedAt
  
  @@index([userId])
  @@index([winRate])
  @@index([lastActive])
}

// System usage analytics
model SystemAnalytics {
  id              String    @id @default(cuid())
  date            DateTime  // daily analytics
  
  // User metrics
  activeUsers     Int       @default(0)
  newUsers        Int       @default(0)
  returningUsers  Int       @default(0)
  peakConcurrentUsers Int   @default(0)
  
  // Tournament metrics
  tournamentsCreated Int    @default(0)
  tournamentsStarted Int    @default(0)
  tournamentsCompleted Int  @default(0)
  totalParticipants Int     @default(0)
  
  // Engagement metrics
  totalGamesPlayed Int      @default(0)
  avgSessionDuration Float? // in minutes
  pageViews       Int       @default(0)
  uniqueVisitors  Int       @default(0)
  
  // Performance metrics
  avgApiResponseTime Float? // in milliseconds
  errorRate       Float?    // percentage of requests that errored
  systemUptime    Float?    // percentage uptime
  
  // Geographic data
  topCountries    Json?     // country codes with user counts
  timezoneDistribution Json? // timezone usage patterns
  
  createdAt       DateTime  @default(now())
  
  @@unique([date])
  @@index([date])
}

// Custom reports configuration
model CustomReport {
  id              String      @id @default(cuid())
  name            String
  description     String?
  
  // Report configuration
  reportType      ReportType
  parameters      Json        // Report-specific parameters
  filters         Json?       // Data filtering criteria
  groupBy         String[]    // Grouping dimensions
  metrics         String[]    // Metrics to include
  
  // Scheduling
  isScheduled     Boolean     @default(false)
  schedulePattern String?     // Cron expression
  nextRun         DateTime?
  
  // Access control
  isPublic        Boolean     @default(false)
  allowedRoles    UserRole[]
  allowedUsers    String[]    // User IDs with access
  
  // Metadata
  createdBy       String
  creator         User        @relation(fields: [createdBy], references: [id])
  createdAt       DateTime    @default(now())
  updatedAt       DateTime    @updatedAt
  lastGenerated   DateTime?
  
  // Relations
  executions      ReportExecution[]
  
  @@index([createdBy])
  @@index([reportType])
  @@index([isScheduled, nextRun])
}

// Report execution history
model ReportExecution {
  id              String       @id @default(cuid())
  reportId        String
  report          CustomReport @relation(fields: [reportId], references: [id], onDelete: Cascade)
  
  // Execution details
  status          ExecutionStatus @default(PENDING)
  startedAt       DateTime     @default(now())
  completedAt     DateTime?
  duration        Int?         // in milliseconds
  
  // Result details
  resultUrl       String?      // URL to generated report file
  resultFormat    String?      // PDF, CSV, Excel, etc.
  recordCount     Int?         // Number of records in result
  fileSize        Int?         // File size in bytes
  
  // Error handling
  errorMessage    String?
  stackTrace      String?
  
  // Triggered by
  triggeredBy     String?      // User ID if manually triggered
  triggerType     TriggerType  @default(MANUAL)
  
  @@index([reportId, startedAt])
  @@index([status])
}

// Rating history for trend analysis
model RatingHistory {
  id              String    @id @default(cuid())
  userId          String
  user            User      @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  // Rating change details
  previousRating  Int?
  newRating       Int
  ratingChange    Int       // positive or negative
  ratingType      RatingType @default(LOCAL)
  
  // Context
  tournamentId    String?
  tournament      Tournament? @relation(fields: [tournamentId], references: [id])
  gameId          String?
  game            Game?     @relation(fields: [gameId], references: [id])
  
  // Change reason
  reason          String?   // "Tournament completion", "Manual adjustment", etc.
  
  createdAt       DateTime  @default(now())
  
  @@index([userId, createdAt])
  @@index([tournamentId])
  @@index([ratingType])
}

enum ReportType {
  TOURNAMENT_SUMMARY
  PLAYER_PERFORMANCE
  SYSTEM_USAGE
  FINANCIAL_SUMMARY
  CUSTOM_QUERY
  COMPARISON_ANALYSIS
  TREND_ANALYSIS
  LEADERBOARD
}

enum ExecutionStatus {
  PENDING
  RUNNING
  COMPLETED
  FAILED
  CANCELLED
}

enum TriggerType {
  MANUAL
  SCHEDULED
  API
  EVENT_TRIGGERED
}

enum RatingType {
  FIDE
  LOCAL
  TOURNAMENT_PERFORMANCE
}
```

#### Implementation

```typescript
// src/server/api/routers/analytics.ts
import { z } from 'zod';
import { createTRPCRouter, protectedProcedure, adminProcedure } from '../trpc';
import { TRPCError } from '@trpc/server';

export const analyticsRouter = createTRPCRouter({
  // Get tournament analytics
  getTournamentAnalytics: protectedProcedure
    .input(z.object({
      tournamentId: z.string().optional(),
      dateRange: z.object({
        from: z.date(),
        to: z.date(),
      }).optional(),
      includeComparison: z.boolean().default(false),
    }))
    .query(async ({ ctx, input }) => {
      const { tournamentId, dateRange, includeComparison } = input;

      if (tournamentId) {
        // Single tournament analytics
        return await getSingleTournamentAnalytics(ctx.db, tournamentId);
      }

      // Multiple tournaments analytics
      const where: any = {};
      if (dateRange) {
        where.tournament = {
          startDate: {
            gte: dateRange.from,
            lte: dateRange.to,
          },
        };
      }

      const analytics = await ctx.db.tournamentAnalytics.findMany({
        where,
        include: {
          tournament: {
            select: {
              id: true,
              name: true,
              startDate: true,
              status: true,
            },
          },
        },
        orderBy: { calculatedAt: 'desc' },
      });

      // Calculate aggregated metrics
      const aggregated = calculateAggregatedTournamentMetrics(analytics);

      return {
        analytics,
        aggregated,
        comparison: includeComparison ? await getComparisonData(ctx.db, dateRange) : null,
      };
    }),

  // Get player performance analytics
  getPlayerAnalytics: protectedProcedure
    .input(z.object({
      userId: z.string().optional(),
      includeRatingHistory: z.boolean().default(false),
      limit: z.number().default(50),
    }))
    .query(async ({ ctx, input }) => {
      const { userId, includeRatingHistory, limit } = input;

      // Check if user can view analytics
      const targetUserId = userId || ctx.session.user.id;
      if (targetUserId !== ctx.session.user.id && ctx.session.user.role !== 'ADMIN') {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'Cannot view other users analytics',
        });
      }

      const analytics = await ctx.db.playerAnalytics.findUnique({
        where: { userId: targetUserId },
        include: {
          user: {
            select: {
              name: true,
              email: true,
              profile: true,
            },
          },
        },
      });

      if (!analytics) {
        // Generate analytics if not exists
        await generatePlayerAnalytics(ctx.db, targetUserId);
        return await ctx.db.playerAnalytics.findUnique({
          where: { userId: targetUserId },
          include: {
            user: {
              select: {
                name: true,
                email: true,
                profile: true,
              },
            },
          },
        });
      }

      let ratingHistory = null;
      if (includeRatingHistory) {
        ratingHistory = await ctx.db.ratingHistory.findMany({
          where: { userId: targetUserId },
          orderBy: { createdAt: 'desc' },
          take: limit,
          include: {
            tournament: {
              select: { name: true },
            },
          },
        });
      }

      return {
        analytics,
        ratingHistory,
      };
    }),

  // Get system analytics dashboard
  getSystemAnalytics: adminProcedure
    .input(z.object({
      dateRange: z.object({
        from: z.date(),
        to: z.date(),
      }).optional(),
      granularity: z.enum(['hour', 'day', 'week', 'month']).default('day'),
    }))
    .query(async ({ ctx, input }) => {
      const { dateRange, granularity } = input;

      const defaultRange = {
        from: new Date(Date.now() - 30 * 24 * 60 * 60 * 1000), // 30 days ago
        to: new Date(),
      };

      const range = dateRange || defaultRange;

      const systemAnalytics = await ctx.db.systemAnalytics.findMany({
        where: {
          date: {
            gte: range.from,
            lte: range.to,
          },
        },
        orderBy: { date: 'asc' },
      });

      // Calculate trends and insights
      const trends = calculateSystemTrends(systemAnalytics);
      const insights = generateSystemInsights(systemAnalytics);

      return {
        analytics: systemAnalytics,
        trends,
        insights,
        summary: {
          totalUsers: trends.userGrowth.total,
          totalTournaments: trends.tournamentActivity.total,
          avgResponseTime: trends.performance.avgResponseTime,
          systemUptime: trends.performance.avgUptime,
        },
      };
    }),

  // Generate custom report
  generateCustomReport: protectedProcedure
    .input(z.object({
      reportId: z.string().optional(),
      reportType: z.enum(['TOURNAMENT_SUMMARY', 'PLAYER_PERFORMANCE', 'SYSTEM_USAGE', 'CUSTOM_QUERY']),
      parameters: z.record(z.any()),
      format: z.enum(['JSON', 'CSV', 'PDF', 'EXCEL']).default('JSON'),
    }))
    .mutation(async ({ ctx, input }) => {
      const { reportId, reportType, parameters, format } = input;

      // Create report execution record
      const execution = await ctx.db.reportExecution.create({
        data: {
          reportId: reportId || `temp-${Date.now()}`,
          status: 'RUNNING',
          triggeredBy: ctx.session.user.id,
          triggerType: 'MANUAL',
        },
      });

      try {
        // Generate report based on type
        const reportData = await generateReportData(ctx.db, reportType, parameters);
        
        // Format and export data
        const exportResult = await exportReportData(reportData, format);

        // Update execution status
        await ctx.db.reportExecution.update({
          where: { id: execution.id },
          data: {
            status: 'COMPLETED',
            completedAt: new Date(),
            duration: Date.now() - execution.startedAt.getTime(),
            resultUrl: exportResult.url,
            resultFormat: format,
            recordCount: reportData.length,
            fileSize: exportResult.fileSize,
          },
        });

        return {
          executionId: execution.id,
          downloadUrl: exportResult.url,
          recordCount: reportData.length,
        };
      } catch (error) {
        // Update execution with error
        await ctx.db.reportExecution.update({
          where: { id: execution.id },
          data: {
            status: 'FAILED',
            completedAt: new Date(),
            errorMessage: error instanceof Error ? error.message : 'Unknown error',
          },
        });

        throw new TRPCError({
          code: 'INTERNAL_SERVER_ERROR',
          message: 'Report generation failed',
        });
      }
    }),

  // Get leaderboard analytics
  getLeaderboardAnalytics: publicProcedure
    .input(z.object({
      type: z.enum(['rating', 'tournaments_won', 'recent_performance']).default('rating'),
      timeframe: z.enum(['all_time', 'year', 'month', 'week']).default('all_time'),
      limit: z.number().default(10),
    }))
    .query(async ({ ctx, input }) => {
      const { type, timeframe, limit } = input;

      let orderBy: any = { 'user.profile.localRating': 'desc' };
      let where: any = {};

      // Apply timeframe filter
      if (timeframe !== 'all_time') {
        const timeMap = {
          week: 7,
          month: 30,
          year: 365,
        };
        const days = timeMap[timeframe];
        const cutoff = new Date(Date.now() - days * 24 * 60 * 60 * 1000);
        where.updatedAt = { gte: cutoff };
      }

      switch (type) {
        case 'rating':
          orderBy = { 'user.profile.localRating': 'desc' };
          break;
        case 'tournaments_won':
          orderBy = { totalTournaments: 'desc' }; // This should be wins specifically
          break;
        case 'recent_performance':
          orderBy = { recentWinRate: 'desc' };
          where.recentGames = { gte: 5 }; // Only include players with recent activity
          break;
      }

      const leaderboard = await ctx.db.playerAnalytics.findMany({
        where,
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
                  localRating: true,
                  fideRating: true,
                  country: true,
                },
              },
            },
          },
        },
      });

      return leaderboard.map((entry, index) => ({
        rank: index + 1,
        userId: entry.userId,
        name: entry.user.name || entry.user.email,
        rating: entry.user.profile?.localRating || 0,
        fideRating: entry.user.profile?.fideRating,
        country: entry.user.profile?.country,
        totalTournaments: entry.totalTournaments,
        winRate: entry.winRate,
        recentPerformance: entry.recentWinRate,
        totalGames: entry.totalGames,
      }));
    }),
});

// Helper functions for analytics calculations
async function getSingleTournamentAnalytics(db: any, tournamentId: string) {
  const analytics = await db.tournamentAnalytics.findUnique({
    where: { tournamentId },
    include: {
      tournament: {
        include: {
          participants: {
            include: {
              user: {
                select: { name: true, profile: true },
              },
            },
          },
          rounds: {
            include: {
              games: {
                include: {
                  white: { select: { name: true } },
                  black: { select: { name: true } },
                },
              },
            },
          },
        },
      },
    },
  });

  if (!analytics) {
    // Generate analytics if not exists
    await generateTournamentAnalytics(db, tournamentId);
    return await getSingleTournamentAnalytics(db, tournamentId);
  }

  return analytics;
}

async function generateTournamentAnalytics(db: any, tournamentId: string) {
  const tournament = await db.tournament.findUnique({
    where: { id: tournamentId },
    include: {
      participants: {
        include: { user: { include: { profile: true } } },
      },
      rounds: {
        include: { games: true },
      },
    },
  });

  if (!tournament) return;

  const totalParticipants = tournament.participants.length;
  const allGames = tournament.rounds.flatMap((r: any) => r.games);
  const completedGames = allGames.filter((g: any) => g.result);
  
  const ratings = tournament.participants
    .map((p: any) => p.user.profile?.localRating)
    .filter(Boolean);
  
  const avgRating = ratings.length > 0 ? ratings.reduce((a, b) => a + b, 0) / ratings.length : null;
  const ratingSpread = ratings.length > 0 ? Math.max(...ratings) - Math.min(...ratings) : null;
  
  const drawCount = completedGames.filter((g: any) => g.result === 'DRAW').length;
  const drawPercentage = completedGames.length > 0 ? (drawCount / completedGames.length) * 100 : null;

  await db.tournamentAnalytics.upsert({
    where: { tournamentId },
    create: {
      tournamentId,
      totalParticipants,
      totalGames: allGames.length,
      completedGames: completedGames.length,
      avgRatingParticipants: avgRating,
      ratingSpread,
      drawPercentage,
    },
    update: {
      totalParticipants,
      totalGames: allGames.length,
      completedGames: completedGames.length,
      avgRatingParticipants: avgRating,
      ratingSpread,
      drawPercentage,
      updatedAt: new Date(),
    },
  });
}

async function generatePlayerAnalytics(db: any, userId: string) {
  // Get all tournament participations
  const participations = await db.tournamentParticipant.findMany({
    where: { userId },
    include: {
      tournament: true,
    },
  });

  // Get all games
  const games = await db.game.findMany({
    where: {
      OR: [
        { whiteId: userId },
        { blackId: userId },
      ],
      result: { not: null },
    },
    include: {
      white: { include: { profile: true } },
      black: { include: { profile: true } },
    },
  });

  const totalTournaments = participations.length;
  const totalGames = games.length;
  
  let wins = 0;
  let losses = 0;
  let draws = 0;
  let opponentRatings: number[] = [];

  games.forEach(game => {
    const isWhite = game.whiteId === userId;
    const opponent = isWhite ? game.black : game.white;
    const opponentRating = opponent.profile?.localRating;
    
    if (opponentRating) {
      opponentRatings.push(opponentRating);
    }

    switch (game.result) {
      case 'WHITE_WINS':
        isWhite ? wins++ : losses++;
        break;
      case 'BLACK_WINS':
        isWhite ? losses++ : wins++;
        break;
      case 'DRAW':
        draws++;
        break;
    }
  });

  const winRate = totalGames > 0 ? (wins / totalGames) * 100 : null;
  const avgOpponentRating = opponentRatings.length > 0 
    ? opponentRatings.reduce((a, b) => a + b, 0) / opponentRatings.length 
    : null;

  // Recent performance (last 30 days)
  const thirtyDaysAgo = new Date(Date.now() - 30 * 24 * 60 * 60 * 1000);
  const recentGames = games.filter(g => g.endTime && g.endTime > thirtyDaysAgo);
  const recentWins = recentGames.filter(g => {
    const isWhite = g.whiteId === userId;
    return (isWhite && g.result === 'WHITE_WINS') || (!isWhite && g.result === 'BLACK_WINS');
  }).length;
  const recentWinRate = recentGames.length > 0 ? (recentWins / recentGames.length) * 100 : null;

  await db.playerAnalytics.upsert({
    where: { userId },
    create: {
      userId,
      totalTournaments,
      totalGames,
      wins,
      losses,
      draws,
      winRate,
      averageOpponentRating: avgOpponentRating,
      recentGames: recentGames.length,
      recentWinRate,
      lastActive: games.length > 0 ? games[games.length - 1].endTime : null,
    },
    update: {
      totalTournaments,
      totalGames,
      wins,
      losses,
      draws,
      winRate,
      averageOpponentRating: avgOpponentRating,
      recentGames: recentGames.length,
      recentWinRate,
      lastActive: games.length > 0 ? games[games.length - 1].endTime : null,
      updatedAt: new Date(),
    },
  });
}
```

#### Acceptance Criteria
- [ ] Analytics data model created and functional
- [ ] Tournament analytics calculation accurate
- [ ] Player performance metrics generated correctly
- [ ] System analytics tracking operational
- [ ] Data aggregation pipeline working

### Phase 2: Report Generation & Visualization (2 days)

#### Tasks
1. **Create Report Generation System**
   - Pre-built report templates
   - Custom report builder interface
   - Scheduled report execution

2. **Implement Data Visualization Components**
   - Interactive charts and graphs
   - Trend analysis visualizations
   - Performance comparison tools

#### Implementation

```typescript
// src/app/_components/analytics/chart-components.tsx
'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { ResponsiveContainer, LineChart, Line, XAxis, YAxis, CartesianGrid, Tooltip, Legend, BarChart, Bar, PieChart, Pie, Cell, Area, AreaChart } from 'recharts';

interface ChartDataPoint {
  name: string;
  value: number;
  [key: string]: any;
}

interface TrendChartProps {
  data: ChartDataPoint[];
  dataKey: string;
  title: string;
  color?: string;
}

export function TrendChart({ data, dataKey, title, color = '#8884d8' }: TrendChartProps) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{title}</CardTitle>
      </CardHeader>
      <CardContent>
        <ResponsiveContainer width="100%" height={300}>
          <LineChart data={data}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="name" />
            <YAxis />
            <Tooltip />
            <Line type="monotone" dataKey={dataKey} stroke={color} strokeWidth={2} />
          </LineChart>
        </ResponsiveContainer>
      </CardContent>
    </Card>
  );
}

interface PerformanceComparisonProps {
  data: ChartDataPoint[];
  title: string;
}

export function PerformanceComparison({ data, title }: PerformanceComparisonProps) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{title}</CardTitle>
      </CardHeader>
      <CardContent>
        <ResponsiveContainer width="100%" height={300}>
          <BarChart data={data}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="name" />
            <YAxis />
            <Tooltip />
            <Legend />
            <Bar dataKey="wins" fill="#22c55e" name="Wins" />
            <Bar dataKey="losses" fill="#ef4444" name="Losses" />
            <Bar dataKey="draws" fill="#6b7280" name="Draws" />
          </BarChart>
        </ResponsiveContainer>
      </CardContent>
    </Card>
  );
}

interface RatingDistributionProps {
  data: { range: string; count: number; percentage: number }[];
  title: string;
}

export function RatingDistribution({ data, title }: RatingDistributionProps) {
  const colors = ['#0088fe', '#00c49f', '#ffbb28', '#ff8042', '#8884d8'];

  return (
    <Card>
      <CardHeader>
        <CardTitle>{title}</CardTitle>
      </CardHeader>
      <CardContent>
        <ResponsiveContainer width="100%" height={300}>
          <PieChart>
            <Pie
              data={data}
              cx="50%"
              cy="50%"
              outerRadius={80}
              fill="#8884d8"
              dataKey="count"
              label={({ range, percentage }) => `${range} (${percentage}%)`}
            >
              {data.map((entry, index) => (
                <Cell key={`cell-${index}`} fill={colors[index % colors.length]} />
              ))}
            </Pie>
            <Tooltip />
          </PieChart>
        </ResponsiveContainer>
      </CardContent>
    </Card>
  );
}

interface TournamentActivityProps {
  data: ChartDataPoint[];
  title: string;
}

export function TournamentActivity({ data, title }: TournamentActivityProps) {
  return (
    <Card>
      <CardHeader>
        <CardTitle>{title}</CardTitle>
      </CardHeader>
      <CardContent>
        <ResponsiveContainer width="100%" height={300}>
          <AreaChart data={data}>
            <CartesianGrid strokeDasharray="3 3" />
            <XAxis dataKey="name" />
            <YAxis />
            <Tooltip />
            <Legend />
            <Area type="monotone" dataKey="created" stackId="1" stroke="#8884d8" fill="#8884d8" name="Created" />
            <Area type="monotone" dataKey="started" stackId="1" stroke="#82ca9d" fill="#82ca9d" name="Started" />
            <Area type="monotone" dataKey="completed" stackId="1" stroke="#ffc658" fill="#ffc658" name="Completed" />
          </AreaChart>
        </ResponsiveContainer>
      </CardContent>
    </Card>
  );
}

// src/app/_components/analytics/analytics-dashboard.tsx
'use client';

import { useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { DatePickerWithRange } from '@/components/ui/date-range-picker';
import { Tabs, TabsContent, TabsList, TabsTrigger } from '@/components/ui/tabs';
import { Badge } from '@/components/ui/badge';
import { TrendingUp, TrendingDown, Users, Trophy, BarChart3, Download } from 'lucide-react';
import { api } from '@/trpc/react';
import { TrendChart, PerformanceComparison, RatingDistribution, TournamentActivity } from './chart-components';
import { addDays } from 'date-fns';

export function AnalyticsDashboard() {
  const [dateRange, setDateRange] = useState({
    from: addDays(new Date(), -30),
    to: new Date(),
  });
  const [timeframe, setTimeframe] = useState<'day' | 'week' | 'month'>('day');

  // Get system analytics
  const { data: systemAnalytics, isLoading } = api.analytics.getSystemAnalytics.useQuery({
    dateRange,
    granularity: timeframe,
  });

  // Get tournament analytics
  const { data: tournamentAnalytics } = api.analytics.getTournamentAnalytics.useQuery({
    dateRange,
    includeComparison: true,
  });

  // Get leaderboard data
  const { data: leaderboard } = api.analytics.getLeaderboardAnalytics.useQuery({
    type: 'rating',
    timeframe: 'month',
    limit: 10,
  });

  const handleExport = (format: 'CSV' | 'PDF' | 'EXCEL') => {
    // Implement export functionality
    console.log('Exporting data in format:', format);
  };

  if (isLoading) {
    return (
      <div className="space-y-6">
        <div className="grid grid-cols-1 md:grid-cols-4 gap-6">
          {[...Array(4)].map((_, i) => (
            <Card key={i}>
              <CardContent className="p-6">
                <div className="animate-pulse space-y-2">
                  <div className="h-4 bg-muted rounded w-3/4"></div>
                  <div className="h-8 bg-muted rounded w-1/2"></div>
                </div>
              </CardContent>
            </Card>
          ))}
        </div>
      </div>
    );
  }

  const formatTrendData = (analytics: any[]) => {
    return analytics.map(item => ({
      name: new Date(item.date).toLocaleDateString(),
      users: item.activeUsers,
      tournaments: item.tournamentsStarted,
      games: item.totalGamesPlayed,
    }));
  };

  return (
    <div className="space-y-6">
      {/* Header */}
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Analytics Dashboard</h1>
        <div className="flex items-center gap-4">
          <DatePickerWithRange
            value={dateRange}
            onChange={setDateRange}
          />
          <Select value={timeframe} onValueChange={(value: any) => setTimeframe(value)}>
            <SelectTrigger className="w-32">
              <SelectValue />
            </SelectTrigger>
            <SelectContent>
              <SelectItem value="day">Daily</SelectItem>
              <SelectItem value="week">Weekly</SelectItem>
              <SelectItem value="month">Monthly</SelectItem>
            </SelectContent>
          </Select>
          <Button variant="outline" onClick={() => handleExport('CSV')}>
            <Download className="w-4 h-4 mr-2" />
            Export
          </Button>
        </div>
      </div>

      {/* Summary Cards */}
      {systemAnalytics && (
        <div className="grid grid-cols-1 md:grid-cols-4 gap-6">
          <Card>
            <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
              <CardTitle className="text-sm font-medium">Total Users</CardTitle>
              <Users className="h-4 w-4 text-muted-foreground" />
            </CardHeader>
            <CardContent>
              <div className="text-2xl font-bold">{systemAnalytics.summary.totalUsers}</div>
              <div className="flex items-center gap-1 text-xs text-muted-foreground">
                <TrendingUp className="h-3 w-3 text-green-500" />
                <span>+{systemAnalytics.trends.userGrowth.percentageChange}% from last period</span>
              </div>
            </CardContent>
          </Card>

          <Card>
            <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
              <CardTitle className="text-sm font-medium">Active Tournaments</CardTitle>
              <Trophy className="h-4 w-4 text-muted-foreground" />
            </CardHeader>
            <CardContent>
              <div className="text-2xl font-bold">{systemAnalytics.summary.totalTournaments}</div>
              <div className="flex items-center gap-1 text-xs text-muted-foreground">
                <TrendingUp className="h-3 w-3 text-green-500" />
                <span>+{systemAnalytics.trends.tournamentActivity.percentageChange}% from last period</span>
              </div>
            </CardContent>
          </Card>

          <Card>
            <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
              <CardTitle className="text-sm font-medium">Avg Response Time</CardTitle>
              <BarChart3 className="h-4 w-4 text-muted-foreground" />
            </CardHeader>
            <CardContent>
              <div className="text-2xl font-bold">{systemAnalytics.summary.avgResponseTime}ms</div>
              <div className="flex items-center gap-1 text-xs text-muted-foreground">
                <TrendingDown className="h-3 w-3 text-green-500" />
                <span>-{Math.abs(systemAnalytics.trends.performance.responseTimeChange)}% improved</span>
              </div>
            </CardContent>
          </Card>

          <Card>
            <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
              <CardTitle className="text-sm font-medium">System Uptime</CardTitle>
              <div className="h-2 w-2 rounded-full bg-green-500"></div>
            </CardHeader>
            <CardContent>
              <div className="text-2xl font-bold">{systemAnalytics.summary.systemUptime}%</div>
              <div className="text-xs text-muted-foreground">
                Last 30 days
              </div>
            </CardContent>
          </Card>
        </div>
      )}

      {/* Charts and Analytics */}
      <Tabs defaultValue="overview" className="space-y-6">
        <TabsList>
          <TabsTrigger value="overview">Overview</TabsTrigger>
          <TabsTrigger value="tournaments">Tournaments</TabsTrigger>
          <TabsTrigger value="players">Players</TabsTrigger>
          <TabsTrigger value="performance">Performance</TabsTrigger>
        </TabsList>

        <TabsContent value="overview" className="space-y-6">
          <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
            {systemAnalytics && (
              <>
                <TrendChart
                  data={formatTrendData(systemAnalytics.analytics)}
                  dataKey="users"
                  title="User Activity Trend"
                  color="#8884d8"
                />
                <TournamentActivity
                  data={formatTrendData(systemAnalytics.analytics)}
                  title="Tournament Activity"
                />
              </>
            )}
          </div>

          {/* Top Players */}
          {leaderboard && (
            <Card>
              <CardHeader>
                <CardTitle>Top Rated Players</CardTitle>
              </CardHeader>
              <CardContent>
                <div className="space-y-3">
                  {leaderboard.slice(0, 5).map((player, index) => (
                    <div key={player.userId} className="flex items-center justify-between">
                      <div className="flex items-center gap-3">
                        <Badge variant="outline">#{player.rank}</Badge>
                        <div>
                          <div className="font-medium">{player.name}</div>
                          <div className="text-sm text-muted-foreground">
                            {player.totalTournaments} tournaments • {player.winRate?.toFixed(1)}% win rate
                          </div>
                        </div>
                      </div>
                      <div className="text-right">
                        <div className="font-bold">{player.rating}</div>
                        <div className="text-sm text-muted-foreground">Rating</div>
                      </div>
                    </div>
                  ))}
                </div>
              </CardContent>
            </Card>
          )}
        </TabsContent>

        <TabsContent value="tournaments" className="space-y-6">
          {tournamentAnalytics && (
            <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
              <Card>
                <CardHeader>
                  <CardTitle>Tournament Completion Rate</CardTitle>
                </CardHeader>
                <CardContent>
                  <div className="space-y-4">
                    {tournamentAnalytics.analytics.slice(0, 5).map(tournament => (
                      <div key={tournament.id} className="flex items-center justify-between">
                        <div>
                          <div className="font-medium">{tournament.tournament.name}</div>
                          <div className="text-sm text-muted-foreground">
                            {tournament.totalParticipants} participants
                          </div>
                        </div>
                        <div className="text-right">
                          <div className="font-bold">
                            {tournament.completedGames}/{tournament.totalGames}
                          </div>
                          <div className="text-sm text-muted-foreground">
                            {((tournament.completedGames / tournament.totalGames) * 100).toFixed(1)}%
                          </div>
                        </div>
                      </div>
                    ))}
                  </div>
                </CardContent>
              </Card>

              <Card>
                <CardHeader>
                  <CardTitle>Average Tournament Metrics</CardTitle>
                </CardHeader>
                <CardContent>
                  <div className="space-y-4">
                    <div className="flex justify-between">
                      <span>Avg Participants</span>
                      <span className="font-medium">
                        {tournamentAnalytics.aggregated.avgParticipants.toFixed(1)}
                      </span>
                    </div>
                    <div className="flex justify-between">
                      <span>Avg Rating</span>
                      <span className="font-medium">
                        {tournamentAnalytics.aggregated.avgRating.toFixed(0)}
                      </span>
                    </div>
                    <div className="flex justify-between">
                      <span>Draw Rate</span>
                      <span className="font-medium">
                        {tournamentAnalytics.aggregated.avgDrawRate.toFixed(1)}%
                      </span>
                    </div>
                    <div className="flex justify-between">
                      <span>Completion Rate</span>
                      <span className="font-medium">
                        {tournamentAnalytics.aggregated.avgCompletionRate.toFixed(1)}%
                      </span>
                    </div>
                  </div>
                </CardContent>
              </Card>
            </div>
          )}
        </TabsContent>

        <TabsContent value="players" className="space-y-6">
          <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
            {/* Rating distribution would go here */}
            <Card>
              <CardHeader>
                <CardTitle>Player Rating Distribution</CardTitle>
              </CardHeader>
              <CardContent>
                <div className="text-center text-muted-foreground">
                  Rating distribution chart will be implemented here
                </div>
              </CardContent>
            </Card>

            <Card>
              <CardHeader>
                <CardTitle>Player Activity</CardTitle>
              </CardHeader>
              <CardContent>
                <div className="text-center text-muted-foreground">
                  Player activity metrics will be implemented here
                </div>
              </CardContent>
            </Card>
          </div>
        </TabsContent>

        <TabsContent value="performance" className="space-y-6">
          {systemAnalytics && (
            <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
              <TrendChart
                data={systemAnalytics.analytics.map(item => ({
                  name: new Date(item.date).toLocaleDateString(),
                  value: item.avgApiResponseTime || 0,
                }))}
                dataKey="value"
                title="API Response Time Trend"
                color="#ef4444"
              />

              <TrendChart
                data={systemAnalytics.analytics.map(item => ({
                  name: new Date(item.date).toLocaleDateString(),
                  value: item.systemUptime || 0,
                }))}
                dataKey="value"
                title="System Uptime"
                color="#22c55e"
              />
            </div>
          )}
        </TabsContent>
      </Tabs>
    </div>
  );
}
```

#### Acceptance Criteria
- [ ] Report generation system functional
- [ ] Data visualization components working
- [ ] Interactive charts displaying correctly
- [ ] Export functionality operational
- [ ] Custom report builder interface created

### Phase 3: Advanced Analytics & Insights (1 day)

#### Tasks
1. **Implement Advanced Analytics Features**
   - Predictive analytics for player performance
   - Tournament outcome predictions
   - Rating trend analysis

2. **Create Insight Generation System**
   - Automated insight detection
   - Performance recommendations
   - Anomaly detection alerts

#### Acceptance Criteria
- [ ] Advanced analytics algorithms functional
- [ ] Insight generation providing valuable information
- [ ] Predictive models working accurately
- [ ] Anomaly detection alerting appropriately
- [ ] Performance recommendations relevant

## Risk Assessment

### High Risk
- **Performance complexity**: Large dataset analytics queries
- **Data accuracy**: Ensuring calculation correctness across all metrics
- **Export functionality**: Reliable file generation and delivery

### Medium Risk
- **Visualization performance**: Chart rendering with large datasets
- **Report scheduling**: Automated execution reliability

### Low Risk
- **UI components**: Standard dashboard and chart patterns
- **Basic analytics**: Simple aggregation calculations

## Success Metrics

- Report generation time: <30 seconds for standard reports
- Analytics query performance: <5 seconds for complex calculations
- Data accuracy: 99.9% correctness in all metrics
- Export success rate: >99% for all formats
- User engagement: 80% of admins use analytics monthly

## Dependencies for Next Phase

This analytics system completes the Chess Community Platform by providing:
- Comprehensive tournament and player insights
- Data-driven decision making capabilities
- Platform performance monitoring and optimization
- Historical analysis and trend identification
- Export capabilities for external analysis
