# Servio Kids — Protótipo (contexto para Claude Code)

> Nomes de arquivo confirmados: `index.html`, `prototipo/style.css` e `prototipo/scripts.js`.

## O que é este projeto

Protótipo interativo (HTML/CSS/JS puro, sem framework, sem backend) do aplicativo **Servio Kids**, da empresa **Marati Tech** — um app mobile para prestadores de serviços recreativos infantis (pula-pula, motinha elétrica, piscina de bolinhas, tobogã inflável etc.) controlarem atendimentos por tempo, cobrança, locações para eventos e fechamento de caixa.

Não é um app real publicável — é um **simulador de interface e fluxo** usado para validar decisões de produto e design com a equipe (Edmara Braz e Luiz Filipe, sócios da Marati Tech) antes do desenvolvimento real.

## Estrutura de arquivos

```
/index.html      → estrutura (markup das telas dentro do "phone frame")
/prototipo/style.css        → todo o design system (cores, tipografia, componentes)
/prototipo/scripts.js        → toda a lógica (dados em memória, navegação, cronômetros, CRUD)
```

Tudo roda 100% no navegador. Não há build step, não há dependências de pacote — basta abrir `index.html` ou servir a pasta com um servidor estático simples (`python3 -m http.server`, `npx serve`, etc.) para visualizar.

## Stack e restrições importantes

- **Vanilla JS** — sem React/Vue/frameworks. Manter assim.
- **Fontes:** Sora (títulos, números, valores monetários) e Inter (corpo), carregadas via Google Fonts.
- **Ícones:** Font Awesome, **autohospedado** (fonte woff2 embutida em base64 no CSS) — não usar CDN externo (cdnjs), pois esse ambiente já teve problemas de bloqueio de recursos externos em sandboxes/iframes. Qualquer novo ícone deve usar uma classe já presente no pacote Font Awesome Free.
- **Imagens** (logo, fotos de brinquedos, fotos de pacotes): embutidas como `data:image/png;base64,...` diretamente no HTML/JS, para o protótipo continuar sendo um artefato autocontido e portátil. Ao adicionar novas imagens, seguir o mesmo padrão (não referenciar arquivos externos soltos, a menos que o projeto passe a ter uma pasta `/assets` — nesse caso, atualizar esta nota).
- **Sem persistência real:** todos os dados (`toys`, `pacotes`, `locacoes`, `users`, `atendimentosAtivos`) vivem em variáveis JS no `scripts.js`, resetando a cada reload. Não há API, banco de dados ou `localStorage`/`sessionStorage` (propositalmente evitado).
- **Idioma:** interface e nomes de variáveis/funções em português (`renderAtLists`, `abrirAtendimento` etc. — nem tudo é traduzido, então siga o padrão já existente arquivo por arquivo).

## Estrutura de navegação (SPA simulada)

Uma única página com várias `<section class="screen">` empilhadas dentro de um "phone frame" (mockup de celular). A navegação troca a classe `.active` entre elas via função `go(nomeDaTela)` — não há roteamento de URL real.

Telas principais: `login` (usuário + senha) → `home` → `novo` (novo atendimento) → `ativos` (atendimentos ativos + histórico) → `atendimento` (detalhe) → `pagamento` → `locacoes` → `relatorios` (fechamento de caixa) → `cadastroBrinquedos` → `cadastroPacotes` → `cadastroUsuarios` → `configuracoes`.

## Perfis de acesso

Dois perfis, cada um vinculado a um usuário específico (não um "modo" genérico):
- **Gestor:** acesso total (financeiro, locações, todos os cadastros, fechamento de caixa).
- **Monitor:** só vê e opera os próprios atendimentos (ativos e histórico), sem acesso a locações, caixa ou cadastros.

Regras de acesso são aplicadas em `applyRoleUI()` e checadas de novo dentro de `go()` (redireciona Monitor para `home` se ele tentar acessar tela restrita).

## Regras de negócio já implementadas (não reinventar)

- **Atendimento:** criado por brinquedo (por minuto, com tempo pré-definido ou personalizado) OU por pacote (tempo fixo, convertido internamente para minutos a partir de `tempoValor` + `tempoUnidade`).
- **Cronômetro real:** baseado em `Date.now()`, com `elapsedMs`/`lastResumeAt`/`paused` por atendimento — não usar `setInterval` para decrementar um contador ingênuo, isso já foi corrigido uma vez por causar dessincronização ao pausar/estender.
- **Ações no atendimento:** pausar/retomar, estender tempo (+5/10/15 min), antecipar pagamento (sem finalizar), trocar brinquedo (com opção de recalcular valor), finalizar (leva a pagamento, exceto se já pago antecipado).
- **Pagamento:** Pix (QR ilustrativo), Dinheiro (calcula troco, bloqueia confirmação se valor insuficiente), Cartão (mensagem de orientação para maquininha).
- **Alerta sonoro:** toca via Web Audio API quando o tempo de um atendimento chega a zero (uma vez só, controlado por `alertPlayed`).
- **Locações e Pacotes:** CRUD completo (cadastrar/editar/cancelar ou ativar-desativar), com edição acionada ao tocar no item da lista.
- **Fechamento de caixa e Histórico:** filtros de período (Hoje/7 dias/13 dias/Personalizado) e por monitor, com agrupamento por dia quando o período cobre mais de 1 dia.

## Design system (resumo — ver documento de especificação completo se precisar de mais detalhe)

| Papel | Cor |
|---|---|
| Fundo | `#F3F5F8` |
| Azul da marca (ações, appbar) | `#3B6FE0` |
| Navy (destaques, avatares) | `#16294D` |
| Sucesso | `#1D8A5C` |
| Alerta | `#C9861D` |
| Erro/cancelamento | `#D6484A` |

Componentes padrão já existem para: botão primário/fantasma/suave, chips (seleção única e múltipla), cards de listagem, anel de progresso (`--pct`, `--ring-color`), toggle switch, tags de status, toast de notificação, modal de confirmação (`askConfirm`/`closeConfirm` — **usar sempre isso em vez de `window.confirm`**, que é bloqueado em ambientes de preview/iframe).

## Ao fazer alterações

1. Preserve o padrão visual existente (cores, fontes, espaçamento) em vez de introduzir estilos novos — o objetivo é consistência entre telas.
2. Novas telas de cadastro devem seguir o padrão já usado (lista + botão "+ Novo" + formulário com `reg-form`, edição ao tocar no item, toggle ativo/inativo).
3. Nunca usar `alert()`/`confirm()`/`prompt()` nativos — usar o toast (`showToast`) e o modal de confirmação (`askConfirm`) já existentes.
4. Não adicionar dependências externas (CDN de ícones, frameworks JS) sem necessidade explícita — o protótipo precisa continuar funcionando offline/isolado.
5. Este é um protótipo de validação, não o produto final — funcionalidades como persistência real, autenticação segura, backend, PDF real, notificações push etc. são propositalmente simuladas/simplificadas. Não assuma que "falta implementar de verdade" seja um bug a menos que o pedido específico seja sobre isso.

## Documentos de referência

Os documentos originais de requisitos (Documento de Visão, Levantamento de Requisitos, TAP) que fundamentaram as regras de negócio deste protótipo estão no projeto Servio Kids do claude.ai. Se precisar consultar um requisito específico não coberto aqui, peça para o usuário colar o trecho relevante ou exportar o documento de especificação (`servio-kids-especificacao.md`) gerado anteriormente.
