# Refactor Component

Guide for safely refactoring React Native components to follow project conventions, reduce complexity, and improve maintainability.

## When to Refactor

Trigger this skill when any of these conditions apply:
- **Component exceeds 200 lines** — split into focused sub-components
- **Prop drilling beyond 3 levels** — introduce context or state management
- **Duplicated logic** across 2+ components — extract a custom hook
- **Mixed concerns** — data fetching and UI rendering in the same component
- **Legacy patterns** — StyleSheet.create, TouchableOpacity, class components, raw fetch
- **On request** — when asked to "refactor", "clean up", "simplify", or "modernize"

---

## Pre-Refactoring Checklist

Complete these steps before making any changes:

1. **Verify tests exist** — if no tests cover the component, write them first
2. **Run existing tests** — confirm they pass before any changes
3. **Identify the code smell** — name the specific problem (too long, prop drilling, mixed concerns, etc.)
4. **Plan the approach** — choose which refactoring pattern(s) from this guide to apply
5. **Refactor in small steps** — run tests after each step, not just at the end
6. **Verify tests still pass** — all existing tests must pass after refactoring

---

## Extract Custom Hook

Extract state logic, effects, and data transformations out of a component into a reusable hook.

### When to Apply
- Component has 3+ `useState` or `useEffect` calls related to the same concern
- Same stateful logic is duplicated across components
- Component mixes business logic with rendering

### Before

```tsx
export function EditProfileScreen() {
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [isSubmitting, setIsSubmitting] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    httpService.get<User>('/me').then((user) => {
      setName(user.name);
      setEmail(user.email);
    });
  }, []);

  const handleSubmit = async () => {
    setIsSubmitting(true);
    setError(null);
    try {
      await httpService.put('/me', { name, email });
      navigation.goBack();
    } catch (err) {
      setError('Failed to update profile');
    } finally {
      setIsSubmitting(false);
    }
  };

  return (
    <View className="flex-1 p-4">
      {error && <Text className="text-red-500">{error}</Text>}
      <TextInput value={name} onChangeText={setName} className="mb-4 rounded-lg border p-3" />
      <TextInput value={email} onChangeText={setEmail} className="mb-4 rounded-lg border p-3" />
      <Pressable onPress={handleSubmit} disabled={isSubmitting} className="rounded-lg bg-blue-600 p-4">
        <Text className="text-center text-white">{isSubmitting ? 'Saving...' : 'Save'}</Text>
      </Pressable>
    </View>
  );
}
```

### After

```tsx
// hooks/useEditProfile.ts
interface UseEditProfileReturn {
  form: UseFormReturn<ProfileFormData>;
  onSubmit: () => void;
  isSubmitting: boolean;
  error: string | null;
}

export function useEditProfile(): UseEditProfileReturn {
  const form = useForm<ProfileFormData>({
    resolver: zodResolver(profileSchema),
  });
  const { data: user } = useQuery({
    queryKey: userKeys.me(),
    queryFn: () => httpService.get<User>('/me'),
  });
  const mutation = useMutation({
    mutationFn: (data: ProfileFormData) => httpService.put('/me', data),
    onSuccess: () => navigation.goBack(),
  });

  useEffect(() => {
    if (user) form.reset({ name: user.name, email: user.email });
  }, [user]);

  return {
    form,
    onSubmit: form.handleSubmit((data) => mutation.mutate(data)),
    isSubmitting: mutation.isPending,
    error: mutation.error ? 'Failed to update profile' : null,
  };
}

// screens/EditProfileScreen.tsx
export function EditProfileScreen() {
  const { form, onSubmit, isSubmitting, error } = useEditProfile();
  // Render only — no business logic here
}
```

**GOOD:** Hook returns a typed interface with clear, named properties.

**BAD:** Hook returns a tuple with positional values — `return [name, setName, email, setEmail, submit, error]`.

---

## Extract Sub-Components

Split a large screen into focused, single-responsibility components.

### When to Apply
- Screen component exceeds 200 lines
- Distinct visual sections that could be independently understood
- Sections that might be reused on other screens

### Before (300+ line screen)

```tsx
export function OrderDetailScreen() {
  // ... 30 lines of hooks and state
  return (
    <ScrollView className="flex-1 bg-gray-50">
      {/* 40 lines of order header */}
      <View className="bg-white p-4">
        <Text className="text-xl font-bold">{order.title}</Text>
        {/* ... status badge, dates, etc. */}
      </View>
      {/* 60 lines of item list */}
      <View className="mt-4 bg-white p-4">
        {order.items.map((item) => (
          /* ... item rendering ... */
        ))}
      </View>
      {/* 40 lines of price summary */}
      <View className="mt-4 bg-white p-4">
        {/* ... subtotal, tax, total ... */}
      </View>
      {/* 30 lines of action buttons */}
    </ScrollView>
  );
}
```

### After

```tsx
// components/OrderHeader.tsx
export interface OrderHeaderProps {
  title: string;
  status: OrderStatus;
  createdAt: string;
}

export function OrderHeader({ title, status, createdAt }: OrderHeaderProps) {
  return (
    <View className="bg-white p-4">
      <Text className="text-xl font-bold text-gray-900">{title}</Text>
      <StatusBadge status={status} />
      <Text className="mt-1 text-sm text-gray-500">{formatDate(createdAt)}</Text>
    </View>
  );
}

// components/OrderItemList.tsx
export interface OrderItemListProps {
  items: ReadonlyArray<OrderItem>;
}

export function OrderItemList({ items }: OrderItemListProps) { /* ... */ }

// components/PriceSummary.tsx
export interface PriceSummaryProps {
  subtotal: number;
  tax: number;
  total: number;
}

export function PriceSummary({ subtotal, tax, total }: PriceSummaryProps) { /* ... */ }

// screens/OrderDetailScreen.tsx
export function OrderDetailScreen() {
  const { data: order } = useOrderDetail(orderId);

  return (
    <ScrollView className="flex-1 bg-gray-50">
      <OrderHeader title={order.title} status={order.status} createdAt={order.createdAt} />
      <OrderItemList items={order.items} />
      <PriceSummary subtotal={order.subtotal} tax={order.tax} total={order.total} />
      <OrderActions orderId={order.id} status={order.status} />
    </ScrollView>
  );
}
```

**GOOD:** Each component has a single responsibility and an exported props interface.

**BAD:** Extracted components still receive the entire `order` object and pull out what they need internally, creating hidden coupling.

---

## Migrate StyleSheet to NativeWind

Replace `StyleSheet.create` patterns with NativeWind utility classes.

### Common Mappings

| StyleSheet | NativeWind |
|-----------|------------|
| `flex: 1` | `flex-1` |
| `flexDirection: 'row'` | `flex-row` |
| `alignItems: 'center'` | `items-center` |
| `justifyContent: 'center'` | `justify-center` |
| `justifyContent: 'space-between'` | `justify-between` |
| `padding: 16` | `p-4` |
| `paddingHorizontal: 12` | `px-3` |
| `paddingVertical: 8` | `py-2` |
| `margin: 16` | `m-4` |
| `marginTop: 8` | `mt-2` |
| `marginBottom: 16` | `mb-4` |
| `borderRadius: 8` | `rounded-lg` |
| `borderRadius: 9999` | `rounded-full` |
| `backgroundColor: '#fff'` | `bg-white` |
| `backgroundColor: '#f3f4f6'` | `bg-gray-100` |
| `color: '#111827'` | `text-gray-900` |
| `fontSize: 18, fontWeight: '700'` | `text-lg font-bold` |
| `fontSize: 14` | `text-sm` |
| `width: '100%'` | `w-full` |
| `position: 'absolute'` | `absolute` |
| `overflow: 'hidden'` | `overflow-hidden` |

### Step-by-Step

1. Identify `StyleSheet.create` block at the bottom of the file
2. For each style, map properties to NativeWind classes using the table above
3. Replace `style={styles.xxx}` with `className="..."` on the component
4. Delete the `StyleSheet.create` block and its import
5. Verify visually — run the app and compare before/after

**GOOD:**
```tsx
<View className="flex-1 items-center justify-center bg-white p-4">
  <Text className="text-lg font-bold text-gray-900">Welcome</Text>
</View>
```

**BAD:**
```tsx
<View style={styles.container}>
  <Text style={styles.title}>Welcome</Text>
</View>

const styles = StyleSheet.create({
  container: { flex: 1, alignItems: 'center', justifyContent: 'center', backgroundColor: '#fff', padding: 16 },
  title: { fontSize: 18, fontWeight: '700', color: '#111827' },
});
```

---

## Migrate TouchableOpacity to Pressable

Replace all `TouchableOpacity` usage with `Pressable` and NativeWind pressed states.

### Direct Replacement

**GOOD:**
```tsx
<Pressable
  className="rounded-lg bg-blue-600 px-6 py-3 active:opacity-70"
  onPress={onSubmit}
  accessibilityRole="button"
  accessibilityLabel="Submit form"
>
  <Text className="text-center font-semibold text-white">Submit</Text>
</Pressable>
```

**BAD:**
```tsx
<TouchableOpacity
  style={{ backgroundColor: '#2563eb', borderRadius: 8, paddingHorizontal: 24, paddingVertical: 12 }}
  activeOpacity={0.7}
  onPress={onSubmit}
>
  <Text style={{ color: '#fff', textAlign: 'center', fontWeight: '600' }}>Submit</Text>
</TouchableOpacity>
```

### Pressed State with Dynamic Classes

For more control over the pressed appearance:

```tsx
<Pressable
  className={({ pressed }) =>
    `rounded-lg px-6 py-3 ${pressed ? 'bg-blue-700 scale-[0.98]' : 'bg-blue-600'}`
  }
  onPress={onPress}
>
  {({ pressed }) => (
    <Text className={`text-center font-semibold ${pressed ? 'text-blue-100' : 'text-white'}`}>
      Press Me
    </Text>
  )}
</Pressable>
```

---

## Extract API Data from Redux to TanStack Query

Migrate server-side data fetching from Redux thunks/slices to TanStack Query.

### When to Apply
- Redux slice exists primarily to cache API data
- Component dispatches a thunk on mount and selects the result
- Multiple components need the same API data (TanStack Query deduplicates automatically)

### Before (Redux for server data)

```typescript
// store/slices/usersSlice.ts
const usersSlice = createSlice({
  name: 'users',
  initialState: { data: [] as User[], loading: false, error: null as string | null },
  extraReducers: (builder) => {
    builder
      .addCase(fetchUsers.pending, (state) => { state.loading = true; })
      .addCase(fetchUsers.fulfilled, (state, action) => {
        state.data = action.payload;
        state.loading = false;
      })
      .addCase(fetchUsers.rejected, (state, action) => {
        state.error = action.error.message ?? 'Failed';
        state.loading = false;
      });
  },
});

const fetchUsers = createAsyncThunk('users/fetch', () => httpService.get<User[]>('/users'));

// In component:
useEffect(() => { dispatch(fetchUsers()); }, []);
const { data, loading, error } = useSelector((state: RootState) => state.users);
```

### After (TanStack Query)

```typescript
// queries/userKeys.ts
export const userKeys = {
  all: ['users'] as const,
  lists: () => [...userKeys.all, 'list'] as const,
  detail: (id: string) => [...userKeys.all, 'detail', id] as const,
};

// hooks/useUsers.ts
export function useUsers(): UseQueryResult<User[]> {
  return useQuery({
    queryKey: userKeys.lists(),
    queryFn: () => httpService.get<User[]>('/users'),
  });
}

// In component:
const { data: users, isLoading, error } = useUsers();
```

### Migration Steps

1. Create query key factory in `queries/` directory
2. Create custom hook wrapping `useQuery` or `useMutation`
3. Replace `useSelector` + `useEffect` + `dispatch` with the new hook
4. Verify behavior matches — loading, error, and data states
5. Remove the Redux slice, thunk, and related selectors
6. Remove the slice from the Redux store configuration

**GOOD:** `useQuery({ queryKey: userKeys.detail(id), queryFn: ... })` — declarative, cached, auto-refetched.

**BAD:** `dispatch(fetchUser(id))` for server data — imperative, manual cache management, boilerplate-heavy.

---

## Reduce Prop Drilling

Choose the right pattern based on the scope and nature of the drilled state.

### Pattern 1: Lift to Redux (Global Client State)

Use when the state is needed across unrelated screens (auth, theme preference, feature flags).

```typescript
// store/slices/preferencesSlice.ts
const preferencesSlice = createSlice({
  name: 'preferences',
  initialState: { theme: 'light' as 'light' | 'dark' },
  reducers: {
    setTheme: (state, action: PayloadAction<'light' | 'dark'>) => {
      state.theme = action.payload;
    },
  },
});

// Any component, any depth:
const theme = useSelector(selectTheme);
```

### Pattern 2: React Context (Subtree-Scoped State)

Use when the state is relevant to a subtree of components, not the entire app.

```tsx
interface OrderFormContextValue {
  selectedItems: ReadonlyArray<OrderItem>;
  addItem: (item: OrderItem) => void;
  removeItem: (id: string) => void;
}

const OrderFormContext = createContext<OrderFormContextValue | null>(null);

export function useOrderForm(): OrderFormContextValue {
  const context = useContext(OrderFormContext);
  if (!context) throw new Error('useOrderForm must be used within OrderFormProvider');
  return context;
}

export function OrderFormProvider({ children }: { children: React.ReactNode }) {
  const [selectedItems, setSelectedItems] = useState<OrderItem[]>([]);
  // ... addItem, removeItem logic
  return (
    <OrderFormContext.Provider value={{ selectedItems, addItem, removeItem }}>
      {children}
    </OrderFormContext.Provider>
  );
}
```

### Pattern 3: Component Composition (Children Pattern)

Use when intermediate components are just passing props through without using them.

**GOOD:**
```tsx
// Parent controls the content, middle layer doesn't need props
function SettingsScreen() {
  const user = useCurrentUser();
  return (
    <SettingsLayout>
      <ProfileSection name={user.name} email={user.email} />
      <PreferencesSection />
    </SettingsLayout>
  );
}

function SettingsLayout({ children }: { children: React.ReactNode }) {
  return <ScrollView className="flex-1 bg-gray-50 p-4">{children}</ScrollView>;
}
```

**BAD:**
```tsx
// Prop drilling: SettingsLayout receives user just to pass it down
function SettingsScreen() {
  const user = useCurrentUser();
  return <SettingsLayout user={user} />;
}

function SettingsLayout({ user }: { user: User }) {
  return (
    <ScrollView>
      <ProfileSection user={user} />
    </ScrollView>
  );
}
```

### Choosing the Right Pattern

| Scenario | Pattern |
|----------|---------|
| Auth state, user preferences, feature flags | Redux Toolkit |
| Form state shared across a wizard/flow | React Context |
| Server data needed by multiple components | TanStack Query (not prop drilling) |
| Layout wrapper passing content through | Component composition (children) |

**GOOD:** Context for subtree-scoped state like a multi-step form.

**BAD:** Context for everything — creates unnecessary re-renders and makes data flow harder to trace.

---

## Related Resources

- [Frontend Dev Guidelines](../development/frontend-dev-guidelines.md) — master skill with full tech stack reference
- [Code Review Checklist](./code-review-checklist.md) — quality checklist for reviewing code
- [Common Patterns](../../guides/common-patterns.md) — reusable implementation patterns
- [Data Fetching Guide](../../guides/data-fetching.md) — HttpService and TanStack Query architecture
- [State Management Guide](../../guides/state-management.md) — Redux Toolkit and TanStack Query patterns
