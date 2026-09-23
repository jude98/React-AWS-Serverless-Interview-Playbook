# TanStack Query Server State and Stale While Revalidate Patterns

> [!abstract] Fundamental Paradigm Shift
> 
> React Query (TanStack Query) shifts state management from **Client State** (UI toggles, form inputs, local state) to **Server State** (data stored on a remote server, asynchronously fetched, out-of-band shared, and inherently out-of-date). It eliminates boilerplate `useEffect` + `useState` fetching patterns and implements the **Stale-While-Revalidate (SWR)** caching pattern on the client.
> 
>   

## Key Concepts

- **Client State vs. Server State**: Client state is synchronous and fully owned by the browser; Server state is asynchronous, remote, shared, and represents a snapshot that can become obsolete at any second.
    
      
    
- **Stale-While-Revalidate (SWR)**: The client immediately renders cached (stale) data from memory to ensure zero perceived loading delay, while simultaneously dispatching a background network request to revalidate and update the cache seamlessly.
    
      
    
- **`staleTime` vs. `gcTime` (formerly `cacheTime`)**:
    
      
    - `staleTime`: Duration until fetched data is considered out-of-date. While fresh, queries read from cache without triggering background refetches. Defaults to `0`.
        
          
        
    - `gcTime`: Duration an unused query entry remains in garbage-collected memory after all subscribing components unmount. Defaults to 5 minutes (`300_000ms`).
        
          
        
- **Request Deduplication**: If multiple independent components on a page call the same query key concurrently, React Query coalesces them into a single network call.
    
      
    
- **Structural Sharing**: Immutably compares incoming network JSON against existing cached data. If objects are structurally identical, React Query preserves memory references to prevent downstream re-renders.
    
      
    

## Common Interview Questions

- What exact problems does React Query solve compared to managing server state in Redux, Zustand, or raw `useEffect`?
    
      
    
- What is the difference between `staleTime` and `gcTime` (`cacheTime`), and what happens when `staleTime` is set to `Infinity`?
    
      
    
- How does the Stale-While-Revalidate pattern improve user perceived performance?
    
      
    
- What is the difference between `isLoading` and `isFetching`?
    
      
    
- How does React Query handle race conditions and request cancellation when component dependencies change rapidly?
    
      
    
- How do you implement Optimistic Updates with rollback on failure using mutations?
    
      
    

## Strong Answers / Talking Points

### 1. What React Query Solves (The Anti-Pattern It Destroys)

- **The Old Pattern**: Storing server data in Redux/Zustand requires manual state flags (`isLoading`, `error`, `data`), manual deduplication, manual cleanup on unmount, and manual polling logic.
    
      
    
- **The Waterfall / Race Condition Problem**: Rapidly updating dependencies in raw `useEffect` triggers out-of-order network responses, where older responses overwrite newer responses. React Query natively supports `AbortController` signal forwarding to cancel stale in-flight requests.
    
      
    
- **Cache Synchronization**: Automatically synchronizes data across disjoint component trees using query keys without needing prop drilling or complex global context selectors.
    
      
    

### 2. The Core Lifecycle Mechanics

1. **First Mount**: Cache is empty $\rightarrow$ triggers hard loading state (`isLoading: true`, `isFetching: true`).
    
      
    
2. **Data Arrives**: Populates the query cache for `['users']` $\rightarrow$ marks data fresh for duration of `staleTime`.
    
      
    
3. **Subsequent Mount (Within `staleTime`)**: Instant render from cache; no network request dispatched (`isLoading: false`, `isFetching: false`).
    
      
    
4. **Subsequent Mount (After `staleTime` expires)**: Instant render from cache (no blank screen), but triggers background revalidation (`isLoading: false`, `isFetching: true`).
    
      
    
5. **Component Unmounts**: Query transitions from _active_ to _inactive_. The `gcTime` timer starts. If no component re-subscribes before `gcTime` elapses, the entry is purged from memory.
    
      
    

### 3. Key Patterns in React Query

- **Automatic Background Revalidation**: Refetches automatically on Window Focus (`refetchOnWindowFocus`), Network Reconnect (`refetchOnReconnect`), or Component Mount (`refetchOnMount`).
    
      
    
- **Dependent / Serial Queries**: Handled declaratively via the `enabled` option without imperative async chains.
    
      
    
- **Pagination & Infinite Loading**: Manages cursor/offset tracking and flattens page arrays via `useInfiniteQuery`.
    
      
    
- **Optimistic UI Updates**: Updates client UI instantaneously before the backend finishes processing; rolls back automatically if the mutation fails.
    
      
    

## Code Snippets / Examples

### Basic SWR Query with Signal Cancellation



```TypeScript
import React from 'react';
import { useQuery } from '@tanstack/react-query';

interface User {
  id: string;
  name: string;
  email: string;
}

const fetchUser = async (userId: string, signal?: AbortSignal): Promise<User> => {
  const res = await fetch(`/api/users/${userId}`, { signal });
  if (!res.ok) throw new Error('Network response was not ok');
  return res.json();
};

export const UserProfile = ({ userId }: { userId: string }) => {
  const {
    data: user,
    isLoading,   // True ONLY during the initial hard fetch when no cache exists
    isFetching,  // True whenever a network request is in flight (including background revalidation)
    error,
  } = useQuery({
    queryKey: ['users', userId],
    // queryFn passes AbortSignal directly to fetch to auto-cancel superseded requests
    queryFn: ({ signal }) => fetchUser(userId, signal),
    staleTime: 60 * 1000,      // Data stays fresh for 1 minute
    gcTime: 5 * 60 * 1000,     // Retain in unused memory cache for 5 minutes
    refetchOnWindowFocus: true, // Auto revalidate when user tabs back
  });

  if (isLoading) return <div>Loading initial skeleton...</div>;
  if (error) return <div>Error loading user details.</div>;

  return (
    <div>
      <h3>{user?.name}</h3>
      <p>{user?.email}</p>
      {/* Visual background sync indicator without blocking UI */}
      {isFetching && <span className="text-xs text-blue-500">Updating...</span>}
    </div>
  );
};
```

### Optimistic Mutation with Automatic Rollback



```TypeScript
import { useMutation, useQueryClient } from '@tanstack/react-query';

interface Todo {
  id: string;
  text: string;
}

export const useAddTodo = () => {
  const queryClient = useQueryClient();

  return useMutation({
    mutationFn: (newTodo: Todo) =>
      fetch('/api/todos', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(newTodo),
      }).then((res) => res.json()),

    // Triggered right before mutation runs
    onMutate: async (newTodo) => {
      // 1. Cancel outgoing queries so they don't overwrite optimistic update
      await queryClient.cancelQueries({ queryKey: ['todos'] });

      // 2. Snapshot current cache state for rollback
      const previousTodos = queryClient.getQueryData<Todo[]>(['todos']);

      // 3. Optimistically insert item into cache immediately
      queryClient.setQueryData<Todo[]>(['todos'], (old = []) => [...old, newTodo]);

      // Return context containing previous state
      return { previousTodos };
    },

    // If server rejects, restore the snapshot
    onError: (err, newTodo, context) => {
      if (context?.previousTodos) {
        queryClient.setQueryData(['todos'], context.previousTodos);
      }
    },

    // Always refetch in background to ensure client matches source-of-truth
    onSettled: () => {
      queryClient.invalidateQueries({ queryKey: ['todos'] });
    },
  });
};
```

## Related Topics

- [[React State Management Architecture, Context vs External Stores vs React Query|Client State vs Server State Architecture]]
    
      
    
- [[TanStack Query Server State and Stale While Revalidate Patterns|HTTP Caching ETag and Stale While Revalidate]]
    
      
    
- [[Comprehensive Performance Optimization Architecture in React|React Performance Optimization]]
    
      
    
- [[AbortController & Multi-Signal Timeout Architectures|Race Conditions and AbortController in JavaScript]]
    
      
    
- [[Why You Might Not Need Redux and Modern State Alternatives|Redux vs Zustand vs TanStack Query]]
    
      
    

## Tags

#fullstack #interview #react-query #tanstack-query #server-state #swr #caching

  

## Revision Checklist

- [ ] Can explain in 60 seconds
    
      
    
- [ ] Can explain trade-offs
    
      
    
- [ ] Can give a real project example
    
      
    
- [ ] Can answer common follow-ups