---
name: react-frontend
description: "Complete React frontend skill: component patterns, hooks, TanStack Query, Zustand, forms, routing, and UI best practices. Load for any React/frontend work."
---

# React & Frontend Mastery — إتقان React

## Component Architecture

```typescript
// ✅ Good component structure
// 1. Imports (grouped)
import { useState, useCallback } from 'react'
import { useNavigate } from 'react-router-dom'
import { useMutation } from '@tanstack/react-query'
import { useForm } from 'react-hook-form'
import { zodResolver } from '@hookform/resolvers/zod'
import { z } from 'zod/v4'

// 2. Types
interface CreatePostProps {
  onSuccess?: () => void
}

// 3. Schema (if form)
const CreatePostSchema = z.object({
  title: z.string().min(1, 'Title is required').max(200),
  content: z.string().min(10, 'Content must be at least 10 characters'),
  published: z.boolean().default(false)
})

type CreatePostInput = z.infer<typeof CreatePostSchema>

// 4. Component
export function CreatePost({ onSuccess }: CreatePostProps) {
  const navigate = useNavigate()
  
  const { register, handleSubmit, formState: { errors } } = useForm<CreatePostInput>({
    resolver: zodResolver(CreatePostSchema)
  })
  
  const { mutate: createPost, isPending, error } = useMutation({
    mutationFn: (data: CreatePostInput) => api.post('/posts', data),
    onSuccess: (post) => {
      onSuccess?.()
      navigate(`/posts/${post.id}`)
    }
  })
  
  return (
    <form onSubmit={handleSubmit((data) => createPost(data))}>
      <Input
        label="Title"
        {...register('title')}
        error={errors.title?.message}
      />
      <Textarea
        label="Content"
        {...register('content')}
        error={errors.content?.message}
      />
      <Button type="submit" loading={isPending}>
        {isPending ? 'Creating...' : 'Create Post'}
      </Button>
      {error && <Alert variant="error">{error.message}</Alert>}
    </form>
  )
}
```

## TanStack Query Patterns

```typescript
// hooks/usePosts.ts

// List query
export function usePosts(params?: { page?: number; limit?: number }) {
  return useQuery({
    queryKey: ['posts', params],
    queryFn: () => api.get('/posts', { params }).then(r => r.data),
    staleTime: 30 * 1000  // 30 seconds
  })
}

// Single item query
export function usePost(id: string) {
  return useQuery({
    queryKey: ['posts', id],
    queryFn: () => api.get(`/posts/${id}`).then(r => r.data),
    enabled: Boolean(id)
  })
}

// Mutation with optimistic update
export function useUpdatePost() {
  const queryClient = useQueryClient()
  
  return useMutation({
    mutationFn: ({ id, ...data }: UpdatePostInput) =>
      api.patch(`/posts/${id}`, data).then(r => r.data),
    
    onMutate: async ({ id, ...data }) => {
      await queryClient.cancelQueries({ queryKey: ['posts', id] })
      const previous = queryClient.getQueryData(['posts', id])
      queryClient.setQueryData(['posts', id], (old: Post) => ({ ...old, ...data }))
      return { previous }
    },
    
    onError: (err, { id }, context) => {
      queryClient.setQueryData(['posts', id], context?.previous)
    },
    
    onSettled: (data, error, { id }) => {
      queryClient.invalidateQueries({ queryKey: ['posts', id] })
      queryClient.invalidateQueries({ queryKey: ['posts'] })
    }
  })
}
```

## Zustand State Management

```typescript
// store/uiStore.ts — for UI-only state
import { create } from 'zustand'

interface UIState {
  sidebarOpen: boolean
  theme: 'light' | 'dark'
  toggleSidebar: () => void
  setTheme: (theme: 'light' | 'dark') => void
}

export const useUIStore = create<UIState>()((set) => ({
  sidebarOpen: false,
  theme: 'light',
  toggleSidebar: () => set(s => ({ sidebarOpen: !s.sidebarOpen })),
  setTheme: (theme) => set({ theme })
}))

// Rule: Use TanStack Query for server state, Zustand for UI state only
// ❌ Don't store API data in Zustand — that's what TanStack Query is for
// ✅ Store: sidebar state, modal state, theme, filters, selected items
```

## Error Boundaries

```tsx
// components/ErrorBoundary.tsx
import { Component, ReactNode } from 'react'

interface Props {
  children: ReactNode
  fallback?: (error: Error, reset: () => void) => ReactNode
}

interface State {
  error: Error | null
}

export class ErrorBoundary extends Component<Props, State> {
  state: State = { error: null }
  
  static getDerivedStateFromError(error: Error): State {
    return { error }
  }
  
  componentDidCatch(error: Error, info: React.ErrorInfo) {
    console.error('Unhandled error:', error, info)
    // Send to error tracking
    reportError(error, info)
  }
  
  render() {
    if (this.state.error) {
      return this.props.fallback?.(this.state.error, () => this.setState({ error: null })) ?? (
        <div className="error-page">
          <h2>Something went wrong</h2>
          <button onClick={() => this.setState({ error: null })}>Try again</button>
        </div>
      )
    }
    return this.props.children
  }
}

// Usage
function App() {
  return (
    <ErrorBoundary fallback={(error, reset) => (
      <div>Error: {error.message} <button onClick={reset}>Retry</button></div>
    )}>
      <Router />
    </ErrorBoundary>
  )
}
```

## Loading States Pattern

```tsx
// Always handle all states explicitly
function PostList() {
  const { data, isLoading, error, isError } = usePosts()
  
  if (isLoading) return <PostListSkeleton />
  if (isError) return <ErrorMessage error={error} retry={() => refetch()} />
  if (!data?.length) return <EmptyState message="No posts yet" action={<CreatePostButton />} />
  
  return (
    <div className="grid gap-4">
      {data.map(post => <PostCard key={post.id} post={post} />)}
    </div>
  )
}

// Skeleton loader
function PostListSkeleton() {
  return (
    <div className="grid gap-4">
      {Array.from({ length: 3 }).map((_, i) => (
        <div key={i} className="animate-pulse">
          <div className="h-6 bg-gray-200 rounded w-3/4 mb-2" />
          <div className="h-4 bg-gray-200 rounded w-full mb-1" />
          <div className="h-4 bg-gray-200 rounded w-2/3" />
        </div>
      ))}
    </div>
  )
}
```

## Performance Best Practices

```typescript
// 1. Memoize expensive computations
const sortedItems = useMemo(
  () => items.sort((a, b) => b.createdAt - a.createdAt),
  [items]
)

// 2. Stable callbacks
const handleClick = useCallback((id: string) => {
  doSomething(id)
}, [doSomething])

// 3. Virtualize long lists
import { useVirtualizer } from '@tanstack/react-virtual'

// 4. Code split routes
const Dashboard = lazy(() => import('./pages/Dashboard'))
const Settings = lazy(() => import('./pages/Settings'))

// 5. Optimize images
<img
  src={url}
  loading="lazy"
  decoding="async"
  width={400}
  height={300}
/>
```
