# Vue 3 Composition API – Coding Conventions

## Stack
- Vue 3 + TypeScript + Composition API (`<script setup lang="ts">`)
- Vite as build tool

---

## File Structure

```
src/
├── assets/
├── components/
│   ├── common/        # Truly generic, reusable UI (Button, Modal, Input)
│   └── [feature]/     # Feature-scoped components (e.g. user/UserCard.vue)
├── composables/       # Reusable logic hooks (useX.ts)
├── views/             # Route-level pages (HomeView.vue)
├── router/
├── stores/            # Pinia stores
├── types/             # Shared TS interfaces/types
└── utils/             # Pure helper functions
```

**Rules:**
- Pages → `views/`, named `AbcView.vue`
- Components → `components/`, named `Abc.vue`
- Shared logic → `composables/useAbc.ts`

### Naming Conventions

| Thing | Convention | Example |
|---|---|---|
| Component files | PascalCase | `UserCard.vue` |
| View files | PascalCase + View suffix | `HomeView.vue` |
| Composables | camelCase + use prefix | `useAuth.ts` |
| Stores | camelCase + Store suffix | `useUserStore.ts` |
| Types/Interfaces | PascalCase | `interface UserProfile` |
| Template refs | camelCase + Ref suffix | `const inputRef = ref()` |

---

## Single File Component Order

Always: **Script → Template → Style**

```vue
<script setup lang="ts">
// ...
</script>

<template>
  <!-- ... -->
</template>

<style scoped>
/* ... */
</style>
```



## TypeScript

- Always use `lang="ts"` on scripts
- Define props and emits with types — no `any`
- Put shared types in `src/types/`

```ts
// types/user.ts
export interface User {
  id: number
  name: string
}
```

```vue
<script setup lang="ts">
import type { User } from '@/types/user'

const props = defineProps<{ user: User }>()
const emit = defineEmits<{ (e: 'select', id: number): void }>()
</script>
```



## Composables

Extract reusable stateful logic into composables. Keep them focused.

```ts
// composables/useCounter.ts
import { ref } from 'vue'

export function useCounter(initial = 0) {
  const count = ref(initial)
  const increment = () => count.value++
  return { count, increment }
}
```

**Rules:**
- One concern per composable
- Always prefix with `use`
- Return only what the caller needs



## Component Rules

- `defineProps` / `defineEmits` at the top, always typed
- Keep components small and focused — if it grows, split it
- Prefer props + emits over direct parent mutation
- Use `scoped` styles unless sharing is intentional

```vue
<script setup lang="ts">
const props = defineProps<{ label: string; disabled?: boolean }>()
const emit = defineEmits<{ (e: 'click'): void }>()
</script>

<template>
  <button :disabled="props.disabled" @click="emit('click')">
    {{ props.label }}
  </button>
</template>

<style scoped>
button { /* ... */ }
</style>
```


---

## KISS / YAGNI Reminders

#### DO NOT abstract until you need it twice

```vue
❌ Over-abstracted too early
<script setup lang="ts">
  // A wrapper that does nothing new
  import BaseButton from './BaseButton.vue'
  import ThemedButton from './ThemedButton.vue'
</script>

<template>
  <ThemedButton v-bind="$attrs" />
</template>

✅ Just use the component directly
<template>
  <button class="btn-primary" @click="submit">Save</button>
</template>
```

#### If a composable or component only has one use, keep logic inline
```vue
❌ Composable for a logic that's only used once
// composables/useAddTwoNumbers.ts
export function useAddTwoNumbers(a: number, b: number) {
  return a + b
}

✅ Keep it inline
<script setup lang="ts">
  const total = props.price + props.tax
</script>
```

#### Don't build for hypothetical future needs
```vue
❌ "We might support multiple payment providers someday..."
abstract class PaymentProvider { ... }
class StripeProvider extends PaymentProvider { ... }
class PaypalProvider extends PaymentProvider { ... }

✅ Keep it inline
async function processPayment(amount: number) {
  return await stripe.charge(amount)
}
```
