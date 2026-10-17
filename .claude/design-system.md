# Design system — `diceduc-web`

> Complementa [arquitetura-frontend.md](arquitetura-frontend.md) (organização de código) com a camada visual: tokens, tipografia, tema, branding por tenant e o inventário de componentes. Leia os dois antes de implementar qualquer tela.
>
> **Nota de versão:** a API exata do Tailwind v4 e do Next 16.3 deve ser validada contra a documentação oficial no momento da implementação — este documento é fonte de verdade sobre tokens/decisões visuais, não sobre a API dos frameworks (mesmo caveat de `arquitetura-frontend.md`).

## 1. Stack

- **Tailwind CSS v4** + **shadcn/ui** — a lib de componentes é copiada para o repositório (`shared/ui/`), não instalada como pacote; cada componente pode ser ajustado localmente.
- `components.json` gerado por `shadcn init` na ficha de bootstrap (`F1-E00`), apontando `shared/ui/` como diretório de componentes.
- Consulta de componentes: usar o MCP do shadcn (`search_items_in_registries`, `view_items_in_registries`, `get_item_examples_from_registries`) para confirmar nome exato e exemplo de uso antes de instalar — só funciona dentro do `diceduc-web`, depois que `components.json` existir; neste hub de documentação ele não tem o que listar.
- `get_add_command_for_items` dá o comando exato de instalação de cada componente (ex.: `npx shadcn@latest add button`).

## 2. Tokens (`globals.css`)

Origem: preset shadcn escolhido para o DICEDUC (Tailwind v4, cores em `oklch`). Copiar exatamente como está — é a fonte de verdade até o `diceduc-web` existir.

```css
:root {
  --background: oklch(1 0 0);
  --foreground: oklch(0.145 0.008 326);
  --card: oklch(1 0 0);
  --card-foreground: oklch(0.145 0.008 326);
  --popover: oklch(1 0 0);
  --popover-foreground: oklch(0.145 0.008 326);
  --primary: oklch(0.491 0.27 292.581);
  --primary-foreground: oklch(0.969 0.016 293.756);
  --secondary: oklch(0.967 0.001 286.375);
  --secondary-foreground: oklch(0.21 0.006 285.885);
  --muted: oklch(0.96 0.003 325.6);
  --muted-foreground: oklch(0.542 0.034 322.5);
  --accent: oklch(0.96 0.003 325.6);
  --accent-foreground: oklch(0.212 0.019 322.12);
  --destructive: oklch(0.577 0.245 27.325);
  --border: oklch(0.922 0.005 325.62);
  --input: oklch(0.922 0.005 325.62);
  --ring: oklch(0.711 0.019 323.02);
  --chart-1: oklch(0.811 0.111 293.571);
  --chart-2: oklch(0.606 0.25 292.717);
  --chart-3: oklch(0.541 0.281 293.009);
  --chart-4: oklch(0.491 0.27 292.581);
  --chart-5: oklch(0.432 0.232 292.759);
  --radius: 0.625rem;
  --sidebar: oklch(0.985 0 0);
  --sidebar-foreground: oklch(0.145 0.008 326);
  --sidebar-primary: oklch(0.541 0.281 293.009);
  --sidebar-primary-foreground: oklch(0.969 0.016 293.756);
  --sidebar-accent: oklch(0.96 0.003 325.6);
  --sidebar-accent-foreground: oklch(0.212 0.019 322.12);
  --sidebar-border: oklch(0.922 0.005 325.62);
  --sidebar-ring: oklch(0.711 0.019 323.02);
}

.dark {
  --background: oklch(0.145 0.008 326);
  --foreground: oklch(0.985 0 0);
  --card: oklch(0.212 0.019 322.12);
  --card-foreground: oklch(0.985 0 0);
  --popover: oklch(0.212 0.019 322.12);
  --popover-foreground: oklch(0.985 0 0);
  --primary: oklch(0.432 0.232 292.759);
  --primary-foreground: oklch(0.969 0.016 293.756);
  --secondary: oklch(0.274 0.006 286.033);
  --secondary-foreground: oklch(0.985 0 0);
  --muted: oklch(0.263 0.024 320.12);
  --muted-foreground: oklch(0.711 0.019 323.02);
  --accent: oklch(0.263 0.024 320.12);
  --accent-foreground: oklch(0.985 0 0);
  --destructive: oklch(0.704 0.191 22.216);
  --border: oklch(1 0 0 / 10%);
  --input: oklch(1 0 0 / 15%);
  --ring: oklch(0.542 0.034 322.5);
  --chart-1: oklch(0.811 0.111 293.571);
  --chart-2: oklch(0.606 0.25 292.717);
  --chart-3: oklch(0.541 0.281 293.009);
  --chart-4: oklch(0.491 0.27 292.581);
  --chart-5: oklch(0.432 0.232 292.759);
  --sidebar: oklch(0.212 0.019 322.12);
  --sidebar-foreground: oklch(0.985 0 0);
  --sidebar-primary: oklch(0.606 0.25 292.717);
  --sidebar-primary-foreground: oklch(0.969 0.016 293.756);
  --sidebar-accent: oklch(0.263 0.024 320.12);
  --sidebar-accent-foreground: oklch(0.985 0 0);
  --sidebar-border: oklch(1 0 0 / 10%);
  --sidebar-ring: oklch(0.542 0.034 322.5);
}
```

Primária violeta (hue ~292 em oklch), neutros com leve tom rosado (hue ~322–326), `--radius: 0.625rem` (10px). Os 5 `--chart-*` são o mesmo violeta em luminosidades diferentes — serve para dado sequencial, não para séries categóricas (ver "Pendências").

**Convenção:** nenhum componente usa hex/oklch literal — só os tokens acima (via classe Tailwind, ex.: `bg-primary`, `text-muted-foreground`) ou, para o que o Tailwind não cobre, `var(--nome-do-token)`.

## 3. Tipografia

`next/font/google` no `app/layout.tsx` — nunca `<link>` para `fonts.googleapis.com`: `next/font` baixa a fonte no build e serve do próprio domínio, sem requisição do navegador do usuário a um terceiro (relevante em um sistema com dado de menor sob LGPD) e sem layout shift.

- **Public Sans** — títulos e elementos de UI (botões, labels, navegação). Exposta como `--font-heading`.
- **Roboto** — corpo de texto, tabelas e formulários. Exposta como `--font-sans` (fonte padrão do body). Roboto tem `tabular-nums`, usar essa variante em toda coluna numérica (nota, valor financeiro, CPF, frequência) para os dígitos alinharem.

```ts
// app/layout.tsx (esqueleto — validar API exata do next/font na implementação)
import { Public_Sans, Roboto } from 'next/font/google'

const publicSans = Public_Sans({ subsets: ['latin'], variable: '--font-heading' })
const roboto = Roboto({ subsets: ['latin'], weight: ['400', '500', '700'], variable: '--font-sans' })
```

Tailwind mapeia `--font-sans`/`--font-heading` para as classes utilitárias (`font-sans`, `font-heading`) no `globals.css`/config.

## 4. Modo escuro

`next-themes` — segue o sistema por padrão, com toggle manual (decisão de produto):

- `ThemeProvider` (`next-themes`) em `app/layout.tsx`, `attribute="class"`, `defaultTheme="system"`, `enableSystem`.
- `<html suppressHydrationWarning>` — evita o warning de hidratação inerente a temas resolvidos no cliente.
- `ThemeToggle.tsx` em `shared/ui/` — três estados (claro/escuro/sistema) ou dois (claro/escuro), a decidir na implementação; fica no header do shell do portal (ver §6).
- O bloco `.dark` do preset (§2) já pressupõe a classe `.dark` no `<html>`, não `prefers-color-scheme` — é o `next-themes` quem aplica essa classe.

## 5. Branding por tenant

Mecanismo de tenant completo em [arquitetura-ignite.md §11](arquitetura-ignite.md#11-multi-tenancy-fundacional-desde-o-dia-zero) e [F1-E0A](docs/features/F1-E0A-tenancy/spec-tenancy.md). Aqui, só a parte visual: como `primaryColor` de `Group`/`School` convive com os tokens do preset (§2).

### Cascata

```
School.primaryColor → Group.primaryColor → preset (--primary do §2)
```

Resolvida no servidor, em `app/[group]/[school]/layout.tsx`, na mesma chamada que já busca o branding (`GET /public/branding/:groupSlug/:schoolSlug`, de `F1-E0A`) — sem requisição extra. Se nenhum dos dois tiver `primaryColor`, não sobrescreve nada e o preset prevalece.

### Tokens sobrescritos

Só os de **identidade de marca**, nunca os de legibilidade/semântica:

| Sobrescrito pelo tenant | Fica do preset |
| --- | --- |
| `--primary`, `--primary-foreground` | `--background`, `--foreground`, `--card`, `--popover`, `--secondary`, `--muted`, `--accent` |
| `--ring`, `--sidebar-primary`, `--sidebar-ring` | `--destructive`, `--border`, `--input`, `--chart-1..5`, `--sidebar` (fundo), `--sidebar-accent` |

Motivo: cor de marca é onde a personalização faz sentido (botão primário, foco, destaque da sidebar); neutros e destrutiva são sobre legibilidade e convenção (vermelho = perigo), e misturar tenant nisso quebra a leitura da interface.

### Derivação (hex do tenant → tokens)

1. Converter o hex para `oklch(L C H)`.
2. Fixar `L` (luminosidade) no intervalo `[0.35, 0.65]` — impede um roxo quase preto ou um amarelo claro demais de virar `--primary` (ilegível como fundo de botão ou com texto sobreposto).
3. `--primary-foreground`: quase-branco (`oklch(0.969 0.016 293.756)`, o mesmo do preset) se `L < 0.5`; quase-preto (`oklch(0.145 0.008 326)`) se `L >= 0.5`.
4. `--ring` = `--primary` com `L` levemente maior (foco precisa ser visível sobre fundo claro e escuro).
5. `--sidebar-primary`/`--sidebar-ring` = mesmos valores de `--primary`/`--ring`.

`ponytail:` a derivação por clamp de `L` é uma heurística simples, não um checador de contraste (WCAG) de verdade — suficiente para o MVP; se algum tenant específico ficar com contraste ruim na prática, o upgrade é rodar um cálculo de contraste real (ex.: APCA) antes de aceitar a cor no onboarding.

### Onde isso roda

No `layout.tsx` do tenant, como `style` inline no elemento wrapper (as variáveis CSS não têm como ir por classe Tailwind, já que o valor é dinâmico por tenant). **Nota de CSP:** se a Content-Security-Policy definida em `F1-E00`/`arquitetura-ignite.md §12` bloquear `style-src` inline, usar `<style nonce={nonce}>` com um nonce gerado por request, em vez de relaxar a CSP globalmente.

## 6. Shell do portal

Os tokens `--sidebar-*` do preset (§2) pressupõem um layout com sidebar — é o shell padrão de `(portal)/`:

- **Sidebar** — navegação lateral (usar o bloco `sidebar` do shadcn, não um componente próprio). Itens filtrados pelo papel do usuário, resolvido via `getSession()` (`arquitetura-frontend.md §6`) — mesmo princípio da checagem de área por papel já descrita lá.
- **Header** — nome/logo do tenant (do branding, §5), `ThemeToggle` (§4), menu da conta (nome do usuário, link para `minha-conta`, sair).
- **Área de conteúdo** — onde a página (`app/[group]/[school]/(portal)/.../page.tsx`) renderiza.
- `(public)/` (login, primeiro acesso) **não** usa este shell — layout simples, centralizado, sem sidebar.

## 7. Inventário de componentes shadcn

Lista única cobrindo as telas já especificadas nas fichas F1-E01 a F1-E10 — evita decidir componente por tela repetidamente:

`button`, `input`, `label`, `select`, `checkbox`, `radio-group`, `form` (wrapper de `react-hook-form`), `table`, `card`, `dialog`, `alert-dialog` (confirmação de ações destrutivas — ex.: excluir turma), `badge` (status: matrícula, fatura, presença), `tabs`, `sidebar`, `dropdown-menu`, `sonner` (toast de erro/sucesso), `skeleton` (estado de carregando), `avatar`, `separator`, `tooltip`, `popover`, `calendar`/`date-picker` (datas de matrícula, calendário letivo), `command` (busca de aluno/turma).

Ao implementar uma ficha, confirmar o nome exato e ver exemplo de uso com o MCP do shadcn (`search_items_in_registries`, depois `get_item_examples_from_registries`) antes de instalar — mais confiável que lembrar de memória a API do componente.

## 8. Convenções

- Nenhum hex/oklch literal em componente — só token (Tailwind ou `var(--token)`).
- Arredondamento só via `--radius` (e suas variantes Tailwind: `rounded-sm`/`rounded-md`/`rounded-lg` mapeadas a partir dele) — nunca um valor fixo solto num componente.
- Espaçamento na escala padrão do Tailwind (múltiplos de `0.25rem`) — não introduzir valores arbitrários (`p-[13px]`) sem motivo forte.
- Estados de vazio/carregando/erro de negócio usam o mesmo componente compartilhado exigido por `arquitetura-frontend.md §5` — nenhuma tela reinventa o próprio card de erro.
- Coluna numérica (nota, valor, CPF, frequência) sempre com a variante tabular da fonte (§3).

## 9. Pendências

- **Paleta categórica para gráficos:** os `--chart-1..5` do preset (§2) são o mesmo violeta em luminosidades diferentes — funciona para dado sequencial (ex.: evolução de um indicador no tempo), não para comparar categorias lado a lado (turmas, escolas, status). Entra quando a **F3-E01** (Analytics/BI) especificar os primeiros gráficos — não resolvido aqui.
- **Cor de tenant fora do padrão:** a heurística de clamp de `L` (§5) não é um checador de contraste real. Se aparecer um caso prático ruim, trocar por um cálculo de contraste (ex.: APCA) antes do onboarding aceitar a cor.
