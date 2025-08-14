# Real-time System Implementation Plan

## Feature Overview

Implement comprehensive real-time update system using WebSocket integration to provide live tournament data, game status updates, and interactive notifications across the Chess Community Platform.

## Objectives

- Real-time tournament scoreboard updates
- Live game status and result notifications
- Instant participant registration updates
- Real-time standings calculations
- Live administrative notifications
- WebSocket-based communication infrastructure

## Technical Requirements

### Dependencies
- Dashboard system (for admin real-time monitoring)
- Standings system (for live ranking updates)
- Scoreboard system (for game status updates)
- Authentication system (for user-specific subscriptions)
- Tournament and game management systems

### Constraints
- Low latency requirements (<1 second for updates)
- Scalable WebSocket connection management
- Efficient data serialization and transmission
- Connection persistence and reconnection handling
- Cross-browser WebSocket compatibility

## Implementation Phases

### Phase 1: WebSocket Infrastructure & tRPC Subscriptions (2 days)

#### Tasks
1. **Setup WebSocket Server Integration**
   - Configure Next.js for WebSocket support
   - Implement connection management and cleanup
   - Setup authentication for WebSocket connections

2. **Create tRPC Subscription System**
   - Implement subscription procedures for real-time data
   - Setup event broadcasting infrastructure
   - Create type-safe subscription contracts

#### Database Changes

```prisma
// Real-time connection tracking
model WebSocketConnection {
  id           String    @id @default(cuid())
  userId       String?   // Null for anonymous viewers
  sessionId    String    @unique
  isActive     Boolean   @default(true)
  
  // Connection metadata
  ipAddress    String?
  userAgent    String?
  connectedAt  DateTime  @default(now())
  lastPing     DateTime  @default(now())
  
  // Subscription preferences
  subscriptions Json     @default("[]") // Array of subscription types
  
  // Relations
  user         User?     @relation(fields: [userId], references: [id], onDelete: SetNull)
  
  @@index([userId, isActive])
  @@index([sessionId])
}

// Real-time event log for debugging and analytics
model RealtimeEvent {
  id          String     @id @default(cuid())
  eventType   EventType
  entityType  String     // Tournament, Game, User, etc.
  entityId    String     // ID of the affected entity
  
  // Event data
  payload     Json       // Event-specific data
  targetUsers String[]   // User IDs to notify (empty = broadcast)
  
  // Delivery tracking
  isProcessed Boolean    @default(false)
  deliveredTo String[]   // User IDs who received the event
  failedTo    String[]   // User IDs who failed to receive
  
  // Metadata
  createdBy   String?    // User who triggered the event
  createdAt   DateTime   @default(now())
  processedAt DateTime?
  
  @@index([eventType, isProcessed])
  @@index([entityType, entityId])
  @@index([createdAt])
}

// User notification preferences
model NotificationPreference {
  id             String  @id @default(cuid())
  userId         String  @unique
  
  // Real-time notification settings
  gameResults    Boolean @default(true)
  tournamentUpdates Boolean @default(true)
  standingsChanges Boolean @default(true)
  pairingUpdates Boolean @default(true)
  systemAlerts   Boolean @default(true)
  
  // Delivery preferences
  realTimeNotifications Boolean @default(true)
  emailNotifications    Boolean @default(true)
  pushNotifications     Boolean @default(false)
  
  // Relations
  user           User    @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@index([userId])
}

enum EventType {
  GAME_STARTED
  GAME_RESULT_SUBMITTED
  GAME_COMPLETED
  ROUND_STARTED
  ROUND_COMPLETED
  TOURNAMENT_STARTED
  TOURNAMENT_COMPLETED
  PARTICIPANT_REGISTERED
  PARTICIPANT_WITHDREW
  STANDINGS_UPDATED
  PAIRING_GENERATED
  SYSTEM_ANNOUNCEMENT
  USER_JOINED
  USER_LEFT
}
```

#### Implementation

```typescript
// src/server/api/ws-server.ts
import { WebSocketServer } from 'ws';
import { parse } from 'url';
import { verifyJWT } from './auth/jwt-utils';
import { db } from './db';

interface WebSocketSession {
  id: string;
  userId?: string;
  ws: WebSocket;
  subscriptions: Set<string>;
  lastPing: Date;
}

export class ChessWebSocketServer {
  private wss: WebSocketServer;
  private sessions = new Map<string, WebSocketSession>();
  private pingInterval: NodeJS.Timeout;

  constructor(port: number = 3001) {
    this.wss = new WebSocketServer({ port });
    this.setupEventHandlers();
    this.startPingInterval();
  }

  private setupEventHandlers() {
    this.wss.on('connection', async (ws, req) => {
      const { query } = parse(req.url || '', true);
      const token = query.token as string;
      
      let userId: string | undefined;
      
      // Authenticate user if token provided
      if (token) {
        try {
          const payload = await verifyJWT(token);
          userId = payload.sub;
        } catch (error) {
          console.error('WebSocket authentication failed:', error);
        }
      }

      // Create session
      const sessionId = this.generateSessionId();
      const session: WebSocketSession = {
        id: sessionId,
        userId,
        ws,
        subscriptions: new Set(),
        lastPing: new Date(),
      };

      this.sessions.set(sessionId, session);

      // Store connection in database
      await this.storeConnection(sessionId, userId, req);

      // Setup message handlers
      ws.on('message', (data) => {
        this.handleMessage(sessionId, data);
      });

      ws.on('close', () => {
        this.handleDisconnection(sessionId);
      });

      ws.on('pong', () => {
        this.handlePong(sessionId);
      });

      // Send connection confirmation
      this.sendToSession(sessionId, {
        type: 'connected',
        sessionId,
        timestamp: new Date().toISOString(),
      });
    });
  }

  private async handleMessage(sessionId: string, data: Buffer) {
    const session = this.sessions.get(sessionId);
    if (!session) return;

    try {
      const message = JSON.parse(data.toString());
      
      switch (message.type) {
        case 'subscribe':
          await this.handleSubscription(sessionId, message.channel, true);
          break;
          
        case 'unsubscribe':
          await this.handleSubscription(sessionId, message.channel, false);
          break;
          
        case 'ping':
          this.sendToSession(sessionId, { type: 'pong' });
          break;
          
        default:
          console.warn('Unknown message type:', message.type);
      }
    } catch (error) {
      console.error('Error handling WebSocket message:', error);
    }
  }

  private async handleSubscription(sessionId: string, channel: string, subscribe: boolean) {
    const session = this.sessions.get(sessionId);
    if (!session) return;

    if (subscribe) {
      session.subscriptions.add(channel);
    } else {
      session.subscriptions.delete(channel);
    }

    // Update database
    await db.webSocketConnection.update({
      where: { sessionId },
      data: {
        subscriptions: Array.from(session.subscriptions),
      },
    });

    this.sendToSession(sessionId, {
      type: subscribe ? 'subscribed' : 'unsubscribed',
      channel,
    });
  }

  public broadcastToChannel(channel: string, data: any) {
    const message = JSON.stringify({
      type: 'broadcast',
      channel,
      data,
      timestamp: new Date().toISOString(),
    });

    for (const session of this.sessions.values()) {
      if (session.subscriptions.has(channel)) {
        if (session.ws.readyState === WebSocket.OPEN) {
          session.ws.send(message);
        }
      }
    }
  }

  public sendToUser(userId: string, data: any) {
    const message = JSON.stringify({
      type: 'user_message',
      data,
      timestamp: new Date().toISOString(),
    });

    for (const session of this.sessions.values()) {
      if (session.userId === userId && session.ws.readyState === WebSocket.OPEN) {
        session.ws.send(message);
      }
    }
  }

  private sendToSession(sessionId: string, data: any) {
    const session = this.sessions.get(sessionId);
    if (session && session.ws.readyState === WebSocket.OPEN) {
      session.ws.send(JSON.stringify(data));
    }
  }

  private startPingInterval() {
    this.pingInterval = setInterval(() => {
      const now = new Date();
      
      for (const [sessionId, session] of this.sessions) {
        // Remove stale connections
        if (now.getTime() - session.lastPing.getTime() > 60000) {
          this.handleDisconnection(sessionId);
          continue;
        }

        // Send ping
        if (session.ws.readyState === WebSocket.OPEN) {
          session.ws.ping();
        }
      }
    }, 30000); // Ping every 30 seconds
  }

  private async handleDisconnection(sessionId: string) {
    const session = this.sessions.get(sessionId);
    if (session) {
      this.sessions.delete(sessionId);
      
      // Update database
      await db.webSocketConnection.update({
        where: { sessionId },
        data: { isActive: false },
      }).catch(console.error);
    }
  }

  private generateSessionId(): string {
    return Math.random().toString(36).substring(2) + Date.now().toString(36);
  }

  private async storeConnection(sessionId: string, userId?: string, req?: any) {
    try {
      await db.webSocketConnection.create({
        data: {
          sessionId,
          userId,
          ipAddress: req?.socket?.remoteAddress,
          userAgent: req?.headers?.['user-agent'],
        },
      });
    } catch (error) {
      console.error('Failed to store WebSocket connection:', error);
    }
  }
}

// Start WebSocket server
export const wsServer = new ChessWebSocketServer();

// src/server/api/routers/realtime.ts
import { z } from 'zod';
import { createTRPCRouter, protectedProcedure, publicProcedure } from '../trpc';
import { observable } from '@trpc/server/observable';
import { EventEmitter } from 'events';

// Global event emitter for real-time events
export const realtimeEventEmitter = new EventEmitter();

export const realtimeRouter = createTRPCRouter({
  // Subscribe to tournament updates
  onTournamentUpdate: publicProcedure
    .input(z.object({ tournamentId: z.string() }))
    .subscription(({ input }) => {
      return observable<{
        type: string;
        tournamentId: string;
        data: any;
      }>((emit) => {
        const handler = (data: any) => {
          if (data.tournamentId === input.tournamentId) {
            emit.next(data);
          }
        };

        realtimeEventEmitter.on('tournament_update', handler);

        return () => {
          realtimeEventEmitter.off('tournament_update', handler);
        };
      });
    }),

  // Subscribe to game updates
  onGameUpdate: publicProcedure
    .input(z.object({ gameId: z.string().optional(), tournamentId: z.string().optional() }))
    .subscription(({ input }) => {
      return observable<{
        type: string;
        gameId?: string;
        tournamentId?: string;
        data: any;
      }>((emit) => {
        const handler = (data: any) => {
          const matchesGame = !input.gameId || data.gameId === input.gameId;
          const matchesTournament = !input.tournamentId || data.tournamentId === input.tournamentId;
          
          if (matchesGame && matchesTournament) {
            emit.next(data);
          }
        };

        realtimeEventEmitter.on('game_update', handler);

        return () => {
          realtimeEventEmitter.off('game_update', handler);
        };
      });
    }),

  // Subscribe to standings updates
  onStandingsUpdate: publicProcedure
    .input(z.object({ tournamentId: z.string() }))
    .subscription(({ input }) => {
      return observable<{
        type: string;
        tournamentId: string;
        standings: any[];
      }>((emit) => {
        const handler = (data: any) => {
          if (data.tournamentId === input.tournamentId) {
            emit.next(data);
          }
        };

        realtimeEventEmitter.on('standings_update', handler);

        return () => {
          realtimeEventEmitter.off('standings_update', handler);
        };
      });
    }),

  // Get user's notification preferences
  getNotificationPreferences: protectedProcedure
    .query(async ({ ctx }) => {
      let preferences = await ctx.db.notificationPreference.findUnique({
        where: { userId: ctx.session.user.id },
      });

      if (!preferences) {
        preferences = await ctx.db.notificationPreference.create({
          data: { userId: ctx.session.user.id },
        });
      }

      return preferences;
    }),

  // Update notification preferences
  updateNotificationPreferences: protectedProcedure
    .input(z.object({
      gameResults: z.boolean().optional(),
      tournamentUpdates: z.boolean().optional(),
      standingsChanges: z.boolean().optional(),
      pairingUpdates: z.boolean().optional(),
      systemAlerts: z.boolean().optional(),
      realTimeNotifications: z.boolean().optional(),
      emailNotifications: z.boolean().optional(),
      pushNotifications: z.boolean().optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      return await ctx.db.notificationPreference.upsert({
        where: { userId: ctx.session.user.id },
        create: {
          userId: ctx.session.user.id,
          ...input,
        },
        update: input,
      });
    }),

  // Get active WebSocket connections (admin only)
  getActiveConnections: protectedProcedure
    .query(async ({ ctx }) => {
      if (ctx.session.user.role !== 'ADMIN') {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'Admin access required',
        });
      }

      const connections = await ctx.db.webSocketConnection.findMany({
        where: { isActive: true },
        include: {
          user: {
            select: { name: true, email: true },
          },
        },
        orderBy: { connectedAt: 'desc' },
      });

      return connections;
    }),
});

// src/lib/realtime-events.ts
import { realtimeEventEmitter } from '@/server/api/routers/realtime';
import { wsServer } from '@/server/api/ws-server';

export class RealtimeEventService {
  static emitTournamentUpdate(tournamentId: string, updateType: string, data: any) {
    const eventData = {
      type: updateType,
      tournamentId,
      data,
      timestamp: new Date().toISOString(),
    };

    // Emit to tRPC subscribers
    realtimeEventEmitter.emit('tournament_update', eventData);
    
    // Broadcast to WebSocket subscribers
    wsServer.broadcastToChannel(`tournament:${tournamentId}`, eventData);
  }

  static emitGameUpdate(gameId: string, tournamentId: string, updateType: string, data: any) {
    const eventData = {
      type: updateType,
      gameId,
      tournamentId,
      data,
      timestamp: new Date().toISOString(),
    };

    // Emit to tRPC subscribers
    realtimeEventEmitter.emit('game_update', eventData);
    
    // Broadcast to WebSocket subscribers
    wsServer.broadcastToChannel(`game:${gameId}`, eventData);
    wsServer.broadcastToChannel(`tournament:${tournamentId}:games`, eventData);
  }

  static emitStandingsUpdate(tournamentId: string, standings: any[]) {
    const eventData = {
      type: 'standings_updated',
      tournamentId,
      standings,
      timestamp: new Date().toISOString(),
    };

    // Emit to tRPC subscribers
    realtimeEventEmitter.emit('standings_update', eventData);
    
    // Broadcast to WebSocket subscribers
    wsServer.broadcastToChannel(`tournament:${tournamentId}:standings`, eventData);
  }

  static emitUserNotification(userId: string, notification: any) {
    const eventData = {
      type: 'user_notification',
      notification,
      timestamp: new Date().toISOString(),
    };

    // Send to specific user via WebSocket
    wsServer.sendToUser(userId, eventData);
  }

  static emitSystemAnnouncement(announcement: any, targetRoles?: string[]) {
    const eventData = {
      type: 'system_announcement',
      announcement,
      targetRoles,
      timestamp: new Date().toISOString(),
    };

    // Broadcast to all connections (filtered by role if specified)
    wsServer.broadcastToChannel('system', eventData);
  }
}
```

#### Acceptance Criteria
- [ ] WebSocket server accepts and manages connections
- [ ] tRPC subscriptions provide type-safe real-time data
- [ ] Connection authentication working correctly
- [ ] Event broadcasting functional across channels
- [ ] Ping/pong mechanism maintains connections

### Phase 2: Real-time Game & Tournament Updates (2 days)

#### Tasks
1. **Implement Game Status Broadcasting**
   - Real-time game result updates
   - Live game progress tracking
   - Player notification system

2. **Create Tournament Event System**
   - Round start/end notifications
   - Registration updates broadcasting
   - Tournament status changes

#### Implementation

```typescript
// src/app/_components/realtime/realtime-provider.tsx
'use client';

import { createContext, useContext, useEffect, useState, ReactNode } from 'react';
import { api } from '@/trpc/react';
import { toast } from 'sonner';

interface RealtimeContextType {
  isConnected: boolean;
  connectionId?: string;
  subscribe: (channel: string) => void;
  unsubscribe: (channel: string) => void;
}

const RealtimeContext = createContext<RealtimeContextType | null>(null);

interface RealtimeProviderProps {
  children: ReactNode;
  userId?: string;
}

export function RealtimeProvider({ children, userId }: RealtimeProviderProps) {
  const [isConnected, setIsConnected] = useState(false);
  const [connectionId, setConnectionId] = useState<string>();
  const [ws, setWs] = useState<WebSocket | null>(null);
  const [subscriptions, setSubscriptions] = useState<Set<string>>(new Set());

  // Get user's notification preferences
  const { data: preferences } = api.realtime.getNotificationPreferences.useQuery(
    undefined,
    { enabled: !!userId }
  );

  useEffect(() => {
    const connectWebSocket = () => {
      const protocol = window.location.protocol === 'https:' ? 'wss:' : 'ws:';
      const wsUrl = `${protocol}//${window.location.host}/ws${userId ? `?token=${localStorage.getItem('auth-token')}` : ''}`;
      
      const websocket = new WebSocket(wsUrl);

      websocket.onopen = () => {
        setIsConnected(true);
        console.log('WebSocket connected');
      };

      websocket.onmessage = (event) => {
        try {
          const message = JSON.parse(event.data);
          handleRealtimeMessage(message);
        } catch (error) {
          console.error('Error parsing WebSocket message:', error);
        }
      };

      websocket.onclose = () => {
        setIsConnected(false);
        setConnectionId(undefined);
        console.log('WebSocket disconnected');
        
        // Attempt to reconnect after 3 seconds
        setTimeout(connectWebSocket, 3000);
      };

      websocket.onerror = (error) => {
        console.error('WebSocket error:', error);
      };

      setWs(websocket);
    };

    connectWebSocket();

    return () => {
      ws?.close();
    };
  }, [userId]);

  const handleRealtimeMessage = (message: any) => {
    switch (message.type) {
      case 'connected':
        setConnectionId(message.sessionId);
        break;
        
      case 'broadcast':
        handleBroadcastMessage(message);
        break;
        
      case 'user_message':
        handleUserMessage(message);
        break;
        
      default:
        console.log('Received message:', message);
    }
  };

  const handleBroadcastMessage = (message: any) => {
    const { channel, data } = message;
    
    switch (data.type) {
      case 'game_result_submitted':
        if (preferences?.gameResults) {
          toast.success(`Game result updated: ${data.result}`);
        }
        break;
        
      case 'tournament_started':
        if (preferences?.tournamentUpdates) {
          toast.info(`Tournament "${data.tournamentName}" has started!`);
        }
        break;
        
      case 'round_started':
        if (preferences?.pairingUpdates) {
          toast.info(`Round ${data.roundNumber} has started`);
        }
        break;
        
      case 'standings_updated':
        if (preferences?.standingsChanges) {
          // Update is handled by the standings component subscription
        }
        break;
    }
  };

  const handleUserMessage = (message: any) => {
    const { data } = message;
    
    switch (data.type) {
      case 'user_notification':
        toast(data.notification.message, {
          description: data.notification.details,
        });
        break;
        
      case 'game_pairing':
        toast.info('You have been paired for a new game!', {
          action: {
            label: 'View Game',
            onClick: () => window.location.href = `/games/${data.gameId}`,
          },
        });
        break;
    }
  };

  const subscribe = (channel: string) => {
    if (ws?.readyState === WebSocket.OPEN && !subscriptions.has(channel)) {
      ws.send(JSON.stringify({ type: 'subscribe', channel }));
      setSubscriptions(prev => new Set(prev).add(channel));
    }
  };

  const unsubscribe = (channel: string) => {
    if (ws?.readyState === WebSocket.OPEN && subscriptions.has(channel)) {
      ws.send(JSON.stringify({ type: 'unsubscribe', channel }));
      setSubscriptions(prev => {
        const newSet = new Set(prev);
        newSet.delete(channel);
        return newSet;
      });
    }
  };

  const value: RealtimeContextType = {
    isConnected,
    connectionId,
    subscribe,
    unsubscribe,
  };

  return (
    <RealtimeContext.Provider value={value}>
      {children}
    </RealtimeContext.Provider>
  );
}

export function useRealtime() {
  const context = useContext(RealtimeContext);
  if (!context) {
    throw new Error('useRealtime must be used within a RealtimeProvider');
  }
  return context;
}

// src/app/_components/realtime/live-scoreboard.tsx
'use client';

import { useEffect, useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { api } from '@/trpc/react';
import { useRealtime } from './realtime-provider';

interface LiveScoreboardProps {
  tournamentId: string;
}

export function LiveScoreboard({ tournamentId }: LiveScoreboardProps) {
  const { subscribe, unsubscribe } = useRealtime();
  const [lastUpdate, setLastUpdate] = useState<Date>(new Date());

  // Get initial scoreboard data
  const { data: scoreboard, refetch } = api.scoreboard.getByTournament.useQuery({
    tournamentId,
  });

  // Subscribe to real-time updates
  const gameUpdateSubscription = api.realtime.onGameUpdate.useSubscription(
    { tournamentId },
    {
      onData: (data) => {
        console.log('Game update received:', data);
        setLastUpdate(new Date());
        refetch(); // Refetch scoreboard data
      },
    }
  );

  useEffect(() => {
    // Subscribe to WebSocket channel
    subscribe(`tournament:${tournamentId}:games`);
    
    return () => {
      unsubscribe(`tournament:${tournamentId}:games`);
    };
  }, [tournamentId, subscribe, unsubscribe]);

  if (!scoreboard) {
    return (
      <Card>
        <CardContent className="p-6">
          <div className="text-center">Loading scoreboard...</div>
        </CardContent>
      </Card>
    );
  }

  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between">
        <CardTitle>Live Scoreboard</CardTitle>
        <div className="flex items-center gap-2">
          <div className="h-2 w-2 rounded-full bg-green-500 animate-pulse"></div>
          <span className="text-xs text-muted-foreground">
            Last update: {lastUpdate.toLocaleTimeString()}
          </span>
        </div>
      </CardHeader>
      
      <CardContent>
        <div className="space-y-4">
          {scoreboard.rounds?.map(round => (
            <div key={round.id} className="space-y-2">
              <h3 className="font-medium">Round {round.roundNumber}</h3>
              <div className="grid gap-2">
                {round.games?.map(game => (
                  <div key={game.id} className="flex items-center justify-between p-3 border rounded-lg">
                    <div className="flex items-center gap-4">
                      <div className="text-sm">
                        <span className="font-medium">{game.white.name}</span>
                        <span className="text-muted-foreground"> vs </span>
                        <span className="font-medium">{game.black.name}</span>
                      </div>
                    </div>
                    
                    <div className="flex items-center gap-2">
                      {game.result ? (
                        <Badge variant={
                          game.result === 'WHITE_WINS' ? 'default' :
                          game.result === 'BLACK_WINS' ? 'secondary' :
                          'outline'
                        }>
                          {game.result === 'DRAW' ? '½-½' :
                           game.result === 'WHITE_WINS' ? '1-0' :
                           game.result === 'BLACK_WINS' ? '0-1' :
                           game.result.replace('_', ' ')}
                        </Badge>
                      ) : (
                        <Badge variant="outline">In Progress</Badge>
                      )}
                    </div>
                  </div>
                ))}
              </div>
            </div>
          ))}
        </div>
      </CardContent>
    </Card>
  );
}

// src/app/_components/realtime/live-standings.tsx
'use client';

import { useEffect, useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { Badge } from '@/components/ui/badge';
import { Trophy, TrendingUp, TrendingDown, Minus } from 'lucide-react';
import { api } from '@/trpc/react';
import { useRealtime } from './realtime-provider';

interface LiveStandingsProps {
  tournamentId: string;
}

interface StandingWithChange {
  id: string;
  userId: string;
  userName: string;
  score: number;
  rank: number;
  previousRank?: number;
  tiebreaks: Record<string, number>;
}

export function LiveStandings({ tournamentId }: LiveStandingsProps) {
  const { subscribe, unsubscribe } = useRealtime();
  const [standings, setStandings] = useState<StandingWithChange[]>([]);
  const [lastUpdate, setLastUpdate] = useState<Date>(new Date());

  // Get initial standings data
  const { data: initialStandings } = api.standings.getByTournament.useQuery({
    tournamentId,
  });

  // Subscribe to real-time standings updates
  const standingsSubscription = api.realtime.onStandingsUpdate.useSubscription(
    { tournamentId },
    {
      onData: (data) => {
        console.log('Standings update received:', data);
        
        // Calculate rank changes
        const newStandings = data.standings.map((standing: any, index: number) => {
          const previousStanding = standings.find(s => s.userId === standing.userId);
          return {
            ...standing,
            rank: index + 1,
            previousRank: previousStanding?.rank,
          };
        });
        
        setStandings(newStandings);
        setLastUpdate(new Date());
      },
    }
  );

  useEffect(() => {
    if (initialStandings) {
      const formattedStandings = initialStandings.map((standing, index) => ({
        id: standing.id,
        userId: standing.userId,
        userName: standing.user.name || standing.user.email,
        score: standing.score,
        rank: index + 1,
        tiebreaks: standing.tiebreaks || {},
      }));
      setStandings(formattedStandings);
    }
  }, [initialStandings]);

  useEffect(() => {
    // Subscribe to WebSocket channel
    subscribe(`tournament:${tournamentId}:standings`);
    
    return () => {
      unsubscribe(`tournament:${tournamentId}:standings`);
    };
  }, [tournamentId, subscribe, unsubscribe]);

  const getRankChangeIcon = (current: number, previous?: number) => {
    if (!previous || current === previous) {
      return <Minus className="h-4 w-4 text-muted-foreground" />;
    }
    
    if (current < previous) {
      return <TrendingUp className="h-4 w-4 text-green-500" />;
    }
    
    return <TrendingDown className="h-4 w-4 text-red-500" />;
  };

  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between">
        <CardTitle className="flex items-center gap-2">
          <Trophy className="h-5 w-5" />
          Live Standings
        </CardTitle>
        <div className="flex items-center gap-2">
          <div className="h-2 w-2 rounded-full bg-green-500 animate-pulse"></div>
          <span className="text-xs text-muted-foreground">
            Updated: {lastUpdate.toLocaleTimeString()}
          </span>
        </div>
      </CardHeader>
      
      <CardContent>
        <Table>
          <TableHeader>
            <TableRow>
              <TableHead className="w-16">Rank</TableHead>
              <TableHead>Change</TableHead>
              <TableHead>Player</TableHead>
              <TableHead className="text-center">Score</TableHead>
              <TableHead className="text-center">Buchholz</TableHead>
              <TableHead className="text-center">S-B</TableHead>
            </TableRow>
          </TableHeader>
          <TableBody>
            {standings.map((standing) => (
              <TableRow key={standing.userId} className="group">
                <TableCell>
                  <div className="flex items-center gap-2">
                    {standing.rank === 1 && (
                      <Trophy className="h-4 w-4 text-yellow-500" />
                    )}
                    <span className="font-medium">#{standing.rank}</span>
                  </div>
                </TableCell>
                
                <TableCell>
                  {getRankChangeIcon(standing.rank, standing.previousRank)}
                </TableCell>
                
                <TableCell>
                  <div className="font-medium">{standing.userName}</div>
                </TableCell>
                
                <TableCell className="text-center">
                  <Badge variant="outline">{standing.score}</Badge>
                </TableCell>
                
                <TableCell className="text-center">
                  <span className="text-sm text-muted-foreground">
                    {standing.tiebreaks.buchholz?.toFixed(1) || '0.0'}
                  </span>
                </TableCell>
                
                <TableCell className="text-center">
                  <span className="text-sm text-muted-foreground">
                    {standing.tiebreaks.sonneborn?.toFixed(1) || '0.0'}
                  </span>
                </TableCell>
              </TableRow>
            ))}
          </TableBody>
        </Table>
      </CardContent>
    </Card>
  );
}
```

#### Acceptance Criteria
- [ ] Game results broadcast in real-time
- [ ] Tournament events trigger immediate notifications
- [ ] Standings update instantly when games complete
- [ ] Player-specific notifications delivered correctly
- [ ] UI components reflect real-time data changes

### Phase 3: Live Dashboard & Admin Notifications (1 day)

#### Tasks
1. **Enhance Dashboard with Real-time Data**
   - Live widget updates without page refresh
   - Real-time system health monitoring
   - Dynamic user activity tracking

2. **Implement Admin Notification System**
   - System-wide announcement broadcasts
   - Emergency alert capabilities
   - Targeted user notifications

#### Implementation

```typescript
// src/app/_components/dashboard/widgets/live-tournament-widget.tsx
'use client';

import { useEffect } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Trophy, Users, Clock } from 'lucide-react';
import { api } from '@/trpc/react';
import { useRealtime } from '@/app/_components/realtime/realtime-provider';

export function LiveTournamentWidget() {
  const { subscribe, unsubscribe } = useRealtime();
  
  // Get tournament overview with real-time updates
  const { data: overview, refetch } = api.dashboard.getTournamentOverview.useQuery();

  // Subscribe to tournament updates
  const tournamentSubscription = api.realtime.onTournamentUpdate.useSubscription(
    { tournamentId: 'all' }, // Special channel for all tournaments
    {
      onData: (data) => {
        console.log('Tournament overview update:', data);
        refetch(); // Refresh tournament data
      },
    }
  );

  useEffect(() => {
    // Subscribe to WebSocket channel for all tournament updates
    subscribe('tournaments:all');
    
    return () => {
      unsubscribe('tournaments:all');
    };
  }, [subscribe, unsubscribe]);

  if (!overview) {
    return (
      <Card>
        <CardContent className="p-6">
          <div className="animate-pulse space-y-4">
            <div className="h-4 bg-muted rounded w-3/4"></div>
            <div className="h-4 bg-muted rounded w-1/2"></div>
          </div>
        </CardContent>
      </Card>
    );
  }

  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
        <CardTitle className="text-sm font-medium">Live Tournament Status</CardTitle>
        <div className="flex items-center gap-2">
          <div className="h-2 w-2 rounded-full bg-green-500 animate-pulse"></div>
          <Trophy className="h-4 w-4 text-muted-foreground" />
        </div>
      </CardHeader>
      <CardContent>
        <div className="space-y-4">
          <div className="grid grid-cols-3 gap-4">
            <div className="text-center">
              <div className="text-2xl font-bold">{overview.total}</div>
              <div className="text-xs text-muted-foreground">Total</div>
            </div>
            <div className="text-center">
              <div className="text-2xl font-bold text-green-600">{overview.active}</div>
              <div className="text-xs text-muted-foreground">Active</div>
            </div>
            <div className="text-center">
              <div className="text-2xl font-bold text-blue-600">{overview.upcoming}</div>
              <div className="text-xs text-muted-foreground">Upcoming</div>
            </div>
          </div>

          {overview.activeTournaments?.length > 0 && (
            <div>
              <div className="text-xs font-medium mb-2">Live Tournaments</div>
              <div className="space-y-1">
                {overview.activeTournaments.map(tournament => (
                  <div key={tournament.id} className="flex items-center justify-between text-xs p-2 bg-muted/50 rounded">
                    <span className="truncate">{tournament.name}</span>
                    <div className="flex items-center gap-1">
                      <Users className="h-3 w-3" />
                      <span>{tournament.participantCount}</span>
                    </div>
                  </div>
                ))}
              </div>
            </div>
          )}
        </div>
      </CardContent>
    </Card>
  );
}

// src/app/_components/admin/notification-center.tsx
'use client';

import { useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Textarea } from '@/components/ui/textarea';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Badge } from '@/components/ui/badge';
import { Bell, Send, AlertTriangle, Info, CheckCircle, X } from 'lucide-react';
import { api } from '@/trpc/react';
import { toast } from 'sonner';

export function AdminNotificationCenter() {
  const [title, setTitle] = useState('');
  const [message, setMessage] = useState('');
  const [type, setType] = useState<'INFO' | 'SUCCESS' | 'WARNING' | 'ERROR' | 'ANNOUNCEMENT'>('INFO');
  const [priority, setPriority] = useState<'LOW' | 'NORMAL' | 'HIGH' | 'CRITICAL'>('NORMAL');
  const [targetRoles, setTargetRoles] = useState<string[]>([]);

  // Get recent notifications
  const { data: notifications, refetch } = api.realtime.getRecentNotifications.useQuery();

  // Create notification mutation
  const createNotification = api.realtime.createNotification.useMutation({
    onSuccess: () => {
      toast.success('Notification sent successfully');
      setTitle('');
      setMessage('');
      setTargetRoles([]);
      refetch();
    },
    onError: (error) => {
      toast.error('Failed to send notification: ' + error.message);
    },
  });

  const handleSendNotification = () => {
    if (!title.trim() || !message.trim()) {
      toast.error('Title and message are required');
      return;
    }

    createNotification.mutate({
      title,
      message,
      type,
      priority,
      targetRoles: targetRoles as any,
    });
  };

  const getTypeIcon = (notificationType: string) => {
    switch (notificationType) {
      case 'SUCCESS':
        return <CheckCircle className="h-4 w-4 text-green-500" />;
      case 'WARNING':
        return <AlertTriangle className="h-4 w-4 text-yellow-500" />;
      case 'ERROR':
        return <X className="h-4 w-4 text-red-500" />;
      default:
        return <Info className="h-4 w-4 text-blue-500" />;
    }
  };

  return (
    <div className="space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Notification Center</h1>
        <Bell className="h-6 w-6 text-muted-foreground" />
      </div>

      <div className="grid grid-cols-1 lg:grid-cols-2 gap-6">
        {/* Send Notification */}
        <Card>
          <CardHeader>
            <CardTitle>Send Notification</CardTitle>
          </CardHeader>
          <CardContent className="space-y-4">
            <div>
              <label className="text-sm font-medium">Title</label>
              <Input
                value={title}
                onChange={(e) => setTitle(e.target.value)}
                placeholder="Notification title"
              />
            </div>

            <div>
              <label className="text-sm font-medium">Message</label>
              <Textarea
                value={message}
                onChange={(e) => setMessage(e.target.value)}
                placeholder="Notification message"
                rows={3}
              />
            </div>

            <div className="grid grid-cols-2 gap-4">
              <div>
                <label className="text-sm font-medium">Type</label>
                <Select value={type} onValueChange={(value: any) => setType(value)}>
                  <SelectTrigger>
                    <SelectValue />
                  </SelectTrigger>
                  <SelectContent>
                    <SelectItem value="INFO">Info</SelectItem>
                    <SelectItem value="SUCCESS">Success</SelectItem>
                    <SelectItem value="WARNING">Warning</SelectItem>
                    <SelectItem value="ERROR">Error</SelectItem>
                    <SelectItem value="ANNOUNCEMENT">Announcement</SelectItem>
                  </SelectContent>
                </Select>
              </div>

              <div>
                <label className="text-sm font-medium">Priority</label>
                <Select value={priority} onValueChange={(value: any) => setPriority(value)}>
                  <SelectTrigger>
                    <SelectValue />
                  </SelectTrigger>
                  <SelectContent>
                    <SelectItem value="LOW">Low</SelectItem>
                    <SelectItem value="NORMAL">Normal</SelectItem>
                    <SelectItem value="HIGH">High</SelectItem>
                    <SelectItem value="CRITICAL">Critical</SelectItem>
                  </SelectContent>
                </Select>
              </div>
            </div>

            <div>
              <label className="text-sm font-medium">Target Roles</label>
              <div className="flex gap-2 mt-1">
                {['ADMIN', 'ORGANIZER', 'PARTICIPANT', 'VIEWER'].map(role => (
                  <Button
                    key={role}
                    variant={targetRoles.includes(role) ? 'default' : 'outline'}
                    size="sm"
                    onClick={() => {
                      setTargetRoles(prev =>
                        prev.includes(role)
                          ? prev.filter(r => r !== role)
                          : [...prev, role]
                      );
                    }}
                  >
                    {role}
                  </Button>
                ))}
              </div>
            </div>

            <Button
              onClick={handleSendNotification}
              disabled={createNotification.isPending}
              className="w-full"
            >
              <Send className="h-4 w-4 mr-2" />
              Send Notification
            </Button>
          </CardContent>
        </Card>

        {/* Recent Notifications */}
        <Card>
          <CardHeader>
            <CardTitle>Recent Notifications</CardTitle>
          </CardHeader>
          <CardContent>
            {notifications ? (
              <div className="space-y-3">
                {notifications.map(notification => (
                  <div key={notification.id} className="flex items-start gap-3 p-3 border rounded-lg">
                    {getTypeIcon(notification.type)}
                    <div className="flex-1 min-w-0">
                      <div className="flex items-center gap-2 mb-1">
                        <span className="font-medium text-sm">{notification.title}</span>
                        <Badge variant="outline" className="text-xs">
                          {notification.priority}
                        </Badge>
                      </div>
                      <p className="text-sm text-muted-foreground">{notification.message}</p>
                      <div className="flex items-center gap-2 mt-1">
                        <span className="text-xs text-muted-foreground">
                          {new Date(notification.createdAt).toLocaleString()}
                        </span>
                        <div className="flex gap-1">
                          {notification.targetRoles.map(role => (
                            <Badge key={role} variant="secondary" className="text-xs">
                              {role}
                            </Badge>
                          ))}
                        </div>
                      </div>
                    </div>
                  </div>
                ))}
              </div>
            ) : (
              <div className="text-center text-muted-foreground">
                Loading notifications...
              </div>
            )}
          </CardContent>
        </Card>
      </div>
    </div>
  );
}
```

#### Acceptance Criteria
- [ ] Dashboard widgets update in real-time
- [ ] Admin notifications broadcast immediately
- [ ] System health monitoring shows live data
- [ ] Emergency alerts reach all connected users
- [ ] Notification preferences respected

### Phase 4: Connection Management & Performance (1 day)

#### Tasks
1. **Implement Connection Optimization**
   - Connection pooling and cleanup
   - Bandwidth optimization for large tournaments
   - Graceful degradation for slow connections

2. **Add Monitoring & Analytics**
   - Real-time connection metrics
   - Performance monitoring dashboard
   - Error tracking and alerting

#### Acceptance Criteria
- [ ] Connection management efficient and stable
- [ ] Performance optimized for scale
- [ ] Monitoring dashboard functional
- [ ] Error handling robust and informative
- [ ] Memory usage within acceptable limits

## Risk Assessment

### High Risk
- **WebSocket scalability**: Managing connections for large tournaments
- **Data synchronization**: Ensuring consistency across real-time updates
- **Connection stability**: Handling network interruptions gracefully

### Medium Risk
- **Performance overhead**: Real-time updates impacting system performance
- **Browser compatibility**: WebSocket support across different browsers

### Low Risk
- **Event serialization**: Standard JSON message formatting
- **UI responsiveness**: React state updates for real-time data

## Success Metrics

- Real-time update latency: <1 second
- Connection stability: >99% uptime
- WebSocket message delivery: >99.9% success rate
- UI responsiveness: No blocking updates
- Memory usage: Stable over extended periods

## Dependencies for Next Phase

This real-time system enables:
- Enhanced user experience with live updates
- Real-time competitive tournament atmosphere
- Improved administrative oversight and control
- Foundation for advanced analytics and reporting
