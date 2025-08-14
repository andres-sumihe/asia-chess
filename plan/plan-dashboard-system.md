# Dashboard System Implementation Plan

## Feature Overview

Implement comprehensive administrative dashboard for tournament organizers and platform administrators with centralized management, analytics, monitoring, and reporting capabilities.

## Objectives

- Centralized tournament management interface
- User and participant administration
- Real-time system monitoring and analytics
- Administrative tools and bulk operations
- Platform settings and configuration management
- Comprehensive reporting and insights

## Technical Requirements

### Dependencies
- All previous systems (auth, tournaments, participants, games, scoreboard, standings)
- Real-time updates infrastructure
- Authentication and authorization system
- Analytics and reporting data pipeline

### Constraints
- Role-based access control required
- Real-time data updates essential
- Mobile-responsive design mandatory
- Performance optimized for large datasets
- Secure administrative operations

## Implementation Phases

### Phase 1: Dashboard Foundation & Layout (2 days)

#### Tasks
1. **Create Dashboard Layout System**
   - Responsive sidebar navigation
   - Main content area with widgets
   - Header with user context and notifications

2. **Implement Role-Based Access**
   - Admin vs Organizer vs Viewer permissions
   - Component-level access control
   - Navigation menu filtering

#### Database Changes

```prisma
// Enhanced User model for dashboard access
model User {
  // Existing fields...
  
  // Dashboard preferences
  dashboardConfig Json?        // Layout preferences, widget settings
  lastLogin       DateTime?
  loginCount      Int          @default(0)
  
  // Administrative tracking
  adminNotes      AdminNote[]
  auditLogs       AuditLog[]
  
  // Notification preferences
  emailNotifications Boolean   @default(true)
  pushNotifications  Boolean   @default(true)
  dashboardAlerts    Boolean   @default(true)
}

// Administrative notes on users/tournaments
model AdminNote {
  id          String    @id @default(cuid())
  userId      String?   // Subject user
  tournamentId String?  // Subject tournament
  authorId    String    // Admin who created note
  
  title       String
  content     String
  priority    Priority  @default(NORMAL)
  isPrivate   Boolean   @default(true)
  
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  
  // Relations
  user        User?     @relation(fields: [userId], references: [id], onDelete: Cascade)
  tournament  Tournament? @relation(fields: [tournamentId], references: [id], onDelete: Cascade)
  author      User      @relation(fields: [authorId], references: [id])
  
  @@index([userId])
  @@index([tournamentId])
  @@index([authorId])
}

// System audit log
model AuditLog {
  id          String    @id @default(cuid())
  userId      String?   // User who performed action
  action      String    // Action performed
  entityType  String    // Type of entity affected
  entityId    String?   // ID of affected entity
  
  // Action details
  details     Json?     // Additional action data
  ipAddress   String?
  userAgent   String?
  
  // Metadata
  timestamp   DateTime  @default(now())
  severity    LogSeverity @default(INFO)
  
  // Relations
  user        User?     @relation(fields: [userId], references: [id])
  
  @@index([userId, timestamp])
  @@index([action, timestamp])
  @@index([entityType, entityId])
}

// System-wide notifications
model SystemNotification {
  id            String     @id @default(cuid())
  title         String
  message       String
  type          NotificationType
  priority      Priority   @default(NORMAL)
  
  // Targeting
  targetRoles   UserRole[] // Which roles should see this
  targetUsers   String[]   // Specific user IDs (JSON array)
  
  // Display settings
  isActive      Boolean    @default(true)
  showUntil     DateTime?  // Auto-expire date
  isDismissible Boolean    @default(true)
  
  // Tracking
  createdBy     String
  createdAt     DateTime   @default(now())
  updatedAt     DateTime   @updatedAt
  
  // Relations
  creator       User       @relation(fields: [createdBy], references: [id])
  
  @@index([isActive, targetRoles])
}

// Dashboard widgets and layout
model DashboardWidget {
  id            String     @id @default(cuid())
  userId        String     // Owner of the widget
  
  // Widget configuration
  type          WidgetType
  title         String
  position      Json       // { x, y, width, height }
  config        Json?      // Widget-specific settings
  
  // Display settings
  isVisible     Boolean    @default(true)
  refreshInterval Int      @default(30) // seconds
  
  // Relations
  user          User       @relation(fields: [userId], references: [id], onDelete: Cascade)
  
  @@index([userId, isVisible])
}

enum Priority {
  LOW
  NORMAL
  HIGH
  CRITICAL
}

enum LogSeverity {
  DEBUG
  INFO
  WARNING
  ERROR
  CRITICAL
}

enum NotificationType {
  INFO
  SUCCESS
  WARNING
  ERROR
  ANNOUNCEMENT
}

enum WidgetType {
  TOURNAMENT_OVERVIEW
  ACTIVE_GAMES
  USER_STATISTICS
  RECENT_ACTIVITY
  SYSTEM_HEALTH
  REVENUE_METRICS
  PARTICIPANT_GROWTH
  PERFORMANCE_CHARTS
  QUICK_ACTIONS
  NOTIFICATIONS
}
```

#### Implementation

```typescript
// src/app/_components/dashboard/dashboard-layout.tsx
'use client';

import { useState, useEffect } from 'react';
import { useSession } from 'next-auth/react';
import { Sidebar } from './sidebar';
import { Header } from './header';
import { NotificationCenter } from './notification-center';
import { api } from '@/trpc/react';

interface DashboardLayoutProps {
  children: React.ReactNode;
}

export function DashboardLayout({ children }: DashboardLayoutProps) {
  const { data: session } = useSession();
  const [sidebarOpen, setSidebarOpen] = useState(true);
  const [notificationsOpen, setNotificationsOpen] = useState(false);

  // Get user dashboard configuration
  const { data: dashboardConfig } = api.dashboard.getConfig.useQuery();

  // Get system notifications
  const { data: notifications } = api.dashboard.getNotifications.useQuery();

  if (!session?.user) {
    return null;
  }

  return (
    <div className="min-h-screen bg-background">
      <Sidebar 
        isOpen={sidebarOpen} 
        onToggle={setSidebarOpen}
        userRole={session.user.role}
      />
      
      <div className={`transition-all duration-300 ${sidebarOpen ? 'ml-64' : 'ml-16'}`}>
        <Header 
          user={session.user}
          notifications={notifications}
          onNotificationsToggle={() => setNotificationsOpen(!notificationsOpen)}
        />
        
        <main className="p-6">
          {children}
        </main>
      </div>

      <NotificationCenter
        isOpen={notificationsOpen}
        onClose={() => setNotificationsOpen(false)}
        notifications={notifications || []}
      />
    </div>
  );
}

// src/app/_components/dashboard/sidebar.tsx
'use client';

import { useState } from 'react';
import Link from 'next/link';
import { usePathname } from 'next/navigation';
import { cn } from '@/lib/utils';
import { Button } from '@/components/ui/button';
import { 
  Home, 
  Trophy, 
  Users, 
  BarChart3, 
  Settings, 
  FileText,
  Shield,
  Bell,
  ChevronLeft,
  ChevronRight 
} from 'lucide-react';

interface SidebarProps {
  isOpen: boolean;
  onToggle: (open: boolean) => void;
  userRole: string;
}

export function Sidebar({ isOpen, onToggle, userRole }: SidebarProps) {
  const pathname = usePathname();

  const navigation = [
    {
      name: 'Overview',
      href: '/dashboard',
      icon: Home,
      roles: ['ADMIN', 'ORGANIZER'],
    },
    {
      name: 'Tournaments',
      href: '/dashboard/tournaments',
      icon: Trophy,
      roles: ['ADMIN', 'ORGANIZER'],
    },
    {
      name: 'Participants',
      href: '/dashboard/participants',
      icon: Users,
      roles: ['ADMIN', 'ORGANIZER'],
    },
    {
      name: 'Analytics',
      href: '/dashboard/analytics',
      icon: BarChart3,
      roles: ['ADMIN', 'ORGANIZER'],
    },
    {
      name: 'Reports',
      href: '/dashboard/reports',
      icon: FileText,
      roles: ['ADMIN', 'ORGANIZER'],
    },
    {
      name: 'User Management',
      href: '/dashboard/users',
      icon: Shield,
      roles: ['ADMIN'],
    },
    {
      name: 'System Settings',
      href: '/dashboard/settings',
      icon: Settings,
      roles: ['ADMIN'],
    },
    {
      name: 'Notifications',
      href: '/dashboard/notifications',
      icon: Bell,
      roles: ['ADMIN'],
    },
  ];

  const filteredNavigation = navigation.filter(item => 
    item.roles.includes(userRole as any)
  );

  return (
    <div className={cn(
      "fixed top-0 left-0 z-40 h-screen transition-all duration-300 bg-card border-r",
      isOpen ? "w-64" : "w-16"
    )}>
      <div className="flex items-center justify-between p-4 border-b">
        {isOpen && (
          <h1 className="text-lg font-semibold">Chess Admin</h1>
        )}
        <Button
          variant="ghost"
          size="icon"
          onClick={() => onToggle(!isOpen)}
        >
          {isOpen ? <ChevronLeft /> : <ChevronRight />}
        </Button>
      </div>

      <nav className="p-4 space-y-2">
        {filteredNavigation.map((item) => {
          const isActive = pathname === item.href;
          
          return (
            <Link key={item.name} href={item.href}>
              <div className={cn(
                "flex items-center gap-3 px-3 py-2 rounded-lg transition-colors",
                isActive 
                  ? "bg-primary text-primary-foreground" 
                  : "hover:bg-muted"
              )}>
                <item.icon className="h-5 w-5 flex-shrink-0" />
                {isOpen && (
                  <span className="text-sm font-medium">{item.name}</span>
                )}
              </div>
            </Link>
          );
        })}
      </nav>
    </div>
  );
}
```

#### Acceptance Criteria
- [ ] Dashboard layout responsive and functional
- [ ] Role-based navigation implemented
- [ ] User preferences saved correctly
- [ ] Mobile-friendly design verified
- [ ] Navigation state persistence working

### Phase 2: Dashboard Widgets & Analytics (3 days)

#### Tasks
1. **Create Dashboard Widget System**
   - Configurable widget grid layout
   - Real-time data widgets
   - Customizable widget settings

2. **Implement Core Analytics Widgets**
   - Tournament overview metrics
   - User activity statistics
   - System performance indicators

#### Implementation

```typescript
// src/app/_components/dashboard/widgets/widget-grid.tsx
'use client';

import { useState, useCallback } from 'react';
import { Responsive, WidthProvider } from 'react-grid-layout';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Button } from '@/components/ui/button';
import { Settings, X } from 'lucide-react';
import { api } from '@/trpc/react';
import { TournamentOverviewWidget } from './tournament-overview-widget';
import { ActiveGamesWidget } from './active-games-widget';
import { UserStatisticsWidget } from './user-statistics-widget';
import { RecentActivityWidget } from './recent-activity-widget';
import { SystemHealthWidget } from './system-health-widget';

const ResponsiveGridLayout = WidthProvider(Responsive);

interface Widget {
  id: string;
  type: string;
  title: string;
  position: { x: number; y: number; w: number; h: number };
  config?: any;
}

export function WidgetGrid() {
  const [editMode, setEditMode] = useState(false);

  // Get user's widget configuration
  const { data: widgets, refetch } = api.dashboard.getWidgets.useQuery();
  
  // Save widget layout
  const saveLayout = api.dashboard.saveWidgetLayout.useMutation({
    onSuccess: () => refetch(),
  });

  const handleLayoutChange = useCallback((layout: any[]) => {
    if (!editMode || !widgets) return;

    const updatedWidgets = widgets.map(widget => {
      const layoutItem = layout.find(item => item.i === widget.id);
      if (layoutItem) {
        return {
          ...widget,
          position: {
            x: layoutItem.x,
            y: layoutItem.y,
            w: layoutItem.w,
            h: layoutItem.h,
          },
        };
      }
      return widget;
    });

    saveLayout.mutate({ widgets: updatedWidgets });
  }, [editMode, widgets, saveLayout]);

  const renderWidget = (widget: Widget) => {
    const commonProps = {
      title: widget.title,
      config: widget.config,
      onEdit: editMode ? () => console.log('Edit widget', widget.id) : undefined,
      onRemove: editMode ? () => console.log('Remove widget', widget.id) : undefined,
    };

    switch (widget.type) {
      case 'TOURNAMENT_OVERVIEW':
        return <TournamentOverviewWidget {...commonProps} />;
      case 'ACTIVE_GAMES':
        return <ActiveGamesWidget {...commonProps} />;
      case 'USER_STATISTICS':
        return <UserStatisticsWidget {...commonProps} />;
      case 'RECENT_ACTIVITY':
        return <RecentActivityWidget {...commonProps} />;
      case 'SYSTEM_HEALTH':
        return <SystemHealthWidget {...commonProps} />;
      default:
        return (
          <Card>
            <CardContent className="p-6">
              <div className="text-center text-muted-foreground">
                Unknown widget type: {widget.type}
              </div>
            </CardContent>
          </Card>
        );
    }
  };

  if (!widgets) {
    return <div>Loading dashboard...</div>;
  }

  const layouts = {
    lg: widgets.map(w => ({ i: w.id, ...w.position })),
    md: widgets.map(w => ({ i: w.id, ...w.position })),
    sm: widgets.map(w => ({ i: w.id, x: 0, y: w.position.y, w: 12, h: w.position.h })),
  };

  return (
    <div className="space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">Dashboard</h1>
        <Button
          variant={editMode ? "default" : "outline"}
          onClick={() => setEditMode(!editMode)}
        >
          <Settings className="w-4 h-4 mr-2" />
          {editMode ? "Done Editing" : "Customize"}
        </Button>
      </div>

      <ResponsiveGridLayout
        className="layout"
        layouts={layouts}
        breakpoints={{ lg: 1200, md: 996, sm: 768 }}
        cols={{ lg: 12, md: 10, sm: 6 }}
        rowHeight={120}
        isDraggable={editMode}
        isResizable={editMode}
        onLayoutChange={handleLayoutChange}
        margin={[16, 16]}
      >
        {widgets.map(widget => (
          <div key={widget.id}>
            {renderWidget(widget)}
          </div>
        ))}
      </ResponsiveGridLayout>
    </div>
  );
}

// src/app/_components/dashboard/widgets/tournament-overview-widget.tsx
'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Badge } from '@/components/ui/badge';
import { Trophy, Users, Clock, TrendingUp } from 'lucide-react';
import { api } from '@/trpc/react';

interface TournamentOverviewWidgetProps {
  title: string;
  config?: any;
  onEdit?: () => void;
  onRemove?: () => void;
}

export function TournamentOverviewWidget({ title, config, onEdit, onRemove }: TournamentOverviewWidgetProps) {
  const { data: overview, isLoading } = api.dashboard.getTournamentOverview.useQuery();

  if (isLoading) {
    return (
      <Card>
        <CardHeader>
          <CardTitle>{title}</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="animate-pulse space-y-4">
            <div className="h-4 bg-muted rounded w-3/4"></div>
            <div className="h-4 bg-muted rounded w-1/2"></div>
          </div>
        </CardContent>
      </Card>
    );
  }

  if (!overview) {
    return (
      <Card>
        <CardHeader>
          <CardTitle>{title}</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="text-center text-muted-foreground">
            No tournament data available
          </div>
        </CardContent>
      </Card>
    );
  }

  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
        <CardTitle className="text-sm font-medium">{title}</CardTitle>
        <Trophy className="h-4 w-4 text-muted-foreground" />
      </CardHeader>
      <CardContent>
        <div className="space-y-4">
          <div className="grid grid-cols-2 gap-4">
            <div>
              <div className="text-2xl font-bold">{overview.total}</div>
              <div className="text-xs text-muted-foreground">Total Tournaments</div>
            </div>
            <div>
              <div className="text-2xl font-bold text-green-600">{overview.active}</div>
              <div className="text-xs text-muted-foreground">Active Now</div>
            </div>
          </div>
          
          <div className="grid grid-cols-2 gap-4">
            <div className="flex items-center gap-2">
              <Users className="h-4 w-4 text-muted-foreground" />
              <div>
                <div className="text-sm font-medium">{overview.totalParticipants}</div>
                <div className="text-xs text-muted-foreground">Participants</div>
              </div>
            </div>
            <div className="flex items-center gap-2">
              <Clock className="h-4 w-4 text-muted-foreground" />
              <div>
                <div className="text-sm font-medium">{overview.upcomingCount}</div>
                <div className="text-xs text-muted-foreground">Upcoming</div>
              </div>
            </div>
          </div>

          {overview.recentTournaments.length > 0 && (
            <div>
              <div className="text-xs font-medium mb-2">Recent Tournaments</div>
              <div className="space-y-1">
                {overview.recentTournaments.slice(0, 3).map(tournament => (
                  <div key={tournament.id} className="flex items-center justify-between text-xs">
                    <span className="truncate">{tournament.name}</span>
                    <Badge variant={tournament.status === 'IN_PROGRESS' ? 'default' : 'secondary'}>
                      {tournament.status}
                    </Badge>
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

// src/app/_components/dashboard/widgets/user-statistics-widget.tsx
'use client';

import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Progress } from '@/components/ui/progress';
import { Users, UserPlus, Activity, TrendingUp } from 'lucide-react';
import { api } from '@/trpc/react';

interface UserStatisticsWidgetProps {
  title: string;
  config?: any;
  onEdit?: () => void;
  onRemove?: () => void;
}

export function UserStatisticsWidget({ title, config, onEdit, onRemove }: UserStatisticsWidgetProps) {
  const { data: stats, isLoading } = api.dashboard.getUserStatistics.useQuery();

  if (isLoading) {
    return (
      <Card>
        <CardHeader>
          <CardTitle>{title}</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="animate-pulse space-y-4">
            <div className="h-4 bg-muted rounded w-3/4"></div>
            <div className="h-4 bg-muted rounded w-1/2"></div>
          </div>
        </CardContent>
      </Card>
    );
  }

  if (!stats) {
    return (
      <Card>
        <CardHeader>
          <CardTitle>{title}</CardTitle>
        </CardHeader>
        <CardContent>
          <div className="text-center text-muted-foreground">
            No user data available
          </div>
        </CardContent>
      </Card>
    );
  }

  return (
    <Card>
      <CardHeader className="flex flex-row items-center justify-between space-y-0 pb-2">
        <CardTitle className="text-sm font-medium">{title}</CardTitle>
        <Users className="h-4 w-4 text-muted-foreground" />
      </CardHeader>
      <CardContent>
        <div className="space-y-4">
          <div className="grid grid-cols-2 gap-4">
            <div>
              <div className="text-2xl font-bold">{stats.totalUsers}</div>
              <div className="text-xs text-muted-foreground">Total Users</div>
            </div>
            <div>
              <div className="text-2xl font-bold text-green-600">{stats.activeUsers}</div>
              <div className="text-xs text-muted-foreground">Active Users</div>
            </div>
          </div>

          <div>
            <div className="flex items-center justify-between mb-2">
              <span className="text-xs font-medium">Active Rate</span>
              <span className="text-xs text-muted-foreground">
                {((stats.activeUsers / stats.totalUsers) * 100).toFixed(1)}%
              </span>
            </div>
            <Progress value={(stats.activeUsers / stats.totalUsers) * 100} className="h-2" />
          </div>

          <div className="grid grid-cols-2 gap-4">
            <div className="flex items-center gap-2">
              <UserPlus className="h-4 w-4 text-green-500" />
              <div>
                <div className="text-sm font-medium">{stats.newUsersThisWeek}</div>
                <div className="text-xs text-muted-foreground">This Week</div>
              </div>
            </div>
            <div className="flex items-center gap-2">
              <Activity className="h-4 w-4 text-blue-500" />
              <div>
                <div className="text-sm font-medium">{stats.onlineNow}</div>
                <div className="text-xs text-muted-foreground">Online Now</div>
              </div>
            </div>
          </div>

          <div>
            <div className="text-xs font-medium mb-2">User Growth</div>
            <div className="flex items-center gap-2">
              <TrendingUp className="h-4 w-4 text-green-500" />
              <span className="text-sm text-green-600">
                +{stats.growthPercentage}% this month
              </span>
            </div>
          </div>
        </div>
      </CardContent>
    </Card>
  );
}
```

#### Acceptance Criteria
- [ ] Widget grid system functional
- [ ] Real-time data updates working
- [ ] Widget customization operational
- [ ] Analytics calculations accurate
- [ ] Responsive widget layouts verified

### Phase 3: Administrative Tools (2 days)

#### Tasks
1. **Create User Management Interface**
   - User search and filtering
   - Role management and permissions
   - Bulk operations and actions

2. **Implement Tournament Administration**
   - Tournament oversight and control
   - Emergency actions and interventions
   - Data export and backup tools

#### Implementation

```typescript
// src/app/_components/dashboard/admin/user-management.tsx
'use client';

import { useState } from 'react';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Select, SelectContent, SelectItem, SelectTrigger, SelectValue } from '@/components/ui/select';
import { Badge } from '@/components/ui/badge';
import { Avatar, AvatarFallback, AvatarImage } from '@/components/ui/avatar';
import { MoreHorizontal, Search, Filter, Download, UserPlus } from 'lucide-react';
import { api } from '@/trpc/react';

export function UserManagement() {
  const [searchTerm, setSearchTerm] = useState("");
  const [roleFilter, setRoleFilter] = useState<string>("");
  const [statusFilter, setStatusFilter] = useState<string>("");
  const [page, setPage] = useState(1);

  // Get users with filtering
  const { data: usersData, isLoading } = api.dashboard.getUsers.useQuery({
    search: searchTerm,
    role: roleFilter || undefined,
    status: statusFilter || undefined,
    page,
    limit: 20,
  });

  // Bulk operations
  const updateUserRole = api.dashboard.updateUserRole.useMutation();
  const suspendUser = api.dashboard.suspendUser.useMutation();
  const exportUsers = api.dashboard.exportUsers.useMutation();

  const handleRoleChange = async (userId: string, newRole: string) => {
    try {
      await updateUserRole.mutateAsync({ userId, role: newRole as any });
    } catch (error) {
      console.error('Failed to update user role:', error);
    }
  };

  const handleSuspendUser = async (userId: string) => {
    try {
      await suspendUser.mutateAsync({ userId });
    } catch (error) {
      console.error('Failed to suspend user:', error);
    }
  };

  const handleExportUsers = async () => {
    try {
      await exportUsers.mutateAsync({
        filters: { search: searchTerm, role: roleFilter, status: statusFilter }
      });
    } catch (error) {
      console.error('Failed to export users:', error);
    }
  };

  return (
    <div className="space-y-6">
      <div className="flex items-center justify-between">
        <h1 className="text-2xl font-bold">User Management</h1>
        <div className="flex items-center gap-2">
          <Button variant="outline" onClick={handleExportUsers}>
            <Download className="w-4 h-4 mr-2" />
            Export
          </Button>
          <Button>
            <UserPlus className="w-4 h-4 mr-2" />
            Add User
          </Button>
        </div>
      </div>

      <Card>
        <CardHeader>
          <div className="flex items-center gap-4">
            <div className="relative flex-1">
              <Search className="absolute left-2 top-2.5 h-4 w-4 text-muted-foreground" />
              <Input
                placeholder="Search users..."
                value={searchTerm}
                onChange={(e) => setSearchTerm(e.target.value)}
                className="pl-8"
              />
            </div>
            
            <Select value={roleFilter} onValueChange={setRoleFilter}>
              <SelectTrigger className="w-40">
                <SelectValue placeholder="All roles" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="">All roles</SelectItem>
                <SelectItem value="ADMIN">Admin</SelectItem>
                <SelectItem value="ORGANIZER">Organizer</SelectItem>
                <SelectItem value="PARTICIPANT">Participant</SelectItem>
                <SelectItem value="VIEWER">Viewer</SelectItem>
              </SelectContent>
            </Select>

            <Select value={statusFilter} onValueChange={setStatusFilter}>
              <SelectTrigger className="w-40">
                <SelectValue placeholder="All statuses" />
              </SelectTrigger>
              <SelectContent>
                <SelectItem value="">All statuses</SelectItem>
                <SelectItem value="ACTIVE">Active</SelectItem>
                <SelectItem value="SUSPENDED">Suspended</SelectItem>
                <SelectItem value="PENDING">Pending</SelectItem>
              </SelectContent>
            </Select>
          </div>
        </CardHeader>

        <CardContent>
          {isLoading ? (
            <div className="text-center py-8">Loading users...</div>
          ) : (
            <Table>
              <TableHeader>
                <TableRow>
                  <TableHead>User</TableHead>
                  <TableHead>Role</TableHead>
                  <TableHead>Status</TableHead>
                  <TableHead>Last Login</TableHead>
                  <TableHead>Tournaments</TableHead>
                  <TableHead className="text-right">Actions</TableHead>
                </TableRow>
              </TableHeader>
              <TableBody>
                {usersData?.users.map(user => (
                  <TableRow key={user.id}>
                    <TableCell>
                      <div className="flex items-center gap-3">
                        <Avatar className="h-8 w-8">
                          <AvatarImage src={user.image || undefined} />
                          <AvatarFallback>
                            {user.name?.charAt(0) || user.email.charAt(0)}
                          </AvatarFallback>
                        </Avatar>
                        <div>
                          <div className="font-medium">{user.name || 'No name'}</div>
                          <div className="text-sm text-muted-foreground">{user.email}</div>
                        </div>
                      </div>
                    </TableCell>
                    
                    <TableCell>
                      <Select 
                        value={user.role} 
                        onValueChange={(value) => handleRoleChange(user.id, value)}
                      >
                        <SelectTrigger className="w-32">
                          <SelectValue />
                        </SelectTrigger>
                        <SelectContent>
                          <SelectItem value="ADMIN">Admin</SelectItem>
                          <SelectItem value="ORGANIZER">Organizer</SelectItem>
                          <SelectItem value="PARTICIPANT">Participant</SelectItem>
                          <SelectItem value="VIEWER">Viewer</SelectItem>
                        </SelectContent>
                      </Select>
                    </TableCell>
                    
                    <TableCell>
                      <Badge variant={user.status === 'ACTIVE' ? 'default' : 'destructive'}>
                        {user.status}
                      </Badge>
                    </TableCell>
                    
                    <TableCell>
                      {user.lastLogin 
                        ? new Date(user.lastLogin).toLocaleDateString()
                        : 'Never'
                      }
                    </TableCell>
                    
                    <TableCell>
                      {user.tournamentCount || 0}
                    </TableCell>
                    
                    <TableCell className="text-right">
                      <Button variant="ghost" size="icon">
                        <MoreHorizontal className="h-4 w-4" />
                      </Button>
                    </TableCell>
                  </TableRow>
                ))}
              </TableBody>
            </Table>
          )}

          {usersData && (
            <div className="flex items-center justify-between mt-4">
              <div className="text-sm text-muted-foreground">
                Showing {((page - 1) * 20) + 1} to {Math.min(page * 20, usersData.total)} of {usersData.total} users
              </div>
              <div className="flex items-center gap-2">
                <Button
                  variant="outline"
                  size="sm"
                  disabled={page === 1}
                  onClick={() => setPage(page - 1)}
                >
                  Previous
                </Button>
                <Button
                  variant="outline"
                  size="sm"
                  disabled={page >= Math.ceil(usersData.total / 20)}
                  onClick={() => setPage(page + 1)}
                >
                  Next
                </Button>
              </div>
            </div>
          )}
        </CardContent>
      </Card>
    </div>
  );
}
```

#### Acceptance Criteria
- [ ] User management interface functional
- [ ] Role assignment working correctly
- [ ] Bulk operations operational
- [ ] Data export functionality working
- [ ] Permission controls enforced

### Phase 4: System Monitoring & Notifications (2 days)

#### Tasks
1. **Create System Health Dashboard**
   - Performance metrics monitoring
   - Error tracking and alerts
   - Usage statistics and trends

2. **Implement Notification System**
   - System-wide announcements
   - Alert management and delivery
   - User notification preferences

#### Implementation

```typescript
// src/server/api/routers/dashboard.ts
export const dashboardRouter = createTRPCRouter({
  // Get dashboard configuration
  getConfig: protectedProcedure
    .query(async ({ ctx }) => {
      const config = await ctx.db.user.findUnique({
        where: { id: ctx.session.user.id },
        select: { dashboardConfig: true },
      });

      return config?.dashboardConfig || getDefaultDashboardConfig();
    }),

  // Get dashboard widgets
  getWidgets: protectedProcedure
    .query(async ({ ctx }) => {
      const widgets = await ctx.db.dashboardWidget.findMany({
        where: { userId: ctx.session.user.id },
        orderBy: { position: 'asc' },
      });

      if (widgets.length === 0) {
        return getDefaultWidgets(ctx.session.user.role);
      }

      return widgets;
    }),

  // Save widget layout
  saveWidgetLayout: protectedProcedure
    .input(z.object({
      widgets: z.array(z.object({
        id: z.string(),
        position: z.object({
          x: z.number(),
          y: z.number(),
          w: z.number(),
          h: z.number(),
        }),
      })),
    }))
    .mutation(async ({ ctx, input }) => {
      await ctx.db.$transaction(
        input.widgets.map(widget =>
          ctx.db.dashboardWidget.update({
            where: { id: widget.id },
            data: { position: widget.position },
          })
        )
      );

      return { success: true };
    }),

  // Get tournament overview
  getTournamentOverview: protectedProcedure
    .query(async ({ ctx }) => {
      const [
        total,
        active,
        upcoming,
        completed,
        totalParticipants,
        recentTournaments,
      ] = await Promise.all([
        ctx.db.tournament.count(),
        ctx.db.tournament.count({ where: { status: 'IN_PROGRESS' } }),
        ctx.db.tournament.count({ where: { status: 'REGISTRATION_OPEN' } }),
        ctx.db.tournament.count({ where: { status: 'COMPLETED' } }),
        ctx.db.tournamentParticipant.count(),
        ctx.db.tournament.findMany({
          take: 5,
          orderBy: { createdAt: 'desc' },
          select: {
            id: true,
            name: true,
            status: true,
            startDate: true,
            _count: {
              select: { participants: true },
            },
          },
        }),
      ]);

      return {
        total,
        active,
        upcoming: upcoming,
        completed,
        totalParticipants,
        upcomingCount: upcoming,
        recentTournaments,
      };
    }),

  // Get user statistics
  getUserStatistics: protectedProcedure
    .query(async ({ ctx }) => {
      const now = new Date();
      const weekAgo = new Date(now.getTime() - 7 * 24 * 60 * 60 * 1000);
      const monthAgo = new Date(now.getTime() - 30 * 24 * 60 * 60 * 1000);

      const [
        totalUsers,
        activeUsers,
        newUsersThisWeek,
        usersLastMonth,
        onlineNow,
      ] = await Promise.all([
        ctx.db.user.count(),
        ctx.db.user.count({
          where: {
            lastLogin: { gte: weekAgo },
          },
        }),
        ctx.db.user.count({
          where: {
            createdAt: { gte: weekAgo },
          },
        }),
        ctx.db.user.count({
          where: {
            createdAt: { gte: monthAgo },
          },
        }),
        // Mock online count - in real app, track active sessions
        Math.floor(Math.random() * 50) + 10,
      ]);

      const growthPercentage = usersLastMonth > 0 
        ? ((totalUsers - usersLastMonth) / usersLastMonth * 100).toFixed(1)
        : '0';

      return {
        totalUsers,
        activeUsers,
        newUsersThisWeek,
        onlineNow,
        growthPercentage,
      };
    }),

  // Get system notifications
  getNotifications: protectedProcedure
    .query(async ({ ctx }) => {
      const notifications = await ctx.db.systemNotification.findMany({
        where: {
          isActive: true,
          OR: [
            { targetRoles: { has: ctx.session.user.role } },
            { targetUsers: { has: ctx.session.user.id } },
          ],
          AND: [
            {
              OR: [
                { showUntil: null },
                { showUntil: { gte: new Date() } },
              ],
            },
          ],
        },
        orderBy: [
          { priority: 'desc' },
          { createdAt: 'desc' },
        ],
        include: {
          creator: {
            select: { name: true },
          },
        },
      });

      return notifications;
    }),

  // Get users for management
  getUsers: protectedProcedure
    .input(z.object({
      search: z.string().optional(),
      role: z.enum(['ADMIN', 'ORGANIZER', 'PARTICIPANT', 'VIEWER']).optional(),
      status: z.string().optional(),
      page: z.number().default(1),
      limit: z.number().default(20),
    }))
    .query(async ({ ctx, input }) => {
      // Check admin permissions
      if (ctx.session.user.role !== 'ADMIN') {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'Admin access required',
        });
      }

      const { search, role, status, page, limit } = input;
      const offset = (page - 1) * limit;

      const where: any = {};

      if (search) {
        where.OR = [
          { name: { contains: search, mode: 'insensitive' } },
          { email: { contains: search, mode: 'insensitive' } },
        ];
      }

      if (role) {
        where.role = role;
      }

      if (status) {
        where.status = status;
      }

      const [users, total] = await Promise.all([
        ctx.db.user.findMany({
          where,
          skip: offset,
          take: limit,
          orderBy: { createdAt: 'desc' },
          select: {
            id: true,
            name: true,
            email: true,
            role: true,
            image: true,
            lastLogin: true,
            createdAt: true,
            _count: {
              select: { tournaments: true },
            },
          },
        }),
        ctx.db.user.count({ where }),
      ]);

      return {
        users: users.map(user => ({
          ...user,
          status: 'ACTIVE', // Mock status - implement proper user status
          tournamentCount: user._count.tournaments,
        })),
        total,
        page,
        totalPages: Math.ceil(total / limit),
      };
    }),

  // Update user role
  updateUserRole: protectedProcedure
    .input(z.object({
      userId: z.string(),
      role: z.enum(['ADMIN', 'ORGANIZER', 'PARTICIPANT', 'VIEWER']),
    }))
    .mutation(async ({ ctx, input }) => {
      // Check admin permissions
      if (ctx.session.user.role !== 'ADMIN') {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'Admin access required',
        });
      }

      const updatedUser = await ctx.db.user.update({
        where: { id: input.userId },
        data: { role: input.role },
      });

      // Log the action
      await ctx.db.auditLog.create({
        data: {
          userId: ctx.session.user.id,
          action: 'UPDATE_USER_ROLE',
          entityType: 'USER',
          entityId: input.userId,
          details: {
            newRole: input.role,
            targetUserId: input.userId,
          },
          severity: 'INFO',
        },
      });

      return updatedUser;
    }),

  // Get system health metrics
  getSystemHealth: protectedProcedure
    .query(async ({ ctx }) => {
      // Check admin permissions
      if (ctx.session.user.role !== 'ADMIN') {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'Admin access required',
        });
      }

      const [
        totalTournaments,
        activeTournaments,
        totalUsers,
        totalGames,
        errorCount,
      ] = await Promise.all([
        ctx.db.tournament.count(),
        ctx.db.tournament.count({ where: { status: 'IN_PROGRESS' } }),
        ctx.db.user.count(),
        ctx.db.game.count(),
        ctx.db.auditLog.count({
          where: {
            severity: { in: ['ERROR', 'CRITICAL'] },
            timestamp: { gte: new Date(Date.now() - 24 * 60 * 60 * 1000) },
          },
        }),
      ]);

      // Mock performance metrics - in real app, get from monitoring service
      const metrics = {
        cpuUsage: Math.random() * 100,
        memoryUsage: Math.random() * 100,
        diskUsage: Math.random() * 100,
        responseTime: Math.random() * 500 + 100,
        uptime: Date.now() - (Math.random() * 7 * 24 * 60 * 60 * 1000),
      };

      return {
        overview: {
          totalTournaments,
          activeTournaments,
          totalUsers,
          totalGames,
          errorCount,
        },
        performance: metrics,
        status: errorCount > 10 ? 'WARNING' : 'HEALTHY',
      };
    }),

  // Create system notification
  createNotification: protectedProcedure
    .input(z.object({
      title: z.string(),
      message: z.string(),
      type: z.enum(['INFO', 'SUCCESS', 'WARNING', 'ERROR', 'ANNOUNCEMENT']),
      priority: z.enum(['LOW', 'NORMAL', 'HIGH', 'CRITICAL']).default('NORMAL'),
      targetRoles: z.array(z.enum(['ADMIN', 'ORGANIZER', 'PARTICIPANT', 'VIEWER'])),
      targetUsers: z.array(z.string()).optional(),
      showUntil: z.date().optional(),
    }))
    .mutation(async ({ ctx, input }) => {
      // Check admin permissions
      if (ctx.session.user.role !== 'ADMIN') {
        throw new TRPCError({
          code: 'FORBIDDEN',
          message: 'Admin access required',
        });
      }

      const notification = await ctx.db.systemNotification.create({
        data: {
          ...input,
          targetUsers: input.targetUsers || [],
          createdBy: ctx.session.user.id,
        },
      });

      return notification;
    }),
});

// Helper functions
function getDefaultDashboardConfig() {
  return {
    theme: 'light',
    refreshInterval: 30,
    showNotifications: true,
    compactMode: false,
  };
}

function getDefaultWidgets(userRole: string) {
  const commonWidgets = [
    {
      id: 'tournament-overview',
      type: 'TOURNAMENT_OVERVIEW',
      title: 'Tournament Overview',
      position: { x: 0, y: 0, w: 6, h: 3 },
    },
    {
      id: 'recent-activity',
      type: 'RECENT_ACTIVITY',
      title: 'Recent Activity',
      position: { x: 6, y: 0, w: 6, h: 3 },
    },
  ];

  if (userRole === 'ADMIN') {
    return [
      ...commonWidgets,
      {
        id: 'user-statistics',
        type: 'USER_STATISTICS',
        title: 'User Statistics',
        position: { x: 0, y: 3, w: 6, h: 3 },
      },
      {
        id: 'system-health',
        type: 'SYSTEM_HEALTH',
        title: 'System Health',
        position: { x: 6, y: 3, w: 6, h: 3 },
      },
    ];
  }

  return [
    ...commonWidgets,
    {
      id: 'active-games',
      type: 'ACTIVE_GAMES',
      title: 'Active Games',
      position: { x: 0, y: 3, w: 12, h: 3 },
    },
  ];
}
```

#### Acceptance Criteria
- [ ] System health monitoring functional
- [ ] Performance metrics tracking accurate
- [ ] Notification system operational
- [ ] Alert delivery working correctly
- [ ] Administrative controls secure

### Phase 5: Reports & Analytics (1 day)

#### Tasks
1. **Create Reporting Interface**
   - Pre-built report templates
   - Custom report builder
   - Data export capabilities

2. **Implement Analytics Dashboard**
   - Tournament performance analytics
   - User engagement metrics
   - Platform usage insights

#### Acceptance Criteria
- [ ] Reporting system functional
- [ ] Analytics calculations accurate
- [ ] Data export working correctly
- [ ] Custom reports operational
- [ ] Performance insights valuable

## Risk Assessment

### High Risk
- **Performance complexity**: Dashboard with multiple real-time widgets
- **Permission management**: Secure role-based access across all features
- **Data consistency**: Ensuring dashboard shows accurate real-time data

### Medium Risk
- **Widget customization**: User-configurable dashboard layouts
- **Notification delivery**: Reliable system-wide messaging

### Low Risk
- **UI/UX patterns**: Standard dashboard design patterns
- **Data visualization**: Using established charting libraries

## Success Metrics

- Dashboard load time: <3 seconds
- Widget refresh rate: <30 seconds for real-time data
- Administrative task completion: <5 clicks for common operations
- System health monitoring: 99.9% uptime tracking
- 90% user satisfaction with dashboard usability

## Dependencies for Next Phase

This dashboard system enables:
- Centralized platform administration
- Real-time system monitoring
- Advanced analytics and reporting
- User and tournament management at scale
