# Componentes, props e composição

> Fonte: `/reactjs/react.dev` via `ctx7 docs` (consultado em 2026-11-10).

## Componente e props

Um componente é uma função que retorna JSX. Props são lidas como parâmetros; a
desestruturação é o padrão para declarar valores padrão.

```tsx
type AvatarProps = {
  person: { name: string; imageId: string };
  size?: number;
};

function Avatar({ person, size = 100 }: AvatarProps) {
  return (
    <img
      className="avatar"
      src={getImageUrl(person)}
      alt={person.name}
      width={size}
      height={size}
    />
  );
}
```

O valor padrão só é usado quando a prop está ausente ou é `undefined`. Passar `null` ou `0`
não aciona o default.

```tsx
<Avatar person={person} />        // size = 100
<Avatar person={person} size={0} /> // size = 0
```

Props não devem ser mutadas. Trate o objeto de props como somente leitura e derive o que
precisar durante a renderização.

## `children` e composição

`children` recebe o JSX aninhado. É o mecanismo de composição preferido: o wrapper não
precisa conhecer o conteúdo, e atualizações de estado do wrapper não forçam a re-renderização
dos filhos passados como JSX.

```tsx
function Card({ children }: { children: React.ReactNode }) {
  return (
    <div className="card">
      <div className="card-content">{children}</div>
    </div>
  );
}

export function Profile() {
  return (
    <Card>
      <h1>Photo</h1>
      <img className="avatar" src={url} alt="Aklilu Lemma" width={100} height={100} />
    </Card>
  );
}
```

Também é possível combinar `children` com props explícitas para garantir estrutura:

```tsx
function Card({ children, title }: { children: React.ReactNode; title: string }) {
  return (
    <div className="card">
      <div className="card-content">
        <h1>{title}</h1>
        {children}
      </div>
    </div>
  );
}

<Card title="Photo">
  <img className="avatar" src={url} alt="Aklilu Lemma" />
</Card>
```

Composição por `children` é preferível a herança ou a componentes que recebem dezenas de
flags de layout. Para múltiplos slots, use props nomeadas com JSX (`header={<Header />}`).

## Listas e `key`

Renderize listas com `map` e uma `key` estável, derivada dos dados (id do banco, id da
entidade). A `key` diz ao React qual item é qual entre renders.

```tsx
const listItems = products.map((product) => (
  <li key={product.id}>{product.title}</li>
));

return <ul>{listItems}</ul>;
```

Regras de identidade:

- Se uma `key` nova aparece, React cria o componente.
- Se uma `key` antiga desaparece, React destrói o componente e seu estado.
- Se duas `keys` coincidem, React move o componente correspondente.

Não faça:

```tsx
// Índice como key: quebra quando a lista é inserida, removida ou reordenada.
{items.map((item, index) => <Row key={index} {...item} />)}

// Key aleatória: nunca casa entre renders, recria DOM e perde input do usuário.
{items.map((item) => <Row key={Math.random()} {...item} />)}

// Key não chega como prop no componente.
function Row({ key, title }) {} // key é undefined
```

Faça:

```tsx
{items.map((item) => <Row key={item.id} id={item.id} title={item.title} />)}
```

`key` também pode agrupar múltiplos elementos sem wrapper usando `Fragment`:

```tsx
import { Fragment } from 'react';

return posts.map((post) => (
  <Fragment key={post.id}>
    <PostTitle title={post.title} />
    <PostBody body={post.body} />
  </Fragment>
));
```

## Renderização condicional

Use `&&` para renderizar ou não um trecho. Valores falsy não produzem saída.

```tsx
function Item({ name, isPacked }: { name: string; isPacked: boolean }) {
  return (
    <li className="item">
      {name} {isPacked && '✅'}
    </li>
  );
}
```

Para alternar entre duas árvores completas, use ternário e considere `key` distinta para
resetar estado ao trocar de entidade:

```tsx
{isPlayerA ? (
  <Counter key="Taylor" person="Taylor" />
) : (
  <Counter key="Sarah" person="Sarah" />
)}
```

## Eventos

Antes de tudo: passe a função, não a chame. `onClick={handleClick}`, nunca
`onClick={handleClick()}`.

```tsx
function Button({
  onClick,
  children,
}: {
  onClick: () => void;
  children: React.ReactNode;
}) {
  return <button onClick={onClick}>{children}</button>;
}

function PlayButton({ movieName }: { movieName: string }) {
  function handlePlayClick() {
    alert(`Playing ${movieName}!`);
  }

  return <Button onClick={handlePlayClick}>Play "{movieName}"</Button>;
}
```

Eventos borbulham para cima. Chame `e.stopPropagation()` no handler do filho para impedir
que o pai também reaja:

```tsx
function Button({ onClick, children }: { onClick: () => void; children: React.ReactNode }) {
  return (
    <button
      onClick={(e) => {
        e.stopPropagation();
        onClick();
      }}
    >
      {children}
    </button>
  );
}
```

Para comportamento padrão do navegador (submit, link), chame `e.preventDefault()`.

Resumo da documentação:

- Handlers podem ser definidos inline ou separados, dentro do componente, e acessam props.
- O pai pode declarar o handler e passá-lo como prop.
- É possível criar props de evento com nomes de domínio (`onPlayMovie`).
- Chame explicitamente a prop de handler no filho como alternativa à propagação.

## Context como componente

Crie o contexto fora de componentes. A partir do React 19, o próprio objeto Context pode ser
renderizado como provider:

```tsx
const ThemeContext = createContext('light');

function App() {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext value={theme}>
      <Page />
    </ThemeContext>
  );
}
```

O formato `<ThemeContext.Provider value={...}>` continua existindo como API legada. O
provider precisa estar **acima** de quem lê; `useContext` no mesmo componente que renderiza o
provider não enxerga o valor novo. React re-renderiza os consumidores quando o `value` muda,
comparando com `Object.is`, e `memo` não impede esse re-render.
