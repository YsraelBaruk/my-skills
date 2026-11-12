# Hooks — assinaturas, regras e exemplos

> Fonte: `/reactjs/react.dev` via `ctx7 docs` (consultado em 2026-11-10).

## Regras dos Hooks

Chame Hooks no topo do componente ou de outro Hook. Não é suportado: dentro de condições ou
laços; depois de `return` condicional; em event handlers; em class components; em callbacks de
`useMemo`, `useReducer` ou `useEffect`; em `try`/`catch`/`finally`. Se precisar de Hook
condicional, extraia um componente.

## `useState`

```ts
const [state, setState] = useState(initialState);
```

- `setState(nextState)` aceita valor ou updater `(prev) => next`.
- Só altera no próximo render; ler a variável logo após o `set` retorna o valor antigo.
- Valor idêntico (`Object.is`) pula o re-render.
- Atualizações são agrupadas; a tela muda depois que todos os handlers do evento rodam.
- Identidade de `setState` é estável; pode ser omitida de dependências.
- Em StrictMode, updaters rodam duas vezes no desenvolvimento.

```tsx
setNumber((n) => n + 1); // cada updater vê o valor pendente
const [todos, setTodos] = useState(createInitialTodos); // chama só na inicialização
```

Estado com objeto/array é imutável: prefira `concat`, spread, `filter`, `map`, `slice`; evite
`push`, `pop`, `splice`, `sort` e atribuição por índice.

## `useReducer`

```ts
const [state, dispatch] = useReducer(reducer, initialArg);
```

Reducer recebe `(state, action)` e retorna o próximo estado; action desconhecida deve lançar
erro. Use quando há várias transições relacionadas ou atualização complexa.

## `useContext`

```ts
const value = useContext(SomeContext);
```

O provider precisa estar acima de quem lê; no mesmo componente não é visto. O valor é
comparado com `Object.is`: mudou, todos os consumidores re-renderizam, e `memo` não evita
isso. Contextos duplicados no bundle (symlinks) quebram a leitura. Para o padrão
provider + reducer com hooks de acesso (`useTasks`, `useTasksDispatch`), veja
`references/components.md`.

## `useRef`

```ts
const ref = useRef(initialValue); // { current: initialValue }
```

Retorna o mesmo objeto em todos os renders; alterar `current` não dispara re-render. Use para
intervalos, DOM e valores que não aparecem na tela. Não leia nem escreva `ref.current` durante
a renderização. Em React 19, `ref` é prop normal; `forwardRef` não é necessário.

## `useEffect`

```ts
useEffect(setup, dependencies?);
```

`setup` roda após o commit e pode retornar cleanup. React roda cleanup antes do próximo setup
e no unmount. Dependências são comparadas com `Object.is` e precisam ter tamanho constante.

```tsx
useEffect(() => {
  const connection = createConnection(serverUrl, roomId);
  connection.connect();
  return () => connection.disconnect();
}, [serverUrl, roomId]);
```

- Sem sistema externo, provavelmente não precisa de Effect.
- StrictMode roda um ciclo extra setup+cleanup em desenvolvimento; implemente o cleanup.
- Objetos/funções criados no componente como dependência fazem o Effect rodar demais.
- Effects só rodam no cliente, nunca na renderização do servidor.
- Se for visual e causar flicker, avalie `useLayoutEffect`.

## `useLayoutEffect`

```ts
useLayoutEffect(setup, dependencies?);
```

Roda de forma síncrona após o commit e antes do browser pintar; use para medir layout. Bloqueia
a pintura, então prefira `useEffect` quando possível.

```tsx
useLayoutEffect(() => {
  const { height } = ref.current!.getBoundingClientRect();
  setTooltipHeight(height);
}, []);
```

## `useEffectEvent`

```ts
const onSomething = useEffectEvent(callback);
```

Sempre vê os valores mais recentes de props/estado, sem re-sincronizar o Effect, e não entra nas
dependências.

```tsx
const onMessage = useEffectEvent((received: string) => {
  setMessages((msgs) => [...msgs, received]);
  if (!isMuted) playSound();
});
useEffect(() => {
  const connection = createConnection();
  connection.on('message', onMessage);
  connection.connect();
  return () => connection.disconnect();
}, []); // onMessage não é dependência
```

Só pode ser chamado de dentro de Effects ou de outro Effect Event; não chame no render nem
passe para outros componentes/Hooks. Não use para esconder dependências reais.

## `useMemo`, `useCallback` e `memo`

```ts
const value = useMemo(() => compute(a, b), [a, b]);
const fn = useCallback((x) => doSomething(x), [dep]);
const Memoized = memo(Component, arePropsEqual?);
```

`useMemo` cacheia o resultado; `useCallback`, a função. Só compensa quando o cálculo é lento e
muda pouco, quando o valor/função vai para um `memo`, ou quando é dependência de outro Hook.
Fora disso, o custo é legibilidade; uma dependência "sempre nova" quebra a memoização inteira.

```tsx
const requirements = useMemo(() => computeRequirements(product), [product]);
const handleSubmit = useCallback(
  (orderDetails: OrderDetails) => post(`/product/${productId}/buy`, { referrer, orderDetails }),
  [productId, referrer],
);
```

Com `arePropsEqual` customizado, compare **todas** as props, inclusive funções; caso contrário
o componente vê props antigas dentro de closures. Evite igualdade profunda.

## `useTransition` e `startTransition`

```ts
const [isPending, startTransition] = useTransition();
startTransition(action);
```

Marca atualizações como Transitions não bloqueantes. O `startTransition` global não expõe
`isPending` e pode ser chamado fora de componentes.

- Não serve para controlar `value` de input de texto.
- Atualizações depois de `await` não são marcadas; envolva-as em outro `startTransition`.
- Transitions podem ser interrompidas por atualizações urgentes.
- Múltiplas Transitions simultâneas são agrupadas.

## `useDeferredValue`

```ts
const deferredValue = useDeferredValue(value, initialValue?);
```

Adia uma parte da UI. No primeiro render usa `initialValue` (ou o próprio valor); nas
atualizações tenta o valor antigo e depois re-renderiza em background. Dentro de uma
Transition sempre retorna o valor novo; o re-render é interrompível; integra com `<Suspense>`
mostrando o valor antigo; não evita requisições sozinho nem adiciona atraso fixo.

## `useId`

```ts
const id = useId();
```

Gera IDs únicos para atributos de acessibilidade. Requer árvore idêntica no servidor e no
cliente na hidratação; não pode ser usado em async Server Components. Para `key` de lista, use
ids dos dados.

## `useImperativeHandle`

```ts
useImperativeHandle(ref, () => ({ focus() {} }), dependencies?);
```

Expõe um objeto customizado ao `ref` do pai, em vez do nó DOM inteiro:

```tsx
function MyInput({ ref }: { ref: React.Ref<{ focus: () => void }> }) {
  const inputRef = useRef<HTMLInputElement>(null);
  useImperativeHandle(ref, () => ({ focus: () => inputRef.current?.focus() }));
  return <input ref={inputRef} />;
}
```

Não use ref para o que pode ser prop: prefira `<Modal isOpen={isOpen} />` a um handle
`{ open, close }`. Refs servem para foco, scroll, seleção de texto e animação.

## `useActionState`

```ts
const [state, dispatchAction, isPending] = useActionState(
  reducerAction,
  initialState,
  permalink?,
);
```

`reducerAction` recebe o estado anterior e o payload; o retorno vira o próximo estado. Passe
`dispatchAction` como `action` de `<form>`; React embrulha o envio em Transition.

Chamadas são enfileiradas e cada uma recebe o resultado da anterior; `dispatchAction` tem
identidade estável; precisa ser chamado de dentro de uma Action (ou `startTransition`); erro
cancela as ações enfileiradas e sobe para o Error Boundary.

## `useOptimistic`

```ts
const [optimisticState, setOptimistic] = useOptimistic(value, reducer?);
```

Mostra um estado provisório enquanto uma Action está pendente. Sem `reducer`, o setter recebe
o valor otimista; com `reducer`, recebe `(currentState, action)` e deve ser puro. Chame o
setter dentro de uma Action/Transition, antes do `await` da operação real.

## `useFormStatus` (react-dom)

```tsx
const { pending } = useFormStatus();
return <button type="submit" disabled={pending}>{pending ? 'Submitting...' : 'Submit'}</button>;
```

Precisa estar dentro do `<form>` cujo status se quer ler.

## `use(resource)`

```ts
const value = use(resource);
```

- `use(promise)` suspende até resolver e exige `<Suspense>` acima.
- `use(Context)` lê contexto e pode ser chamado em `if` e laços (diferente de `useContext`).
- Para Promise criada no render: `ctx7 docs /reactjs/react.dev "use(promise)"`.
