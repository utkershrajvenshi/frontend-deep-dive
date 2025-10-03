# Managing State in Large Applications: A Comprehensive Guide

Excellent question! This is where senior developers separate themselves—understanding not just *how* to manage state, but *when* to use each approach and *why*. Let me coach you through a complete answer.

---

## **The Framework for Your Answer**

Structure your response around these key areas:

1. **State categorization** (identifying what kind of state you have)
2. **State placement strategies** (where state should live)
3. **Tools and patterns** (the solutions available)
4. **Trade-offs and decision criteria** (how to choose)
5. **Real-world architecture** (practical examples)

---

## **Part 1: Understanding State Categories**

**Interview Talking Point:**
*"Before choosing a state management solution, I first categorize state by its characteristics. Not all state is the same, and different types require different approaches."*

### **The State Categorization Matrix**

```javascript
// 1. SERVER STATE (Cached data from backend)
const userProfile = {
  id: 123,
  name: 'John Doe',
  email: 'john@example.com'
  // Fetched from API, can become stale, needs revalidation
};

// 2. UI STATE (Visual interaction state)
const uiState = {
  isModalOpen: false,
  selectedTab: 'profile',
  isDarkMode: true
  // Ephemeral, client-only, resets on unmount
};

// 3. FORM STATE (Temporary user input)
const formState = {
  values: { email: '', password: '' },
  errors: { email: 'Invalid format' },
  touched: { email: true },
  isDirty: true,
  isSubmitting: false
  // Complex validation, needs to be serializable
};

// 4. URL STATE (Navigation and sharing state)
const urlState = {
  page: 2,
  sortBy: 'date',
  filters: { category: 'tech', status: 'published' }
  // Bookmarkable, shareable, persists across sessions
};

// 5. GLOBAL APPLICATION STATE (Cross-cutting concerns)
const appState = {
  currentUser: { id: 1, role: 'admin' },
  featureFlags: { newDesign: true },
  i18n: { locale: 'en-US' }
  // Accessed by many components, changes infrequently
};

// 6. LOCAL COMPONENT STATE (Isolated, temporary)
const localState = {
  isHovered: false,
  animationProgress: 0.5,
  internalCounter: 5
  // Only used by one component, doesn't need sharing
};
```

**Why This Matters:**

```javascript
// ❌ Common mistake: Treating everything as global state
const store = createStore({
  // Overkill - this is local UI state!
  isDropdownOpen: false,
  
  // Wrong place - this is server state!
  usersList: [],
  usersLoading: false,
  usersError: null,
  
  // Should be in URL!
  currentPage: 1,
  searchQuery: ''
});

// ✅ Better: Right tool for each state type
function UsersList() {
  // Server state - use React Query
  const { data: users, isLoading } = useQuery('users', fetchUsers);
  
  // URL state - use router
  const [searchParams, setSearchParams] = useSearchParams();
  const currentPage = Number(searchParams.get('page')) || 1;
  
  // Local UI state - use useState
  const [isDropdownOpen, setIsDropdownOpen] = useState(false);
  
  // Global app state - use Context or Redux
  const { currentUser } = useAuth();
}
```

---

## **Part 2: State Placement Strategies**

### **The Lifting State Rule**

**Interview Principle:**
*"Keep state as local as possible, lift only when necessary. This is about finding the lowest common ancestor."*

```javascript
// SCENARIO: Multiple components need shared state

// ❌ BAD: State too low (can't share)
function Parent() {
  return (
    <>
      <Counter />  {/* Has count state */}
      <Display />  {/* Can't access count! */}
    </>
  );
}

// ❌ BAD: State too high (prop drilling)
function App() {
  const [count, setCount] = useState(0);
  
  return (
    <Layout>
      <Sidebar>
        <Navigation>
          <Menu>
            <Counter count={count} setCount={setCount} /> {/* 5 levels! */}
          </Menu>
        </Navigation>
      </Sidebar>
    </Layout>
  );
}

// ✅ GOOD: State at lowest common ancestor
function CounterSection() {
  const [count, setCount] = useState(0);
  
  return (
    <>
      <Counter count={count} setCount={setCount} />
      <Display count={count} />
    </>
  );
}

function App() {
  return (
    <Layout>
      <Sidebar>
        <CounterSection /> {/* State contained here */}
      </Sidebar>
    </Layout>
  );
}
```

### **The Composition Pattern**

```javascript
// ✅ EXCELLENT: Avoid prop drilling with composition
function App() {
  const [count, setCount] = useState(0);
  
  // Pass components as children/props instead of drilling
  return (
    <Layout
      sidebar={
        <Sidebar>
          <Navigation>
            <Menu>
              <Counter count={count} setCount={setCount} />
            </Menu>
          </Navigation>
        </Sidebar>
      }
    />
  );
}

// Layout doesn't need to know about count
function Layout({ sidebar }) {
  return (
    <div className="layout">
      {sidebar}
    </div>
  );
}
```

### **Colocation Principle**

```javascript
// ✅ Keep state close to where it's used
function TodoList() {
  const [todos, setTodos] = useState([]);
  const [filter, setFilter] = useState('all');
  
  // Everything todos needs is right here
  const filteredTodos = useMemo(() => {
    return todos.filter(todo => {
      if (filter === 'completed') return todo.done;
      if (filter === 'active') return !todo.done;
      return true;
    });
  }, [todos, filter]);
  
  return (
    <>
      <FilterButtons filter={filter} onChange={setFilter} />
      <TodoItems todos={filteredTodos} />
      <AddTodoForm onAdd={newTodo => setTodos([...todos, newTodo])} />
    </>
  );
}

// Don't do this unless multiple unrelated components need it
// const todosAtom = atom([]); // Global state
```

---

## **Part 3: State Management Tools & Patterns**

### **1. Local State (useState/useReducer)**

**When to use:**
- State used by single component or small component tree
- Simple UI interactions
- No need for time-travel debugging or middleware

```javascript
// Simple counter - useState is perfect
function Counter() {
  const [count, setCount] = useState(0);
  
  return (
    <div>
      <p>{count}</p>
      <button onClick={() => setCount(count + 1)}>+</button>
    </div>
  );
}

// Complex form - useReducer is better
function ComplexForm() {
  const [state, dispatch] = useReducer(formReducer, initialState);
  
  return (
    <form onSubmit={e => {
      e.preventDefault();
      dispatch({ type: 'SUBMIT' });
    }}>
      <input
        value={state.values.email}
        onChange={e => dispatch({ 
          type: 'CHANGE_FIELD', 
          field: 'email', 
          value: e.target.value 
        })}
      />
      {state.errors.email && <span>{state.errors.email}</span>}
    </form>
  );
}

function formReducer(state, action) {
  switch (action.type) {
    case 'CHANGE_FIELD':
      return {
        ...state,
        values: { ...state.values, [action.field]: action.value },
        touched: { ...state.touched, [action.field]: true }
      };
    
    case 'SET_ERROR':
      return {
        ...state,
        errors: { ...state.errors, [action.field]: action.error }
      };
    
    case 'SUBMIT':
      return { ...state, isSubmitting: true };
    
    case 'SUBMIT_SUCCESS':
      return initialState; // Reset form
    
    default:
      return state;
  }
}
```

### **2. Context API (React.Context)**

**When to use:**
- Avoid prop drilling for 3+ levels
- Theme, i18n, auth state (changes infrequently)
- **NOT for frequently updating state** (causes re-renders)

```javascript
// ✅ GOOD: Infrequent updates (theme)
const ThemeContext = createContext();

function ThemeProvider({ children }) {
  const [theme, setTheme] = useState('light');
  
  // Theme changes rarely
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      {children}
    </ThemeContext.Provider>
  );
}

function Button() {
  const { theme } = useContext(ThemeContext);
  return <button className={theme}>Click</button>;
}

// ❌ BAD: Frequent updates (every keystroke!)
const FormContext = createContext();

function FormProvider({ children }) {
  const [values, setValues] = useState({});
  
  // Every input change re-renders ALL consumers! 😱
  return (
    <FormContext.Provider value={{ values, setValues }}>
      {children}
    </FormContext.Provider>
  );
}

// ✅ BETTER: Split contexts by update frequency
const FormValuesContext = createContext(); // Updates frequently
const FormActionsContext = createContext(); // Never changes

function FormProvider({ children }) {
  const [values, setValues] = useState({});
  
  // Actions object is stable (won't cause re-renders)
  const actions = useMemo(() => ({
    updateField: (name, value) => {
      setValues(prev => ({ ...prev, [name]: value }));
    }
  }), []);
  
  return (
    <FormActionsContext.Provider value={actions}>
      <FormValuesContext.Provider value={values}>
        {children}
      </FormValuesContext.Provider>
    </FormActionsContext.Provider>
  );
}

// Components only re-render if they use FormValuesContext
function FormField({ name }) {
  const values = useContext(FormValuesContext);
  const { updateField } = useContext(FormActionsContext);
  
  return (
    <input
      value={values[name] || ''}
      onChange={e => updateField(name, e.target.value)}
    />
  );
}
```

**Context Performance Optimization:**

```javascript
// ✅ Advanced pattern: Selector-based context
function createFastContext(initialValue) {
  const Context = createContext(null);
  
  function Provider({ children }) {
    const [state, setState] = useState(initialValue);
    const subscribers = useRef(new Set());
    
    const store = useMemo(() => ({
      get: () => state,
      set: setState,
      subscribe: (callback) => {
        subscribers.current.add(callback);
        return () => subscribers.current.delete(callback);
      }
    }), [state]);
    
    useEffect(() => {
      subscribers.current.forEach(callback => callback(state));
    }, [state]);
    
    return <Context.Provider value={store}>{children}</Context.Provider>;
  }
  
  function useStore(selector) {
    const store = useContext(Context);
    const [selectedState, setSelectedState] = useState(() => 
      selector(store.get())
    );
    
    useEffect(() => {
      return store.subscribe((state) => {
        const newSelectedState = selector(state);
        if (!Object.is(newSelectedState, selectedState)) {
          setSelectedState(newSelectedState);
        }
      });
    }, [store, selector, selectedState]);
    
    return selectedState;
  }
  
  return { Provider, useStore };
}

// Usage - only re-renders when selected data changes
const { Provider, useStore } = createFastContext({ users: [], posts: [] });

function UsersList() {
  // Only re-renders when users array changes, not posts
  const users = useStore(state => state.users);
  return <>{users.map(u => <div key={u.id}>{u.name}</div>)}</>;
}
```

### **3. Redux (or Redux Toolkit)**

**When to use:**
- Large application with complex state interactions
- Need time-travel debugging
- Team already familiar with Redux patterns
- Want centralized state with strict update rules

```javascript
// Modern Redux with Redux Toolkit
import { createSlice, configureStore } from '@reduxjs/toolkit';

// ✅ Feature-based slice
const todosSlice = createSlice({
  name: 'todos',
  initialState: {
    items: [],
    filter: 'all',
    loading: false
  },
  reducers: {
    addTodo: (state, action) => {
      // Immer allows "mutations"
      state.items.push({
        id: Date.now(),
        text: action.payload,
        done: false
      });
    },
    toggleTodo: (state, action) => {
      const todo = state.items.find(t => t.id === action.payload);
      if (todo) {
        todo.done = !todo.done;
      }
    },
    setFilter: (state, action) => {
      state.filter = action.payload;
    }
  },
  // Async actions
  extraReducers: (builder) => {
    builder
      .addCase(fetchTodos.pending, (state) => {
        state.loading = true;
      })
      .addCase(fetchTodos.fulfilled, (state, action) => {
        state.loading = false;
        state.items = action.payload;
      });
  }
});

// Thunk for async operations
import { createAsyncThunk } from '@reduxjs/toolkit';

const fetchTodos = createAsyncThunk(
  'todos/fetchTodos',
  async (userId) => {
    const response = await fetch(`/api/users/${userId}/todos`);
    return response.json();
  }
);

// Store configuration
const store = configureStore({
  reducer: {
    todos: todosSlice.reducer,
    user: userSlice.reducer,
    settings: settingsSlice.reducer
  },
  middleware: (getDefaultMiddleware) =>
    getDefaultMiddleware().concat(logger, analytics)
});

// Component usage
function TodoList() {
  const dispatch = useDispatch();
  const todos = useSelector(state => state.todos.items);
  const filter = useSelector(state => state.todos.filter);
  
  // Memoized selector to avoid re-renders
  const filteredTodos = useSelector(
    state => selectFilteredTodos(state, filter),
    shallowEqual
  );
  
  return (
    <>
      {filteredTodos.map(todo => (
        <TodoItem
          key={todo.id}
          todo={todo}
          onToggle={() => dispatch(todosSlice.actions.toggleTodo(todo.id))}
        />
      ))}
      <button onClick={() => dispatch(fetchTodos(userId))}>
        Refresh
      </button>
    </>
  );
}

// Reselect for memoization
import { createSelector } from '@reduxjs/toolkit';

const selectFilteredTodos = createSelector(
  [state => state.todos.items, (state, filter) => filter],
  (items, filter) => {
    if (filter === 'completed') return items.filter(t => t.done);
    if (filter === 'active') return items.filter(t => !t.done);
    return items;
  }
);
```

**Redux Trade-offs:**

```javascript
// ✅ Pros:
// - Single source of truth
// - Time-travel debugging (Redux DevTools)
// - Predictable state updates
// - Great for complex state logic
// - Middleware for side effects

// ❌ Cons:
// - Boilerplate (even with RTK)
// - Learning curve
// - Overkill for simple apps
// - Not optimized for server state

// When Redux makes sense:
const reduxFits = {
  appSize: 'large',
  stateComplexity: 'high',
  teamExperience: 'familiar with Redux',
  debuggingNeeds: 'time-travel required',
  stateType: 'mostly client state, not server cache'
};
```

### **4. Zustand (Lightweight Alternative)**

**When to use:**
- Want Redux-like patterns without boilerplate
- Small to medium apps
- Need simple global state

```javascript
import create from 'zustand';

// ✅ Simple, clean API
const useStore = create((set, get) => ({
  // State
  count: 0,
  user: null,
  todos: [],
  
  // Actions
  increment: () => set(state => ({ count: state.count + 1 })),
  
  setUser: (user) => set({ user }),
  
  addTodo: (text) => set(state => ({
    todos: [...state.todos, { id: Date.now(), text, done: false }]
  })),
  
  // Async action
  fetchTodos: async () => {
    const todos = await fetch('/api/todos').then(r => r.json());
    set({ todos });
  },
  
  // Computed values
  completedCount: () => {
    return get().todos.filter(t => t.done).length;
  }
}));

// Component usage - only re-renders when selected state changes
function Counter() {
  const count = useStore(state => state.count);
  const increment = useStore(state => state.increment);
  
  return <button onClick={increment}>{count}</button>;
}

// Middleware support
import { devtools, persist } from 'zustand/middleware';

const useStore = create(
  devtools(
    persist(
      (set) => ({
        count: 0,
        increment: () => set(state => ({ count: state.count + 1 }))
      }),
      { name: 'counter-storage' }
    )
  )
);
```

### **5. Jotai/Recoil (Atomic State)**

**When to use:**
- Need fine-grained reactivity
- Want to avoid provider hell
- Bottom-up state approach

```javascript
import { atom, useAtom, useAtomValue, useSetAtom } from 'jotai';

// ✅ Define atoms (pieces of state)
const countAtom = atom(0);
const userAtom = atom(null);

// Derived atoms (computed state)
const doubledCountAtom = atom(
  (get) => get(countAtom) * 2
);

// Async atoms
const todosAtom = atom(async (get) => {
  const userId = get(userAtom)?.id;
  if (!userId) return [];
  
  const response = await fetch(`/api/users/${userId}/todos`);
  return response.json();
});

// Write-only atoms (actions)
const incrementAtom = atom(
  null, // no read
  (get, set) => {
    set(countAtom, get(countAtom) + 1);
  }
);

// Component usage
function Counter() {
  const [count, setCount] = useAtom(countAtom);
  const doubled = useAtomValue(doubledCountAtom);
  
  return (
    <div>
      <p>Count: {count}</p>
      <p>Doubled: {doubled}</p>
      <button onClick={() => setCount(c => c + 1)}>+</button>
    </div>
  );
}

// Only subscribe to what you need
function TodoCount() {
  const count = useAtomValue(
    useMemo(() => atom(get => get(todosAtom).length), [])
  );
  
  return <span>{count} todos</span>;
}
```

### **6. React Query / TanStack Query (Server State)**

**When to use:**
- Managing server data (the RIGHT tool for this!)
- Automatic caching, refetching, synchronization
- Loading and error states

```javascript
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

// ✅ EXCELLENT for server state
function UserProfile({ userId }) {
  const queryClient = useQueryClient();
  
  // Automatic caching, refetching, error handling
  const { data: user, isLoading, error, refetch } = useQuery({
    queryKey: ['user', userId],
    queryFn: () => fetch(`/api/users/${userId}`).then(r => r.json()),
    staleTime: 5 * 60 * 1000, // 5 minutes
    cacheTime: 10 * 60 * 1000, // 10 minutes
    refetchOnWindowFocus: true,
    retry: 3
  });
  
  // Mutations with optimistic updates
  const updateUser = useMutation({
    mutationFn: (updates) => 
      fetch(`/api/users/${userId}`, {
        method: 'PATCH',
        body: JSON.stringify(updates)
      }),
    
    // Optimistic update
    onMutate: async (newData) => {
      await queryClient.cancelQueries(['user', userId]);
      
      const previousUser = queryClient.getQueryData(['user', userId]);
      
      queryClient.setQueryData(['user', userId], old => ({
        ...old,
        ...newData
      }));
      
      return { previousUser };
    },
    
    // Rollback on error
    onError: (err, newData, context) => {
      queryClient.setQueryData(
        ['user', userId],
        context.previousUser
      );
    },
    
    // Refetch on success
    onSettled: () => {
      queryClient.invalidateQueries(['user', userId]);
    }
  });
  
  if (isLoading) return <Spinner />;
  if (error) return <Error error={error} retry={refetch} />;
  
  return (
    <div>
      <h1>{user.name}</h1>
      <button onClick={() => updateUser.mutate({ name: 'New Name' })}>
        Update
      </button>
    </div>
  );
}

// Prefetching
function UsersList() {
  const queryClient = useQueryClient();
  const { data: users } = useQuery(['users'], fetchUsers);
  
  return users.map(user => (
    <Link
      key={user.id}
      to={`/users/${user.id}`}
      onMouseEnter={() => {
        // Prefetch on hover!
        queryClient.prefetchQuery(['user', user.id], () =>
          fetchUser(user.id)
        );
      }}
    >
      {user.name}
    </Link>
  ));
}
```

**Why React Query Changes Everything:**

```javascript
// ❌ OLD WAY: Manual server state in Redux
const todosSlice = createSlice({
  name: 'todos',
  initialState: {
    data: [],
    loading: false,
    error: null,
    lastFetched: null
  },
  reducers: {
    fetchStart: (state) => { state.loading = true; },
    fetchSuccess: (state, action) => {
      state.loading = false;
      state.data = action.payload;
      state.lastFetched = Date.now();
    },
    fetchError: (state, action) => {
      state.loading = false;
      state.error = action.payload;
    }
  }
});

// Need to manually:
// - Handle loading states
// - Handle errors
// - Implement caching
// - Implement refetching
// - Handle race conditions
// - Implement optimistic updates
// - Handle pagination
// - etc...

// ✅ NEW WAY: React Query handles it all
const { data, isLoading, error } = useQuery('todos', fetchTodos);

// Automatic:
// ✓ Caching
// ✓ Background refetching
// ✓ Stale-while-revalidate
// ✓ Deduplication
// ✓ Window focus refetching
// ✓ Network status refetching
// ✓ Garbage collection
// ✓ Pagination
// ✓ Infinite scroll
```

---

## **Part 4: Decision Framework**

### **The State Management Decision Tree**

```javascript
// START HERE: What kind of state is it?

// 1. Is it SERVER data (from API)?
if (isServerData) {
  return 'React Query / SWR / RTK Query';
  // DON'T put server data in Redux/Context!
}

// 2. Is it URL state (for navigation/sharing)?
if (isURLState) {
  return 'React Router / Next.js router / URL params';
  // searchParams, pathname, etc.
}

// 3. Is it FORM state?
if (isFormState) {
  if (isSimple) {
    return 'useState + controlled inputs';
  } else {
    return 'React Hook Form / Formik';
  }
}

// 4. Is it used by SINGLE component?
if (isSingleComponent) {
  if (isSimple) {
    return 'useState';
  } else {
    return 'useReducer';
  }
}

// 5. Is it used by FEW nearby components?
if (isFewComponents && areNearby) {
  return 'Lift state to common parent + props';
}

// 6. Does it update FREQUENTLY?
if (updatesFrequently) {
  if (needsFineGrainedReactivity) {
    return 'Jotai / Recoil';
  } else {
    return 'Zustand with selectors';
  }
}

// 7. Is it GLOBAL app state?
if (isGlobalState) {
  if (isSimple && updatesInfrequently) {
    return 'React Context';
  }
  
  if (needsComplexLogic || debugging) {
    return 'Redux Toolkit';
  }
  
  return 'Zustand'; // Sweet spot for most cases
}
```

### **Real-World Application Architecture**

```javascript
// ✅ RECOMMENDED: Hybrid approach
const AppStateArchitecture = {
  // Server data → React Query
  serverData: {
    tool: 'React Query',
    examples: ['users', 'posts', 'products', 'orders']
  },
  
  // Global app state → Zustand (or Redux)
  globalState: {
    tool: 'Zustand',
    examples: ['currentUser', 'theme', 'featureFlags', 'i18n']
  },
  
  // URL state → Router
  urlState: {
    tool: 'React Router',
    examples: ['page', 'filters', 'sortBy', 'selectedId']
  },
  
  // Form state → React Hook Form
  formState: {
    tool: 'React Hook Form',
    examples: ['loginForm', 'profileForm', 'checkoutForm']
  },
  
  // Local state → useState/useReducer
  localState: {
    tool: 'useState/useReducer',
    examples: ['isDropdownOpen', 'currentStep', 'localFilters']
  }
};
```

**Example: E-commerce App Architecture**

```javascript
// app/store/globalStore.js - Zustand for global client state
import create from 'zustand';
import { persist } from 'zustand/middleware';

export const useGlobalStore = create(
  persist(
    (set) => ({
      // Auth state
      user: null,
      setUser: (user) => set({ user }),
      logout: () => set({ user: null }),
      
      // UI preferences
      theme: 'light',
      toggleTheme: () => set(state => ({ 
        theme: state.theme === 'light' ? 'dark' : 'light' 
      })),
      
      // Shopping cart (client-side only)
      cart: [],
      addToCart: (product) => set(state => ({
        cart: [...state.cart, product]
      })),
      removeFromCart: (productId) => set(state => ({
        cart: state.cart.filter(p => p.id !== productId)
      }))
    }),
    { name: 'app-storage' }
  )
);

// app/api/products.js - React Query for server state
import { useQuery, useMutation, useQueryClient } from '@tanstack/react-query';

export function useProducts(filters) {
  return useQuery({
    queryKey: ['products', filters],
    queryFn: () => fetchProducts(filters),
    staleTime: 5 * 60 * 1000
  });
}

export function useProduct(id) {
  return useQuery({
    queryKey: ['product', id],
    queryFn: () => fetchProduct(id)
  });
}

export function useCreateProduct() {
  const queryClient = useQueryClient();
  
  return useMutation({
    mutationFn: createProduct,
    onSuccess: () => {
      queryClient.invalidateQueries(['products']);
    }
  });
}

// app/components/ProductList.jsx
function ProductList() {
  // URL state from router
  const [searchParams, setSearchParams] = useSearchParams();
  const category = searchParams.get('category') || 'all';
  const page = Number(searchParams.get('page')) || 1;
  
  // Server state from React Query
  const { data: products, isLoading } = useProducts({ category, page });
  
  // Global state from Zustand
  const { cart, addToCart } = useGlobalStore();
  
  // Local UI state
  const [selectedProduct, setSelectedProduct] = useState(null);
  
  return (
    <div>
      <CategoryFilter
        value={category}
        onChange={(cat) => setSearchParams({ category: cat, page: 1 })}
      />
      
      {isLoading ? (
        <Spinner />
      ) : (
        <div>
          {products.map(product => (
            <ProductCard
              key={product.id}
              product={product}
              onSelect={() => setSelectedProduct(product)}
              onAddToCart={() => addToCart(product)}
            />
          ))}
        </div>
      )}
      
      <Pagination
        current={page}
        onChange={(p) => setSearchParams({ category, page: p })}
      />
      
      {selectedProduct && (
        <ProductModal
          product={selectedProduct}
          onClose={() => setSelectedProduct(null)}
        />
      )}
    </div>
  );
}

// app/components/Checkout.jsx - React Hook Form for forms
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import * as z from 'zod';

const checkoutSchema = z.object({
  email: z.string().email(),
  cardNumber: z.string().length(16),
  expiryDate: z.string().regex(/^\d{2}\/\d{2}$/),
  cvv: z.string().length(3)
});

function Checkout() {
  const { cart } = useGlobalStore();
  const createOrder = useCreateOrder();
  
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting }
  } = useForm({
    resolver: zodResolver(checkoutSchema)
  });
  
  const onSubmit = async (data) => {
    await createOrder.mutateAsync({
      items: cart,
      payment: data
    });
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <input
        {...register('email')}
        placeholder="Email"
      />
      {errors.email && <span>{errors.email.message}</span>}
      
      <input
        {...register('cardNumber')}
        placeholder="Card Number"
      />
      {errors.cardNumber && <span>{errors.cardNumber.message}</span>}
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Processing...' : 'Place Order'}
      </button>
    </form>
  );
}
```

---

## **Part 5: Common Pitfalls & Best Practices**

### **Pitfall 1: Over-Globalizing State**

```javascript
// ❌ BAD: Everything in global store
const store = create((set) => ({
  // This is local UI state!
  isModalOpen: false,
  currentTab: 0,
  hoveredItem: null,
  
  // This is server state!
  users: [],
  posts: [],
  
  // This should be in URL!
  searchQuery: '',
  currentPage: 1
}));

// ✅ GOOD: Only truly global state
const store = create((set) => ({
  currentUser: null,
  theme: 'light',
  featureFlags: {},
  notifications: []
}));
```

### **Pitfall 2: Not Normalizing Related Data**

```javascript
// ❌ BAD: Nested, duplicated data
const state = {
  posts: [
    {
      id: 1,
      title: 'Hello',
      author: { id: 1, name: 'John', email: 'john@example.com' },
      comments: [
        {
          id: 1,
          text: 'Great!',
          author: { id: 1, name: 'John', email: 'john@example.com' } // Duplicate!
        }
      ]
    }
  ]
};

// Updating John's email requires updating multiple places!

// ✅ GOOD: Normalized structure
const state = {
  users: {
    1: { id: 1, name: 'John', email: 'john@example.com' }
  },
  posts: {
    1: { id: 1, title: 'Hello', authorId: 1, commentIds: [1] }
  },
  comments: {
    1: { id: 1, text: 'Great!', authorId: 1 }
  }
};

// Single source of truth for each entity!

// Use normalizr library
import { normalize, schema } from 'normalizr';

const user = new schema.Entity('users');
const comment = new schema.Entity('comments', { author: user });
const post = new schema.Entity('posts', {
  author: user,
  comments: [comment]
});

const normalizedData = normalize(originalData, [post]);
```

### **Pitfall 3: Ignoring Selector Performance**

```javascript
// ❌ BAD: Creating new arrays in selector
function TodoList() {
  // New array every render, even if todos haven't changed!
  const completedTodos = useSelector(state =>
    state.todos.filter(t => t.completed)
  );
}

// ✅ GOOD: Memoized selector
import { createSelector } from 'reselect';

const selectCompletedTodos = createSelector(
  state => state.todos,
  todos => todos.filter(t => t.completed)
  // Only recalculates if todos array changed
);

function TodoList() {
  const completedTodos = useSelector(selectCompletedTodos);
}

// ✅ BETTER: Use library helpers
import { useSelector, shallowEqual } from 'react-redux';

function TodoList() {
  const completedTodos = useSelector(
    state => state.todos.filter(t => t.completed),
    shallowEqual // Don't re-render if array contents are same
  );
}
```

### **Pitfall 4: Mixing Server and Client State**

```javascript
// ❌ BAD: Server data in Redux
const usersSlice = createSlice({
  name: 'users',
  initialState: {
    list: [],
    loading: false,
    error: null
  },
  reducers: {
    // Manually managing server state 😢
  }
});

// ✅ GOOD: Separate concerns
// Server state → React Query
const { data: users } = useQuery('users', fetchUsers);

// Client state → Redux/Zustand
const { currentUserId } = useGlobalStore();
```

---

## **Interview Wrap-Up Points**

### **The Complete Answer Structure**

**Opening:**
*"State management in large applications requires a multi-layered approach. I start by categorizing state—server data, URL state, global app state, form state, and local component state—because each category has different characteristics and optimal solutions."*

**Middle (pick 2-3 relevant tools):**
*"For server data, React Query is my go-to because it handles caching, revalidation, and synchronization automatically. For global app state, I prefer Zustand for its simplicity or Redux Toolkit for complex applications that need time-travel debugging. I keep state as local as possible and only lift to Context or global stores when necessary."*

**Closing:**
*"The key is using the right tool for each type of state rather than forcing everything through a single solution. This keeps the architecture maintainable, performant, and easy to reason about."*

### **Follow-Up Questions You Might Get**

**Q: "When would you choose Redux over Zustand?"**
**A:** *"Redux when you need DevTools time-travel debugging, complex middleware chains, or have a team already experienced with Redux patterns. Zustand for faster development, less boilerplate, and when you don't need those Redux-specific features."*

**Q: "How do you handle auth state?"**
**A:** *"Auth state is interesting because it's both server and client state. I store the auth token in Zustand or Context (persisted to localStorage), but use React Query for fetching the current user profile. The token rarely changes, but user data might, so this separation works well."*

**Q: "What about performance with Context?"**
**A:** *"Context's main limitation is that all consumers re-render when any value changes. I mitigate this by splitting contexts by update frequency, using composition to avoid prop drilling, or using a library like Jotai for fine-grained subscriptions."*

**Q: "How do you prevent prop drilling?"**
**A:** *"First, I question if the state needs to be shared—often it doesn't. If it does, I use composition to pass components as props rather than drilling through intermediates. For truly global state, Context or a global store. The goal is to keep state as local as possible."*

This comprehensive approach will demonstrate senior-level thinking about state management!
