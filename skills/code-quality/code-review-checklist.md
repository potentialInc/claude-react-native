# Code Review Checklist

Structured checklist for reviewing React Native code quality, patterns compliance, and potential issues in Expo/React Native projects.

## When to Use

Use this checklist when:
- **Reviewing PRs** or code changes before merge
- **Quality audits** before app store releases
- **Code review requests** — when asked to "review code", "check quality", or "audit this"
- **Onboarding new devs** — to communicate project standards

---

## TypeScript Quality

### Strict Type Safety

- [ ] No `any` types anywhere — use `unknown` for truly unknown types, or proper generics
- [ ] Explicit return types on all exported functions and hooks
- [ ] Use `type` imports for type-only usage: `import type { User } from '@/types'`
- [ ] Zod schema inference for runtime validation types
- [ ] Strict null checks respected — no non-null assertions (`!`) without justification

**GOOD:**
```typescript
import type { User } from '@/types';

export function useUserProfile(id: string): UseQueryResult<User> {
  return useQuery({
    queryKey: userKeys.detail(id),
    queryFn: () => httpService.get<User>(`/users/${id}`),
  });
}
```

**BAD:**
```typescript
import { User } from '@/types'; // not a type import

export function useUserProfile(id: any) { // any type, no return type
  return useQuery({
    queryKey: ['user', id],
    queryFn: () => fetch(`/users/${id}`).then((r: any) => r.json()),
  });
}
```

### Zod Schema Inference

- [ ] API response shapes validated with Zod at runtime
- [ ] Types inferred from schemas, not manually duplicated

**GOOD:**
```typescript
const UserSchema = z.object({
  id: z.string().uuid(),
  name: z.string().min(1),
  email: z.string().email(),
});

type User = z.infer<typeof UserSchema>;
```

**BAD:**
```typescript
// Manual type that can drift from API reality
interface User {
  id: string;
  name: string;
  email: string;
}
```

---

## Component Patterns

- [ ] Function components only — no class components
- [ ] `Pressable` used for all touchable elements — never `TouchableOpacity`
- [ ] NativeWind `className` for styling — never `StyleSheet.create` as the primary approach
- [ ] React Native Paper for Material Design components (Button, Card, TextInput, etc.)
- [ ] Props interface defined and exported for reusable components
- [ ] No inline style objects — use NativeWind utility classes

**GOOD:**
```tsx
import { Pressable, Text, View } from 'react-native';

export interface ProfileCardProps {
  name: string;
  avatarUrl: string;
  onPress: () => void;
}

export function ProfileCard({ name, avatarUrl, onPress }: ProfileCardProps) {
  return (
    <Pressable
      className="flex-row items-center rounded-xl bg-white p-4 shadow-sm"
      onPress={onPress}
      accessibilityRole="button"
      accessibilityLabel={`View ${name}'s profile`}
    >
      <Image source={{ uri: avatarUrl }} className="h-12 w-12 rounded-full" />
      <Text className="ml-3 text-base font-semibold text-gray-900">{name}</Text>
    </Pressable>
  );
}
```

**BAD:**
```tsx
// Class component, TouchableOpacity, StyleSheet, no types
class ProfileCard extends Component {
  render() {
    return (
      <TouchableOpacity style={styles.card} onPress={this.props.onPress}>
        <Image source={{ uri: this.props.avatarUrl }} style={styles.avatar} />
        <Text style={styles.name}>{this.props.name}</Text>
      </TouchableOpacity>
    );
  }
}
const styles = StyleSheet.create({ /* ... */ });
```

---

## Performance Review

- [ ] `React.memo` applied to expensive pure components that receive stable props
- [ ] `useMemo` / `useCallback` used with correct dependency arrays (not over-applied)
- [ ] `FlatList` used for long lists — never `ScrollView` with `.map()`
- [ ] `FlatList` has `keyExtractor` and `getItemLayout` where item height is fixed
- [ ] No anonymous arrow functions in `FlatList` `renderItem`
- [ ] Images optimized — proper sizing, `cachePolicy`, and `placeholder` where supported
- [ ] No unnecessary re-renders from parent state updates

**GOOD:**
```tsx
const renderItem = useCallback(({ item }: { item: Product }) => (
  <ProductCard product={item} onPress={handlePress} />
), [handlePress]);

<FlatList
  data={products}
  renderItem={renderItem}
  keyExtractor={item => item.id}
  getItemLayout={(_, index) => ({ length: 88, offset: 88 * index, index })}
/>
```

**BAD:**
```tsx
<ScrollView>
  {products.map((item) => (
    <ProductCard
      key={item.id}
      product={item}
      onPress={() => navigate('Product', { id: item.id })} // new fn each render
    />
  ))}
</ScrollView>
```

---

## Accessibility Check

- [ ] `accessibilityLabel` on all interactive elements (Pressable, buttons, icons)
- [ ] `accessibilityRole` set correctly (`button`, `link`, `header`, `image`, `text`, etc.)
- [ ] `accessibilityHint` provided for non-obvious actions
- [ ] Touch targets are minimum 44x44pt (use `hitSlop` or padding if needed)
- [ ] Sufficient color contrast for all text (4.5:1 for normal, 3:1 for large text)
- [ ] Screen reader flow makes logical sense (check element order)

**GOOD:**
```tsx
<Pressable
  className="min-h-[44px] min-w-[44px] items-center justify-center rounded-lg bg-blue-600 px-6 py-3"
  onPress={onSubmit}
  accessibilityRole="button"
  accessibilityLabel="Submit order"
  accessibilityHint="Places your order and navigates to confirmation"
>
  <Text className="text-base font-semibold text-white">Submit</Text>
</Pressable>
```

**BAD:**
```tsx
<Pressable onPress={onSubmit} className="p-1">
  <Icon name="check" size={16} />
</Pressable>
// No accessibilityLabel, tiny touch target, no role
```

---

## State Management

- [ ] **Redux Toolkit** used for client state only (auth, UI preferences, onboarding state)
- [ ] **TanStack Query** used for all server/API data — never Redux for API caching
- [ ] No prop drilling beyond 2 levels — use context or state management instead
- [ ] `useSelector` with granular selectors — never select entire Redux state slice
- [ ] Query keys follow a factory pattern for consistency

**GOOD:**
```typescript
// Server data with TanStack Query
const { data: user } = useQuery({
  queryKey: userKeys.detail(userId),
  queryFn: () => userService.getById(userId),
});

// Client state with Redux
const isOnboarded = useSelector(selectIsOnboarded);
```

**BAD:**
```typescript
// Server data in Redux — wrong layer
dispatch(fetchUser(userId));
const user = useSelector((state: RootState) => state.users); // selects entire slice
```

---

## Navigation

- [ ] Type-safe navigation params defined in `RootStackParamList`
- [ ] Deep link configuration present if the screen is linkable
- [ ] Screen names defined as constants, not inline strings
- [ ] Navigation actions use typed hooks (`useNavigation<ScreenNavigationProp>()`)

---

## Error Handling

- [ ] `try/catch` on every async operation
- [ ] Error boundaries at screen or section level to prevent full-app crashes
- [ ] Network errors handled in httpService interceptors (401 redirect, retry logic)
- [ ] User-facing error messages are human-readable — never expose raw error strings or stack traces
- [ ] Loading and error states handled in UI (not just happy path)

**GOOD:**
```tsx
const { data, isLoading, error } = useQuery({ queryKey: ['orders'], queryFn: fetchOrders });

if (isLoading) return <LoadingSpinner />;
if (error) return <ErrorState message="Unable to load orders" onRetry={refetch} />;
return <OrderList orders={data} />;
```

**BAD:**
```tsx
const [data, setData] = useState(null);
useEffect(() => {
  fetchOrders().then(setData); // no error handling, no loading state
}, []);
return <OrderList orders={data} />; // crashes if data is null
```

---

## Security

- [ ] No secrets, API keys, or tokens hardcoded in source — use environment variables
- [ ] Sensitive tokens stored in MMKV with encryption or Expo SecureStore
- [ ] All user-facing form inputs validated with Zod schemas
- [ ] No `eval()`, `new Function()`, or dynamic code execution
- [ ] API requests go through httpService — never raw `fetch()` or direct `axios`

---

## Testing

- [ ] Test files exist for all new components and custom hooks
- [ ] Tests cover the happy path and at least one error/edge case
- [ ] No snapshot tests for complex components — use behavior-driven tests with RNTL
- [ ] Mocks are minimal — prefer testing real behavior over mocking internals
- [ ] Tests do not depend on implementation details (avoid testing state directly)

---

## Severity Levels

Use these severity levels when reporting review findings:

| Level | Label | Meaning | Action |
|-------|-------|---------|--------|
| P0 | **Critical** | Security vulnerability, crash, data loss, `any` types in shared code | Must fix before merge |
| P1 | **Warning** | Performance issue, missing error handling, accessibility gap, wrong state layer | Should fix before merge |
| P2 | **Suggestion** | Code style, naming, minor optimization, better pattern available | Nice to have, fix when convenient |

---

## Review Report Template

Present findings using this format:

```markdown
## Code Review: [PR Title / Component Name]

**Reviewer:** Claude
**Date:** YYYY-MM-DD
**Overall:** Approved / Changes Requested / Needs Discussion

### Summary
[1-2 sentence summary of what the PR does and overall quality]

### Findings

| # | Severity | File | Line | Issue | Suggestion |
|---|----------|------|------|-------|------------|
| 1 | Critical | `UserProfile.tsx` | 42 | Uses `any` for API response | Use `User` type from `@/types` |
| 2 | Warning | `OrderList.tsx` | 18 | Missing error state | Add `<ErrorState>` fallback |
| 3 | Suggestion | `useAuth.ts` | 7 | Could use type import | `import type { AuthState }` |

### What Looks Good
- [Positive observations about the code]

### Checklist Summary
- TypeScript Quality: X/5
- Component Patterns: X/6
- Performance: X/6
- Accessibility: X/5
- State Management: X/4
- Error Handling: X/4
- Security: X/5
- Testing: X/4
```

---

## Related Resources

- [Frontend Dev Guidelines](../development/frontend-dev-guidelines.md) — master skill with full tech stack reference
- [Organize Types](./organize-types.md) — TypeScript type organization patterns
- [Refactor Component](./refactor-component.md) — step-by-step refactoring patterns
- [Common Patterns](../../guides/common-patterns.md) — reusable implementation patterns
- [Data Fetching Guide](../../guides/data-fetching.md) — HttpService and TanStack Query architecture
