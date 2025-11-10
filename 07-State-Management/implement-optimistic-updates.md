# What are optimistic updates?

## Question
What are optimistic updates?

## Answer
Update UI immediately, then handle server response. Rollback on error.

## Basic Example
```javascript
const toggleTodo = createAsyncThunk(
  'todos/toggle',
  async (todoId, { rejectWithValue }) => {
    try {
      const response = await fetch(`/api/todos/${todoId}/toggle`, {
        method: 'PATCH'
      });
      return await response.json();
    } catch (error) {
      return rejectWithValue(error.message);
    }
  }
);

const todosSlice = createSlice({
  name: 'todos',
  initialState: { items: [] },
  reducers: {
    // Optimistic update
    toggleOptimistic: (state, action) => {
      const todo = state.items.find(t => t.id === action.payload);
      if (todo) {
        todo.completed = !todo.completed; // Update immediately
      }
    }
  },
  extraReducers: (builder) => {
    builder
      .addCase(toggleTodo.rejected, (state, action) => {
        // Rollback on server error
        const todo = state.items.find(t => t.id === action.meta.arg);
        if (todo) {
          todo.completed = !todo.completed; // Undo the change
        }
      });
  }
});
```

## In Component
```javascript
function TodoItem({ todo }) {
  const dispatch = useDispatch();

  const handleToggle = () => {
    // Update UI immediately
    dispatch(toggleOptimistic(todo.id));
    
    // Then make API call
    dispatch(toggleTodo(todo.id));
  };

  return (
    <div>
      <input
        type="checkbox"
        checked={todo.completed}
        onChange={handleToggle}
      />
      {todo.text}
    </div>
  );
}
```

## Interview Q&A

**Q: What are optimistic updates?**

A: Update the UI immediately, then handle the server response. Makes app feel faster.

**Q: What happens if the server request fails?**

A: Rollback the optimistic update to match server state.

**Q: When should you use optimistic updates?**

A: For actions that usually succeed, like toggles or likes, where immediate feedback improves UX.
    }
);

// Slice with optimistic update
const todosSlice = createSlice({
    name: 'todos',
    initialState,
    reducers: {
        // Optimistic update action
        toggleTodoOptimistic: (state, action) => {
            const todo = state.items.find(t => t.id === action.payload);
            if (todo) {
                todo.completed = !todo.completed;
            }
        },
        // Rollback action
        rollbackToggle: (state, action) => {
            const todo = state.items.find(t => t.id === action.payload);
            if (todo) {
                todo.completed = !todo.completed; // Revert the change
            }
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(toggleTodo.pending, (state, action) => {
                // Apply optimistic update immediately
                const todo = state.items.find(t => t.id === action.meta.arg);
                if (todo) {
                    todo.completed = !todo.completed;
                }
            })
            .addCase(toggleTodo.fulfilled, (state, action) => {
                // Server confirmed the change - no action needed
                // The optimistic update was correct
            })
            .addCase(toggleTodo.rejected, (state, action) => {
                // Server rejected the change - rollback
                const todo = state.items.find(t => t.id === action.meta.arg);
                if (todo) {
                    todo.completed = !todo.completed; // Rollback
                }
                state.error = action.payload as string;
            });
    },
});

export const { toggleTodoOptimistic, rollbackToggle } = todosSlice.actions;
```

### 2. **Component Implementation**

```typescript
import { useDispatch } from 'react-redux';
import { toggleTodo } from './todosSlice';

function TodoItem({ todo }: { todo: Todo }) {
    const dispatch = useDispatch();

    const handleToggle = () => {
        dispatch(toggleTodo(todo.id));
    };

    return (
        <div>
            <input
                type="checkbox"
                checked={todo.completed}
                onChange={handleToggle}
            />
            <span style={{
                textDecoration: todo.completed ? 'line-through' : 'none'
            }}>
                {todo.text}
            </span>
        </div>
    );
}
```

## Advanced Optimistic Update Patterns

### 1. **Optimistic Updates with Rollback Tracking**

```typescript
interface TodosState {
    items: Todo[];
    optimisticUpdates: Record<number, {
        previousState: Partial<Todo>;
        timestamp: number;
    }>;
    loading: Record<number, boolean>;
    error: Record<number, string | null>;
}

const initialState: TodosState = {
    items: [],
    optimisticUpdates: {},
    loading: {},
    error: {},
};

// Enhanced slice with rollback tracking
const todosSlice = createSlice({
    name: 'todos',
    initialState,
    reducers: {
        // Track optimistic update
        applyOptimisticUpdate: (state, action) => {
            const { todoId, updates } = action.payload;
            const todo = state.items.find(t => t.id === todoId);

            if (todo) {
                // Store previous state for potential rollback
                state.optimisticUpdates[todoId] = {
                    previousState: {
                        completed: todo.completed,
                        text: todo.text,
                    },
                    timestamp: Date.now(),
                };

                // Apply optimistic update
                Object.assign(todo, updates);
                state.loading[todoId] = true;
            }
        },

        // Rollback optimistic update
        rollbackOptimisticUpdate: (state, action) => {
            const todoId = action.payload;
            const optimisticUpdate = state.optimisticUpdates[todoId];

            if (optimisticUpdate) {
                const todo = state.items.find(t => t.id === todoId);
                if (todo) {
                    // Restore previous state
                    Object.assign(todo, optimisticUpdate.previousState);
                }

                delete state.optimisticUpdates[todoId];
                delete state.loading[todoId];
                state.error[todoId] = 'Update failed';
            }
        },

        // Confirm optimistic update
        confirmOptimisticUpdate: (state, action) => {
            const todoId = action.payload;
            delete state.optimisticUpdates[todoId];
            delete state.loading[todoId];
            delete state.error[todoId];
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(toggleTodo.pending, (state, action) => {
                const todoId = action.meta.arg;
                const todo = state.items.find(t => t.id === todoId);

                if (todo) {
                    // Track optimistic update
                    state.optimisticUpdates[todoId] = {
                        previousState: { completed: todo.completed },
                        timestamp: Date.now(),
                    };

                    // Apply update
                    todo.completed = !todo.completed;
                    state.loading[todoId] = true;
                    delete state.error[todoId];
                }
            })
            .addCase(toggleTodo.fulfilled, (state, action) => {
                const { todoId } = action.payload;
                // Confirm the optimistic update
                delete state.optimisticUpdates[todoId];
                delete state.loading[todoId];
                delete state.error[todoId];
            })
            .addCase(toggleTodo.rejected, (state, action) => {
                const todoId = action.meta.arg;
                // Rollback the optimistic update
                const optimisticUpdate = state.optimisticUpdates[todoId];

                if (optimisticUpdate) {
                    const todo = state.items.find(t => t.id === todoId);
                    if (todo) {
                        todo.completed = optimisticUpdate.previousState.completed!;
                    }
                }

                delete state.optimisticUpdates[todoId];
                delete state.loading[todoId];
                state.error[todoId] = action.payload as string;
            });
    },
});

export const {
    applyOptimisticUpdate,
    rollbackOptimisticUpdate,
    confirmOptimisticUpdate
} = todosSlice.actions;
```

### 2. **Optimistic Updates with Conflict Resolution**

```typescript
// Thunk with conflict detection
export const updateTodoText = createAsyncThunk(
    'todos/updateText',
    async ({ id, text, version }: { id: number; text: string; version: number }, { rejectWithValue }) => {
        try {
            const response = await fetch(`/api/todos/${id}`, {
                method: 'PATCH',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ text, version }),
            });

            if (response.status === 409) {
                // Conflict - someone else updated it
                const latestTodo = await response.json();
                return rejectWithValue({
                    type: 'conflict',
                    latestTodo,
                    attemptedText: text,
                });
            }

            if (!response.ok) {
                throw new Error('Failed to update todo');
            }

            return await response.json();
        } catch (error) {
            return rejectWithValue({ type: 'error', message: 'Network error' });
        }
    }
);

interface TodosState {
    items: Todo[];
    optimisticUpdates: Record<number, any>;
    conflicts: Record<number, {
        localText: string;
        serverText: string;
        serverTodo: Todo;
    }>;
}

const todosSlice = createSlice({
    name: 'todos',
    initialState: {
        items: [],
        optimisticUpdates: {},
        conflicts: {},
    } as TodosState,
    reducers: {
        // Resolve conflict by choosing local changes
        resolveConflictKeepLocal: (state, action) => {
            const todoId = action.payload;
            const conflict = state.conflicts[todoId];

            if (conflict) {
                const todo = state.items.find(t => t.id === todoId);
                if (todo) {
                    todo.text = conflict.localText;
                    // Re-apply optimistic update
                    state.optimisticUpdates[todoId] = { text: conflict.localText };
                }

                delete state.conflicts[todoId];
            }
        },

        // Resolve conflict by accepting server changes
        resolveConflictKeepServer: (state, action) => {
            const todoId = action.payload;
            const conflict = state.conflicts[todoId];

            if (conflict) {
                const todo = state.items.find(t => t.id === todoId);
                if (todo) {
                    Object.assign(todo, conflict.serverTodo);
                }

                delete state.conflicts[todoId];
                delete state.optimisticUpdates[todoId];
            }
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(updateTodoText.pending, (state, action) => {
                const { id, text } = action.meta.arg;
                const todo = state.items.find(t => t.id === id);

                if (todo) {
                    state.optimisticUpdates[id] = {
                        previousText: todo.text,
                        newText: text,
                        timestamp: Date.now(),
                    };
                    todo.text = text;
                }
            })
            .addCase(updateTodoText.fulfilled, (state, action) => {
                const todoId = action.payload.id;
                delete state.optimisticUpdates[todoId];
                delete state.conflicts[todoId];
            })
            .addCase(updateTodoText.rejected, (state, action) => {
                const { id } = action.meta.arg;
                const payload = action.payload as any;

                if (payload?.type === 'conflict') {
                    // Handle conflict
                    state.conflicts[id] = {
                        localText: state.optimisticUpdates[id]?.newText,
                        serverText: payload.latestTodo.text,
                        serverTodo: payload.latestTodo,
                    };
                } else {
                    // Regular error - rollback
                    const optimisticUpdate = state.optimisticUpdates[id];
                    if (optimisticUpdate) {
                        const todo = state.items.find(t => t.id === id);
                        if (todo) {
                            todo.text = optimisticUpdate.previousText;
                        }
                        delete state.optimisticUpdates[id];
                    }
                }
            });
    },
});

export const {
    resolveConflictKeepLocal,
    resolveConflictKeepServer
} = todosSlice.actions;
```

### 3. **Optimistic Updates with Queues**

```typescript
// For handling multiple rapid updates
interface OptimisticQueue {
    operations: Array<{
        id: string;
        action: any;
        timestamp: number;
        applied: boolean;
    }>;
}

const optimisticQueueSlice = createSlice({
    name: 'optimisticQueue',
    initialState: {
        operations: [],
    } as OptimisticQueue,
    reducers: {
        enqueueOperation: (state, action) => {
            state.operations.push({
                id: action.payload.id,
                action: action.payload.action,
                timestamp: Date.now(),
                applied: false,
            });
        },
        markOperationApplied: (state, action) => {
            const operation = state.operations.find(op => op.id === action.payload);
            if (operation) {
                operation.applied = true;
            }
        },
        removeOperation: (state, action) => {
            state.operations = state.operations.filter(op => op.id !== action.payload);
        },
        rollbackUnappliedOperations: (state) => {
            // Rollback operations that haven't been confirmed by server
            state.operations = state.operations.filter(op => op.applied);
        },
    },
});

// Thunk that uses optimistic queue
export const optimisticToggleTodo = createAsyncThunk(
    'todos/optimisticToggle',
    async (todoId: number, { dispatch, getState }) => {
        const operationId = `toggle-${todoId}-${Date.now()}`;

        // Enqueue the operation
        dispatch(enqueueOperation({
            id: operationId,
            action: { type: 'toggle', todoId },
        }));

        try {
            // Apply optimistic update
            dispatch(toggleTodoOptimistic(todoId));

            // Send to server
            const response = await fetch(`/api/todos/${todoId}/toggle`, {
                method: 'PATCH',
            });

            if (!response.ok) {
                throw new Error('Failed to toggle');
            }

            // Mark as applied
            dispatch(markOperationApplied(operationId));

            return { todoId, completed: await response.json() };
        } catch (error) {
            // Rollback on error
            dispatch(rollbackToggle(todoId));
            dispatch(removeOperation(operationId));
            throw error;
        }
    }
);
```

## Optimistic Update Patterns for Different Operations

### 1. **Create Operations**

```typescript
export const createTodoOptimistic = createAsyncThunk(
    'todos/createOptimistic',
    async (text: string, { rejectWithValue }) => {
        const tempId = `temp-${Date.now()}`;
        const newTodo = {
            id: tempId,
            text,
            completed: false,
            isOptimistic: true,
        };

        try {
            // Create optimistic todo
            const response = await fetch('/api/todos', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ text }),
            });

            if (!response.ok) {
                throw new Error('Failed to create todo');
            }

            const realTodo = await response.json();
            return { tempId, realTodo };
        } catch (error) {
            return rejectWithValue({ tempId, error: 'Failed to create todo' });
        }
    }
);

const todosSlice = createSlice({
    name: 'todos',
    initialState,
    extraReducers: (builder) => {
        builder
            .addCase(createTodoOptimistic.pending, (state, action) => {
                const tempId = `temp-${Date.now()}`;
                const newTodo = {
                    id: tempId,
                    text: action.meta.arg,
                    completed: false,
                    isOptimistic: true,
                };
                state.items.push(newTodo);
            })
            .addCase(createTodoOptimistic.fulfilled, (state, action) => {
                const { tempId, realTodo } = action.payload;
                // Replace optimistic todo with real one
                const index = state.items.findIndex(t => t.id === tempId);
                if (index !== -1) {
                    state.items[index] = { ...realTodo, isOptimistic: false };
                }
            })
            .addCase(createTodoOptimistic.rejected, (state, action) => {
                const { tempId } = action.payload as any;
                // Remove optimistic todo
                state.items = state.items.filter(t => t.id !== tempId);
                state.error = 'Failed to create todo';
            });
    },
});
```

### 2. **Delete Operations**

```typescript
export const deleteTodoOptimistic = createAsyncThunk(
    'todos/deleteOptimistic',
    async (todoId: number, { rejectWithValue }) => {
        try {
            const response = await fetch(`/api/todos/${todoId}`, {
                method: 'DELETE',
            });

            if (!response.ok) {
                throw new Error('Failed to delete todo');
            }

            return todoId;
        } catch (error) {
            return rejectWithValue(todoId);
        }
    }
);

const todosSlice = createSlice({
    name: 'todos',
    initialState,
    reducers: {
        // Store deleted items for potential restoration
        markDeleted: (state, action) => {
            const todo = state.items.find(t => t.id === action.payload);
            if (todo) {
                todo.isDeleted = true;
                todo.deletedAt = Date.now();
            }
        },
        restoreDeleted: (state, action) => {
            const todo = state.items.find(t => t.id === action.payload);
            if (todo) {
                delete todo.isDeleted;
                delete todo.deletedAt;
            }
        },
    },
    extraReducers: (builder) => {
        builder
            .addCase(deleteTodoOptimistic.pending, (state, action) => {
                const todoId = action.meta.arg;
                const todo = state.items.find(t => t.id === todoId);
                if (todo) {
                    todo.isDeleted = true;
                    todo.deletedAt = Date.now();
                }
            })
            .addCase(deleteTodoOptimistic.fulfilled, (state, action) => {
                // Permanently remove the todo
                state.items = state.items.filter(t => t.id !== action.payload);
            })
            .addCase(deleteTodoOptimistic.rejected, (state, action) => {
                const todoId = action.payload as number;
                // Restore the deleted todo
                const todo = state.items.find(t => t.id === todoId);
                if (todo) {
                    delete todo.isDeleted;
                    delete todo.deletedAt;
                }
                state.error = 'Failed to delete todo';
            });
    },
});
```

## Component Patterns for Optimistic Updates

### 1. **Optimistic Update Hook**

```typescript
function useOptimisticUpdate<T>(
    asyncAction: (payload: T) => any,
    rollbackAction: (payload: T) => any
) {
    const dispatch = useDispatch();
    const [isOptimistic, setIsOptimistic] = useState(false);
    const [error, setError] = useState<string | null>(null);

    const executeOptimistic = useCallback(async (payload: T) => {
        setIsOptimistic(true);
        setError(null);

        try {
            await dispatch(asyncAction(payload)).unwrap();
            setIsOptimistic(false);
        } catch (err) {
            setError(err as string);
            // Rollback the optimistic update
            dispatch(rollbackAction(payload));
            setIsOptimistic(false);
        }
    }, [dispatch, asyncAction, rollbackAction]);

    return { executeOptimistic, isOptimistic, error };
}

// Usage
function TodoItem({ todo }: { todo: Todo }) {
    const { executeOptimistic, isOptimistic, error } = useOptimisticUpdate(
        (id: number) => toggleTodo(id),
        (id: number) => rollbackToggle(id)
    );

    return (
        <div>
            <input
                type="checkbox"
                checked={todo.completed}
                onChange={() => executeOptimistic(todo.id)}
                disabled={isOptimistic}
            />
            <span>{todo.text}</span>
            {isOptimistic && <span>Updating...</span>}
            {error && <span style={{ color: 'red' }}>{error}</span>}
        </div>
    );
}
```

### 2. **Conflict Resolution UI**

```typescript
function ConflictResolver({ conflict, onResolve }: {
    conflict: any;
    onResolve: (resolution: 'local' | 'server') => void;
}) {
    return (
        <div className="conflict-modal">
            <h3>Conflict Detected</h3>
            <p>This item was modified by someone else.</p>

            <div className="conflict-options">
                <div className="option">
                    <h4>Your Changes:</h4>
                    <p>{conflict.localText}</p>
                    <button onClick={() => onResolve('local')}>
                        Keep Your Changes
                    </button>
                </div>

                <div className="option">
                    <h4>Server Changes:</h4>
                    <p>{conflict.serverText}</p>
                    <button onClick={() => onResolve('server')}>
                        Accept Server Changes
                    </button>
                </div>
            </div>
        </div>
    );
}

function TodoList() {
    const dispatch = useDispatch();
    const conflicts = useSelector((state: RootState) => state.todos.conflicts);

    const handleResolveConflict = (todoId: number, resolution: 'local' | 'server') => {
        if (resolution === 'local') {
            dispatch(resolveConflictKeepLocal(todoId));
        } else {
            dispatch(resolveConflictKeepServer(todoId));
        }
    };

    return (
        <div>
            {/* Render conflicts */}
            {Object.entries(conflicts).map(([todoId, conflict]) => (
                <ConflictResolver
                    key={todoId}
                    conflict={conflict}
                    onResolve={(resolution) =>
                        handleResolveConflict(Number(todoId), resolution)
                    }
                />
            ))}

            {/* Render todos */}
            {/* ... */}
        </div>
    );
}
```

## Best Practices

### 1. **When to Use Optimistic Updates**

```typescript
// ✅ Good: Fast operations with high success rate
const toggleTodo = () => dispatch(toggleTodoOptimistic(todoId));

// ✅ Good: Operations user expects to succeed immediately
const markAsRead = () => dispatch(markAsReadOptimistic(notificationId));

// ❌ Avoid: Operations with high failure rate
// const transferMoney = () => dispatch(transferMoneyOptimistic(amount));

// ❌ Avoid: Operations requiring user confirmation
// const deleteAccount = () => dispatch(deleteAccountOptimistic());
```

### 2. **Error Handling Strategy**

```typescript
// ✅ Good: Clear error messages
.addCase(toggleTodo.rejected, (state, action) => {
    // Rollback optimistic update
    const todo = state.items.find(t => t.id === action.meta.arg);
    if (todo) {
        todo.completed = !todo.completed;
    }

    // Show user-friendly error
    state.error = 'Unable to update todo. Please try again.';
});

// ✅ Good: Retry mechanism
.addCase(toggleTodo.rejected, (state, action) => {
    const todoId = action.meta.arg;

    if (state.retryCount[todoId] < 3) {
        // Auto-retry after delay
        setTimeout(() => {
            dispatch(toggleTodo(todoId));
        }, 1000 * Math.pow(2, state.retryCount[todoId]));

        state.retryCount[todoId] += 1;
    } else {
        // Give up and rollback
        const todo = state.items.find(t => t.id === todoId);
        if (todo) {
            todo.completed = !todo.completed;
        }
        state.error = 'Update failed after multiple attempts';
    }
});
```

### 3. **Performance Considerations**

```typescript
// ✅ Good: Debounce rapid updates
const debouncedUpdate = useMemo(
    () => debounce((text: string) => {
        dispatch(updateTodoText({ id: todo.id, text, version: todo.version }));
    }, 500),
    [dispatch, todo.id, todo.version]
);

// ✅ Good: Batch related updates
const batchUpdateTodos = (updates: Array<{ id: number; completed: boolean }>) => {
    // Apply all optimistic updates at once
    updates.forEach(update => {
        dispatch(toggleTodoOptimistic(update.id));
    });

    // Send batch request
    dispatch(batchToggleTodos(updates));
};
```

### 4. **Testing Optimistic Updates**

```typescript
describe('optimistic updates', () => {
    it('applies optimistic update immediately', () => {
        const state = todosSlice.reducer(
            { items: [{ id: 1, completed: false }] },
            toggleTodo.pending(1)
        );

        expect(state.items[0].completed).toBe(true);
    });

    it('rolls back on rejection', () => {
        let state = todosSlice.reducer(
            { items: [{ id: 1, completed: false }] },
            toggleTodo.pending(1)
        );

        state = todosSlice.reducer(state, toggleTodo.rejected(null, 1));

        expect(state.items[0].completed).toBe(false);
    });

    it('handles conflicts correctly', () => {
        const conflictPayload = {
            type: 'conflict',
            latestTodo: { id: 1, text: 'Server text', version: 2 },
        };

        const state = todosSlice.reducer(
            {
                items: [{ id: 1, text: 'Local text' }],
                optimisticUpdates: { 1: { newText: 'Local text' } }
            },
            updateTodoText.rejected(conflictPayload, { id: 1, text: 'Local text', version: 1 })
        );

        expect(state.conflicts[1]).toBeDefined();
        expect(state.conflicts[1].serverText).toBe('Server text');
    });
});
```

## Common Interview Questions

### Q: What are optimistic updates and when should you use them?

**A:** Optimistic updates immediately update the UI before the server confirms the action, providing instant feedback. Use them for fast, reliable operations like toggling a checkbox or marking notifications as read, but avoid them for critical operations like financial transactions.

### Q: How do you handle rollbacks in optimistic updates?

**A:** I store the previous state before applying the optimistic update. If the server request fails, I restore the previous state and show an error message. For complex scenarios, I track optimistic updates with timestamps and provide manual conflict resolution.

### Q: What are the challenges with optimistic updates?

**A:** The main challenges are handling conflicts when multiple users edit the same data, managing rollbacks properly, and ensuring the UI stays consistent. You need good error handling and possibly conflict resolution UI for complex applications.

### Q: How do optimistic updates differ from regular async operations?

**A:** Regular async operations update the UI only after the server responds successfully. Optimistic updates change the UI immediately, then handle success or failure. This provides better perceived performance but requires careful error handling and rollback logic.

## Summary

**Key Optimistic Update Patterns:**
1. **Basic Pattern**: Apply update immediately, rollback on error
2. **Tracked Updates**: Store previous state for reliable rollback
3. **Conflict Resolution**: Handle concurrent modifications
4. **Queued Operations**: Manage rapid successive updates
5. **Create/Delete**: Handle temporary IDs and restoration

**Best Practices:**
- Use for fast, reliable operations only
- Always implement proper rollback logic
- Handle conflicts gracefully
- Provide clear user feedback
- Test thoroughly including error scenarios

**Interview Tip:** "Optimistic updates improve perceived performance by updating the UI immediately while the server request is pending. I implement them by applying the change instantly, storing the previous state for rollback, and reverting on server errors. They're great for responsive UIs but require careful error handling."