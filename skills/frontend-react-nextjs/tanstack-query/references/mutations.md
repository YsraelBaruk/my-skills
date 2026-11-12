# Mutations

> Fonte: `/tanstack/query` via `ctx7 docs` (consultado em 2026-11-10).

A documentação do Context7 vem da branch `main`. Os callbacks aparecem lá na forma
`(data, variables, onMutateResult, context)`; essa assinatura é mais nova que a linha 5.69
declarada no `package.json`. Se a tipagem instalada reclamar de aridade, use a forma
`(data, variables, context)`.

## Assinatura

`useMutation` também perdeu as sobrecargas na v5: só existe a forma de objeto.

```tsx
useMutation(fn, options); // v4, removido
useMutation({ mutationFn, ...options }); // v5
```

Retorna `UseMutationResult<TData, TError, TVariables, TOnMutateResult>` com `mutate`,
`mutateAsync`, `data`, `error`, `variables`, `status`/`isPending`/`isSuccess`/`isError` e
`reset`.

```tsx
import { useMutation, useQueryClient } from "@tanstack/react-query";

function AddTodo() {
  const queryClient = useQueryClient();

  const mutation = useMutation({
    mutationFn: addTodo,
    onSuccess: () => queryClient.invalidateQueries({ queryKey: ["todos"] }),
  });

  return (
    <button onClick={() => mutation.mutate({ title: "Novo" })} disabled={mutation.isPending}>
      {mutation.isPending ? "Salvando..." : "Adicionar"}
    </button>
  );
}
```

`mutationFn` recebe as variáveis do `mutate` e deve retornar uma Promise. No tRPC deste
repo o `mutationFn` é gerado pelo próprio hook (`api.post.create.useMutation(...)`), então
normalmente só se configuram os callbacks.

## mutate vs mutateAsync

- `mutate(variables, options?)` — dispara e não retorna Promise do resultado; erros vão
  para `onError`/estado. Ideal para handlers de UI.
- `mutateAsync(variables, options?)` — retorna Promise que resolve com `data` ou rejeita
  com o erro. Ideal quando é preciso `await` (encadear, navegar, fechar modal).

```tsx
const createPost = api.post.create.useMutation();

try {
  const post = await createPost.mutateAsync({ name });
  router.push(`/posts/${post.id}`);
} catch (error) {
  // erro já passou por onError, se definido
}
```

Uma Promise rejeitada de `mutateAsync` sem `catch`/`try` gera unhandled rejection.

## Callbacks: hook-level vs per-call

`onSuccess`, `onError` e `onSettled` passados nas opções disparam para toda mutação.
Os passados como segundo argumento de `mutate`/`mutateAsync` disparam só para a última
chamada e só enquanto o componente está montado — desmontar antes de assentar remove a
assinatura e eles não rodam.

Ordem: callbacks do hook primeiro, depois os da chamada.

```tsx
useMutation({
  mutationFn: addTodo,
  onSuccess: (data, variables, context) => {
    // dispara primeiro
  },
  onError: (error, variables, context) => {},
  onSettled: (data, error, variables, context) => {},
});

mutate(todo, {
  onSuccess: (data, variables, context) => {
    // dispara depois — só para esta chamada
  },
});
```

`onMutate` roda antes da requisição e é o único lugar adequado para snapshot/update
otimista; o valor retornado é entregue como `context` (ou `onMutateResult` nas versões
novas) a `onError` e `onSettled`. Se `onSuccess`/`onSettled` retornarem Promise, ela é
aguardada antes de continuar.

## Invalidação após mutação

`invalidateQueries` marca as queries como stale **ignorando o `staleTime`** e dispara
refetch em background para os observers ativos.

```ts
queryClient.invalidateQueries(); // tudo
queryClient.invalidateQueries({ queryKey: ["todos"] }); // prefixo: pega ['todos', 5] também
```

No tRPC deste repo o equivalente é o `utils`:

```tsx
const utils = api.useUtils();

// depois de criar um post
await utils.post.invalidate(); // todas as queries do router post
await utils.post.getLatest.invalidate(); // só essa procedure
await utils.invalidate(); // todos os routers
```

Recomendação: invalide no `onSuccess`/`onSettled`, não em `mutate` direto no handler de
submit, para cobrir também quem chama a mutação de outro lugar.

## Atualização manual do cache

`setQueryData` atualiza de forma síncrona. Deve ser imutável — nunca mutar `oldData` nem o
objeto devolvido por `getQueryData`. Se o updater retornar `undefined`, o cache não é
tocado. Para atingir várias queries por prefixo de chave, use `setQueriesData`.

```ts
queryClient.setQueryData(["posts"], newPosts);

queryClient.setQueryData(["posts"], (oldPosts) => [...oldPosts, newPost]);
```

Combo comum (atualização imediata + re-sync com o servidor):

```ts
queryClient.setQueryData<TodosResponse>(["todos"], (existing) => {
  if (!existing) return existing;
  return { ...existing, items: [...existing.items, createdTodo] };
});

await queryClient.invalidateQueries({ queryKey: ["todos"] });
```

No tRPC: `utils.post.getLatest.setData(undefined, (old) => novo)` e
`utils.post.getLatest.getData()`.

## Update otimista com rollback

Sequência canônica: `cancelQueries` (refetch em voo não sobrescreve o update otimista),
snapshot via `getQueryData`, `setQueryData` com o estado desejado, snapshot retornado como
contexto, `onError` restaura o snapshot e `onSettled` invalida para reconciliar com o servidor.

```ts
const queryClient = useQueryClient();

const updateTodo = useMutation({
  mutationFn: updateTodoApi,
  onMutate: async (newTodo) => {
    // 1. cancela refetches pendentes
    await queryClient.cancelQueries({ queryKey: ["todos"] });

    // 2. snapshot do valor anterior
    const previousTodos = queryClient.getQueryData(["todos"]);

    // 3. aplica o update otimista
    queryClient.setQueryData(["todos"], (old: Todo[] | undefined) => [
      ...(old ?? []),
      newTodo,
    ]);

    // 4. devolve o snapshot para rollback
    return { previousTodos };
  },
  onError: (_err, _newTodo, context) => {
    // 5. restaura o snapshot
    queryClient.setQueryData(["todos"], context?.previousTodos);
  },
  onSettled: () => {
    // 6. sempre re-sincroniza
    queryClient.invalidateQueries({ queryKey: ["todos"] });
  },
});
```

Com o tRPC deste repo, o caminho é `utils.post.getLatest.getData()`,
`utils.post.getLatest.setData(undefined, updater)` e `utils.post.getLatest.invalidate()`,
que encapsulam `getQueryData`/`setQueryData`/`invalidateQueries` com a chave correta.

## Erros

- Estado: `mutation.error`, `mutation.isError`, `mutation.failureCount`.
- `throwOnError` (`boolean` ou `(error) => boolean`), default `false`: quando verdadeiro, o
  erro é lançado para o error boundary em vez de virar estado.
- `reset()` limpa o estado de erro para permitir nova tentativa sem remontar.
- `mutateAsync` rejeita: trate com `try/catch` ou `.catch`.

```tsx
const mutation = useMutation({
  mutationFn: savePost,
  throwOnError: (error) => error instanceof HttpError,
  onError: (error) => toast.error(error.message),
});

// limpar erro exibido
mutation.reset();
```

## Retry, scope e chave de mutação

- `retry` default é `0` para mutações (`retry: 3` reexecuta três vezes; `false` desativa);
  `retryDelay` default é backoff exponencial com teto de 30s.
- `scope: { id }` serializa mutações que compartilham o mesmo `id`, na ordem de disparo;
  sem `scope`, cada mutação roda imediatamente.
- `mutationKey` + `mutationOptions` permitem compartilhar opções entre call sites e
  inspecionar o estado com `useMutationState`.

```ts
useMutation({
  mutationKey: ["todos", "add"],
  mutationFn: addTodo,
  scope: { id: "todos" }, // evita corrida entre adds
  retry: 0,
});
```

## Callbacks globais

`MutationCache` aceita `onError`, `onSuccess` e `onSettled` para toda mutação do cache,
independentemente de quem disparou. O retorno em Promise é aguardado antes de continuar.

```ts
import { MutationCache, QueryClient } from "@tanstack/react-query";

const queryClient = new QueryClient({
  mutationCache: new MutationCache({
    onError: (error, variables, context, mutation) => {
      // log centralizado / telemetria
    },
    onSuccess: (data, variables, context, mutation) => {},
    onSettled: (data, error, variables, context, mutation) => {},
  }),
});
```

Para queries o equivalente é `QueryCache({ onError, onSuccess, onSettled })`, recebendo
`(error, query)`. Como a v5 removeu callbacks de `useQuery`, esses são os pontos de
observação globais (alternativa: tratar o erro no `useQuery`/error boundary).
