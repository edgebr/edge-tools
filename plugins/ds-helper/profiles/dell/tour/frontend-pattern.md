# Frontend: Dell Design System (DDS v3)

**Quando aplicar:** Qualquer atividade que toque UI Angular (telas, componentes, estilo, layout). Carregar em SPEC/PLAN (escolher o componente e o token certos) e em IMPLEMENT.

**Fonte canônica (viva):** Storybook Angular `https://angular.delldesignsystem.com` (API real dos componentes) · docs `https://www.delldesignsystem.com` (tokens/foundations) · Figma UI Kit v3. Referência local detalhada: `MAPEAMENTO_DDS_v3.md`.

> Este pattern **não congela** a API dos componentes — para selector/`@Input`/`@Output` exatos, consultar o Storybook Angular.
>
> ⚠️ Assume **`@dds/angular` v3**. Antes de valer como regra: confirmar a versão instalada + como os tokens são importados (`angular.json`/CSS). Sinais de paridade em evolução: **Table** ainda sem implementação Angular v3; foundations **Color** e **Spacing** marcados "in progress" no site.

## Ferramentas DDS por fase

Três capacidades cobrem o ciclo — o mentor deve puxar a certa conforme a fase:

| Fase | Ferramenta | Papel |
|---|---|---|
| SPEC / design-time | `design-builder` (skill `/edge-dds:design-builder`) | Cria a tela no Figma vinculada **estritamente** ao DDS V3 (só instâncias/variáveis da biblioteca). Vira a referência de UI da spec. |
| IMPLEMENT | `design-assist` *(planejada)* | Traduz o nó do Figma em código: extrai componentes/tokens (`get_design_context` / `get_variable_defs`), mapeia via **Code Connect** e scaffolda o Angular já importando `@dds/angular` e amarrando `var(--dds-*)`. |
| REVIEW / quality-gate | `figma-fidelity` (skill) | Valida que o código usa os mesmos tokens do design (CAPTURAR o snapshot, VERIFICAR headless). |

> `design-builder` e `figma-fidelity` já existem; `design-assist` é o **elo ativo ainda a construir**. Enquanto não existir, no IMPLEMENT use as **Regras** e **Tokens** abaixo como guia manual (escolher o componente/token certos) e feche com `figma-fidelity` no REVIEW.

## Regras

1. **Componente do DDS → importar de `@dds/angular`, nunca recriar.** Só lógica de negócio, configuração e integração.
2. **Estilo → só token**, nunca valor literal. Proibido `#hex`/`rgb()` e `px` mágico; usar `var(--dds-...)`.
3. **Angular novo:** control flow nativo `@for`/`@if`, standalone `imports`, `signal()`. Sem `*ngFor`/`*ngIf`/`CommonModule` para fluxo.
4. **Configurar por props/variants** do componente; não sobrescrever o estilo interno dele.
5. **Responsivo mobile-first** nos breakpoints do DDS; **a11y AA** (foco visível, teclado, hierarquia de heading, `prefers-reduced-motion`).

## Tokens (semânticos — usar estes; nível global é interno)

Convenção: `--dds-{category}-{type}-{variant}--{state}`.

- **Cor:** texto `--dds-color-text-{primary|secondary|tertiary|inactive|neutral|brand}` e `--dds-color-text-{brand|success|warning|error}-{rest|hover|pressed}`; superfície `--dds-color-background-surface-{primary|secondary|tertiary|inactive|neutral|transparent|overlay}`; fundo `--dds-color-background-{brand|neutral|info|success|warning|error}-{base|subtle}-{rest|hover|pressed}`; borda `--dds-color-border-{intent}-{base|subtle}`.
- **Espaçamento (base 4px):** `--dds-spacing-xs`=4 · `sm`=8 · `md`=12 · `lg`=16 · `xl`=24 · `2xl`=32 · `3xl`=40 · `4xl`=48 · `5xl`=64 · `6xl`=80 · `7xl`=96.
- **Tipografia:** família **Roboto**; estilos do font ramp são token-based e independentes da tag — não hardcodar tamanho/fonte.
- **Estrutura:** `--dds-radius-*`, `--dds-elevation-*`, `--dds-motion-duration-*`.

## Breakpoints (mobile-first)

XS 320–479 (2 col) · S 480–767 (6) · M 768–1023 (6) · L 1024–1365 (12) · XL 1366–1583 · 2XL 1584–1919 · 3XL 1920–2559 · 4XL 2560–3839 · 5XL 3840+.

## Componentes v3

Catálogo completo no site/Storybook. Implementados em Angular v3 (entre outros): Button, Breadcrumb, Card, Modal, Drawer, Dropdown, Select, Text Input, Text Area, Checkbox, Radio Button, Switch, Date Picker, Time Picker, File Input, Search, Accordion, Tabs, Tag, Badge, Tooltip, Pagination, Progress Bar/Tracker, Side Nav, Notification, Message Bar, List, Divider, etc.
**`Table`:** sem implementação Angular v3 no Storybook — confirmar antes de planejar uma tela que dependa dela.

## Exemplo (padrão de código; props ilustrativas)

```typescript
import { Component, signal } from '@angular/core';
import { ButtonComponent } from '@dds/angular';

@Component({
  selector: 'app-exemplo',
  imports: [ButtonComponent],
  template: `
    @for (item of itens(); track item.id) {
      <dds-button (click)="acao(item)">{{ item.label }}</dds-button>
    }
  `
})
export class ExemploComponent {
  itens = signal<Item[]>([]);
  acao(i: Item): void { /* lógica de negócio */ }
}
```
> Selector/props exatos: confirmar no Storybook Angular.

## Verificação (elo com o quality-gate)

- **stylelint** (bloqueante): proíbe `#hex`/`px` fora da escala; exige `var(--dds-...)`.
- **angular-eslint:** proíbe `*ngFor`/`*ngIf`/`CommonModule` para fluxo.
- **Figma (MCP):** token aplicado vs UI Kit.
- **Playwright:** responsividade nos breakpoints + contraste/foco.

## Anti-padrões

- ❌ Recriar componente que já existe no DDS.
- ❌ `#hex`/`rgb()`/`px` mágico em vez de token.
- ❌ `*ngFor`/`*ngIf`/`CommonModule` para controle de fluxo.
- ❌ Sobrescrever o estilo interno de um componente do DDS.
- ❌ Misturar v2 e v3 na mesma experiência.
- ❌ Congelar/adivinhar a API do componente aqui em vez de consultar o Storybook.
