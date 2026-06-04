---
name: react-vite-best-practices
description: Guide for developing scalable Client-Side Rendering (CSR) React applications using Vite. Use for setting up a new React+Vite project, organizing folder structure by features, configuring API layers and lib abstractions, managing state (Zustand/TanStack Query), and applying modern best practices.
---

# React + Vite Best Practices and Architecture

Comprehensive guidelines for building, structuring, and maintaining production-grade React applications using Vite as the build tool. `datagol-app-development` is the source of truth for *what* to build in a DataGOL app — this skill is the source of truth for *how* to structure the code.

## 1. Project Initialization & Tooling Setup

Vite has replaced Create React App as the modern standard for building Single Page Applications (SPAs).

### Scaffold and Build Configuration

- **Init**: `npm create vite@latest my-app -- --template react-ts` for a fresh project. Inside the codex sandbox the template is already in place — don't re-init, edit `src/` files directly.
- **SWC Plugin**: Prefer `@vitejs/plugin-react-swc` over the default Babel plugin for significantly faster development builds.
- **Path Aliases**: Configure path aliases (e.g., `@/components/*`) in both `tsconfig.json` (`compilerOptions.paths`) and `vite.config.ts` (`resolve.alias`) to avoid messy relative imports.

### Code Quality Tooling

- **TypeScript**: Enforce strict type-checking (`"strict": true`).
- **ESLint & Prettier**: Use ESLint to catch logical errors and Prettier for code formatting.
- **Husky & Lint-Staged**: Pre-commit hooks to run linting and formatting automatically before code is committed.

## 2. Scalable Folder Structure

A **feature-based** (domain-driven) folder structure is critical for maintainability as the application scales. Avoid dumping everything into global `components/` or `hooks/` folders.

```text
src/
├── assets/               # Static assets (images, global CSS, fonts)
├── components/           # ONLY global, shared UI components (e.g., Button, Modal, Input)
├── config/               # Global configuration and environment variables
├── constants/            # Application-wide constants (e.g., route paths, query keys)
├── features/             # Feature-specific modules — the core of the app
│   ├── auth/             # Example feature: Authentication
│   │   ├── components/   # Auth-specific components (e.g., LoginForm)
│   │   ├── hooks/        # Auth-specific hooks
│   │   ├── services/     # API calls for auth
│   │   ├── store/        # Auth-specific state
│   │   └── types.ts      # TypeScript types for auth
│   └── dashboard/        # Example feature: Dashboard
├── hooks/                # Global custom hooks (e.g., useLocalStorage, useMediaQuery)
├── layouts/              # Shared layout components (e.g., MainLayout, AuthLayout)
├── lib/                  # Third-party library initializations and wrappers
├── pages/                # Route-level components that compose features together
├── services/             # Global API clients and base configurations
├── store/                # Global state stores (if applicable)
├── types/                # Global TypeScript definitions
└── utils/                # Pure utility functions (formatters, validators, etc.)
```

## 3. API Layer and Library Abstraction

To build a resilient architecture, decouple your application's business logic from third-party libraries and raw network requests.

### The `lib/` Folder Pattern

Never initialize or configure third-party libraries directly inside components. Instead, wrap them in the `lib/` directory and export the initialized instances.

- **Example (`src/lib/axios.ts`)**: Create and export an Axios instance with base URL and default headers.
- **Example (`src/lib/queryClient.ts`)**: Initialize the TanStack Query client with global default options (e.g., `staleTime`, retry logic).

**Benefit**: If you ever need to swap out a library (e.g., moving from Axios to native fetch), you only update the `lib/` file, not your entire component tree.

### API Service Layer Architecture

Follow a strict data flow pattern: **UI Component → Custom Hook (React Query) → Service Layer → API Client**.

1. **API Client (`lib/axios.ts`)**: Base configuration.
2. **Interceptors**: Use Axios interceptors to centralize authentication (attaching Bearer tokens) and global error handling (e.g., redirecting to login on 401 Unauthorized).
3. **Service Layer (`services/` or `features/*/services/`)**: Define plain asynchronous functions that execute the HTTP requests (e.g., `getUsers()`, `createOrder(data)`).
4. **Hooks Layer (`hooks/` or `features/*/hooks/`)**: Wrap the service functions in TanStack Query hooks (`useQuery`, `useMutation`). Components should only call these hooks, never the service functions directly.

### Query Key Factories

To manage TanStack Query cache effectively, organize query keys using a factory pattern in your `constants/` or feature directories. This prevents typos and makes cache invalidation reliable.

```ts
// src/features/todos/constants/queryKeys.ts
export const todoKeys = {
  all: ['todos'] as const,
  lists: () => [...todoKeys.all, 'list'] as const,
  list: (filters: string) => [...todoKeys.lists(), { filters }] as const,
  details: () => [...todoKeys.all, 'detail'] as const,
  detail: (id: number) => [...todoKeys.details(), id] as const,
};
```

## 4. State Management

Modern React state management relies on **separating server state from client state**.

### Server State (Data Fetching & Caching)

- **TanStack Query (React Query)**: Use this for all asynchronous operations, data fetching, caching, and synchronization. Do **not** use `useEffect` and `useState` for fetching data.

### Client State (Local UI State)

- **Local State**: Use `useState` and `useReducer` for state that is isolated to a single component (e.g., a dropdown open/close state).
- **Global Client State**: Use **Zustand** for lightweight, un-opinionated global state management (e.g., user preferences, theme, complex multi-step form data). Avoid Redux unless working on a legacy enterprise codebase.

## 5. Performance Optimization

### Code Splitting & Lazy Loading

- Use `React.lazy` and `<Suspense>` to code-split routes at the page level. This ensures users only download the JavaScript needed for the page they are viewing.
- Configure Vite's `manualChunks` in `vite.config.ts` to separate vendor libraries (like `react` and `react-dom`) from application code.

### Rendering Optimization

- **Measure First**: Do not prematurely optimize. Use React DevTools Profiler to identify actual bottlenecks.
- **`useMemo` & `useCallback`**: Use these hooks specifically when passing objects/functions as props to memoized child components, or when a value is used as a dependency in another hook (like `useEffect`).
- **Push State Down**: Move state as close to where it's needed as possible to prevent unnecessary re-renders of parent components.

## 6. Testing Strategy

- **Vitest**: Use Vitest instead of Jest. It uses the same configuration as Vite, supports ES modules out of the box, and is significantly faster.
- **React Testing Library (RTL)**: Use RTL for component testing. Focus on testing user behavior rather than implementation details.

## 7. Naming Conventions

- **Components**: PascalCase (e.g., `UserProfile.tsx`).
- **Hooks**: camelCase, prefixed with `use` (e.g., `useAuth.ts`).
- **Utility files / Services**: camelCase or kebab-case (e.g., `formatDate.ts`, `api-client.ts`).
