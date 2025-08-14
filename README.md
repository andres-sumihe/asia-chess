# Chess Community Platform 🏆

A comprehensive tournament management platform built for chess communities, enabling organizers to manage participants, track scores, maintain standings, and provide administrative oversight through a modern web interface.

## 🚀 Built With Modern Tech Stack

This project is built using the [T3 Stack](https://create.t3.gg/) and modern web technologies:

- **[Next.js 15](https://nextjs.org)** - React framework with App Router
- **[React 19](https://react.dev)** - UI library with concurrent features
- **[TypeScript](https://typescriptlang.org)** - Type-safe development
- **[NextAuth.js](https://next-auth.js.org)** - Authentication solution
- **[Prisma](https://prisma.io)** - Database ORM with PostgreSQL
- **[tRPC](https://trpc.io)** - End-to-end typesafe APIs
- **[Tailwind CSS](https://tailwindcss.com)** - Utility-first styling
- **[shadcn/ui](https://ui.shadcn.com)** - Beautiful UI components

## 🌟 Features

- **Tournament Management** - Create and manage chess tournaments with various formats
- **Participant Registration** - Easy sign-up and profile management for players
- **Real-time Scoreboard** - Live game tracking and result updates
- **Standings Calculation** - Automatic ranking with tiebreak systems
- **Administrative Dashboard** - Comprehensive management interface
- **Real-time Updates** - WebSocket integration for live data
- **Analytics & Reports** - Tournament insights and player statistics
- **Role-based Access** - Admin, Organizer, Participant, and Viewer roles

## 🤝 Contributing to Chess Community Platform

We welcome contributions from the chess and developer community! This guide will help you get started with contributing to the project.

### 📋 Prerequisites

Before you begin, ensure you have the following installed:

- **Node.js** (v18 or higher) - [Download here](https://nodejs.org/)
- **pnpm** (recommended) or npm - [Install pnpm](https://pnpm.io/installation)
- **Git** - [Download here](https://git-scm.com/)
- **PostgreSQL** (for local development) - [Download here](https://postgresql.org/)

### 🔧 Getting Started

#### 1. Fork the Repository

1. Navigate to the [Chess Community Platform repository](https://github.com/your-org/asia-chess)
2. Click the **"Fork"** button in the top-right corner
3. Select your GitHub account to create your fork

#### 2. Clone Your Fork

```bash
# Clone your forked repository
git clone https://github.com/YOUR_USERNAME/asia-chess.git

# Navigate to the project directory
cd asia-chess

# Add the original repository as upstream
git remote add upstream https://github.com/your-org/asia-chess.git
```

#### 3. Set Up Development Environment

```bash
# Install dependencies
pnpm install

# Copy environment variables
cp .env.example .env

# Set up your local database
# Edit .env with your PostgreSQL connection string
# DATABASE_URL="postgresql://username:password@localhost:5432/chess_platform"

# Generate Prisma client and run migrations
pnpm db:generate
pnpm db:migrate
pnpm db:seed
```

#### 4. Start Development Server

```bash
# Start the development server
pnpm dev

# Your app will be available at http://localhost:3000
```

### 🌿 Branch Strategy

Our repository uses a protected `main` branch. All contributions must go through pull requests.

#### Creating a Feature Branch

```bash
# Ensure you're on main and up to date
git checkout main
git pull upstream main

# Create a new feature branch
git checkout -b feature/your-feature-name

# Or for bug fixes
git checkout -b fix/issue-description

# Or for documentation
git checkout -b docs/what-youre-documenting
```

#### Branch Naming Conventions

- **Features**: `feature/tournament-management`, `feature/real-time-updates`
- **Bug Fixes**: `fix/scoreboard-calculation`, `fix/user-authentication`
- **Documentation**: `docs/api-documentation`, `docs/setup-guide`
- **Refactoring**: `refactor/database-schema`, `refactor/ui-components`

### 📝 Making Changes

#### Development Workflow

1. **Choose an Issue**: Look for issues labeled `good first issue` or `help wanted`
2. **Assign Yourself**: Comment on the issue to let others know you're working on it
3. **Follow Implementation Plans**: Check the `/plan` directory for detailed specifications
4. **Write Tests**: Add unit and integration tests for new features
5. **Update Documentation**: Update relevant docs and comments

#### Code Quality Standards

```bash
# Before committing, ensure your code passes all checks
pnpm lint          # ESLint checks
pnpm type-check    # TypeScript compilation
pnpm test          # Run test suite
pnpm format        # Prettier formatting
```

#### Commit Message Guidelines

We follow [Conventional Commits](https://conventionalcommits.org/):

```bash
# Format: type(scope): description

# Examples:
git commit -m "feat(tournament): add Swiss system pairing algorithm"
git commit -m "fix(auth): resolve session timeout issue"
git commit -m "docs(api): update tRPC procedure documentation"
git commit -m "test(scoreboard): add unit tests for score calculation"
```

### 🔄 Submitting Your Contribution

#### 1. Push Your Changes

```bash
# Push your feature branch to your fork
git push origin feature/your-feature-name
```

#### 2. Create a Pull Request

1. Go to your fork on GitHub
2. Click **"New Pull Request"**
3. Select `main` as the base branch and your feature branch as compare
4. Fill out the pull request template:

```markdown
## Description
Brief description of what this PR does

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] Documentation update

## Implementation Plan Reference
- Related to: plan/plan-[feature-name].md
- Implements: [Phase X] of the implementation plan

## Testing
- [ ] I have added tests that prove my fix is effective or that my feature works
- [ ] New and existing unit tests pass locally with my changes
- [ ] I have tested the changes in a local development environment

## Screenshots (if applicable)
Add screenshots to help explain your changes

## Checklist
- [ ] My code follows the project's style guidelines
- [ ] I have performed a self-review of my own code
- [ ] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation
- [ ] My changes generate no new warnings
```

#### 3. Code Review Process

- **Automated Checks**: GitHub Actions will run tests, linting, and type checking
- **Peer Review**: At least one maintainer will review your code
- **Feedback**: Address any requested changes promptly
- **Approval**: Once approved, your PR will be merged into `main`

### 🏗️ Project Structure

```
asia-chess/
├── plan/                    # Implementation plans and specifications
├── spec/                    # Design specifications
├── src/
│   ├── app/                # Next.js App Router pages and layouts
│   │   ├── _components/    # Reusable React components
│   │   └── api/           # API routes
│   ├── server/            # Backend logic
│   │   ├── api/           # tRPC routers and procedures
│   │   ├── auth/          # Authentication configuration
│   │   └── db.ts          # Database client
│   ├── styles/            # Global styles and CSS
│   └── trpc/              # tRPC client configuration
├── prisma/                # Database schema and migrations
├── public/                # Static assets
└── tests/                 # Test files
```

### 🧪 Testing Guidelines

#### Running Tests

```bash
# Run all tests
pnpm test

# Run tests in watch mode
pnpm test:watch

# Run tests with coverage
pnpm test:coverage

# Run specific test files
pnpm test tournament.test.ts
```

#### Writing Tests

- **Unit Tests**: Test individual functions and components
- **Integration Tests**: Test API routes and database operations
- **E2E Tests**: Test complete user workflows

```typescript
// Example test structure
describe('Tournament Management', () => {
  it('should create a new tournament', async () => {
    // Test implementation
  });

  it('should calculate standings correctly', async () => {
    // Test implementation
  });
});
```

### 📚 Implementation Plans

Before starting development, review the relevant implementation plans in the `/plan` directory:

1. **Authentication System** - `plan/plan-auth-system.md`
2. **Tournament Management** - `plan/plan-tournament-management.md`
3. **Real-time Updates** - `plan/plan-realtime-system.md`
4. **Analytics & Reports** - `plan/plan-reports-system.md`

Each plan contains:
- Feature specifications
- Database schema changes
- API implementation details
- Frontend component requirements
- Testing requirements

### 🔍 Finding Issues to Work On

#### Labels to Look For

- `good first issue` - Perfect for new contributors
- `help wanted` - Community help needed
- `bug` - Bug fixes needed
- `enhancement` - New features to implement
- `documentation` - Documentation improvements

#### Issue Templates

When creating new issues, use our templates:
- **Bug Report**: For reporting bugs
- **Feature Request**: For suggesting new features
- **Documentation**: For documentation improvements

### 💬 Community & Support

#### Getting Help

- **GitHub Discussions**: Ask questions and share ideas
- **Discord Server**: Real-time chat with maintainers and contributors
- **Issue Comments**: Get help on specific issues

#### Code of Conduct

Please read and follow our [Code of Conduct](CODE_OF_CONDUCT.md) to ensure a welcoming environment for all contributors.

### 🏆 Recognition

Contributors will be recognized in:
- **README Contributors Section**: Listed on the main README
- **Release Notes**: Mentioned in version releases
- **Hall of Fame**: Special recognition for significant contributions

## 📄 License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## 🙏 Acknowledgments

- Built with [T3 Stack](https://create.t3.gg/) - The best way to start a full-stack TypeScript app
- UI components from [shadcn/ui](https://ui.shadcn.com) - Beautiful and accessible components
- Chess community for inspiration and feedback

---

**Ready to contribute?** Fork the repo, make your changes, and submit a pull request! We can't wait to see what you build. 🚀
