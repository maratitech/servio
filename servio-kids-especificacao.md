# SERVIO KIDS — Especificação de Produto, Fluxo de Telas e Design System

*Documento de referência atualizado a partir do protótipo interativo. Reflete o estado final construído nesta conversa — serve de guia para desenvolvimento e para uso no Claude Code junto com o `CLAUDE.md`.*

---

## 1. Visão Geral do Produto

**Nome:** Servio (a logomarca atual não usa mais o sufixo "Kids", removido do app e do texto de identidade visual).
**Proposta:** aplicativo mobile para prestadores de serviços recreativos infantis (pula-pula, motinha elétrica, piscina de bolinhas, tobogã inflável etc.), permitindo controlar atendimentos por tempo ou por pacote de tempo fixo, cobrança, locações para eventos e fechamento de caixa — tudo em poucos toques.
**Plataforma:** mobile, prioridade Android (conforme documento de requisitos original).
**Marca responsável:** Marati Tech.

---

## 2. Perfis de Usuário e Regras de Acesso

| Recurso / Tela | Gestor | Monitor |
|---|---|---|
| Início (resumo do dia) | ✅ com faturamento em destaque | ✅ sem faturamento, só "meus atendimentos" |
| Novo atendimento | ✅ | ✅ |
| Atendimentos ativos | ✅ todos | ✅ só os seus |
| Histórico de atendimentos | ✅ todos, com filtros | ✅ só os seus, sem filtro de monitor |
| Pausar / Estender / Trocar brinquedo / Antecipar / Finalizar | ✅ | ✅ |
| Locações (CRUD) | ✅ | ❌ aba oculta |
| Fechamento de caixa | ✅ | ❌ aba oculta |
| Cadastro de Brinquedos / Pacotes / Usuários | ✅ via menu ⋮ | ❌ |
| Configurações gerais | ✅ via menu ⋮ | ❌ |
| Receber pagamento | ✅ sempre | ⚠️ depende da configuração "Permitir que o monitor receba pagamentos" |
| Sair | ✅ via menu ⋮ | ✅ via menu ⋮ |

O cadastro de usuário tem nome, perfil, **e-mail** (que é o usuário de login), **telefone**, **data de nascimento** e **senha**. Cada usuário tem status **Ativo/Inativo** — inativos não conseguem entrar, mesmo com a senha correta. A lista de usuários permite edição ao tocar no item, no mesmo padrão dos demais cadastros.

O sistema dinâmico de perfis/permissões (`tipos_usuarios` + `acessos_sistemas` + `tipos_usuarios_acessos`) segue **fora de escopo** — os perfis continuam fixos em Gestor/Monitor.

---

## 3. Autenticação (Login)

**3.1 Tela única de login** (substituiu a seleção de perfil e o teclado de PIN, ambos removidos)
- Logomarca oficial da Servio no topo.
- Dois campos: **Usuário** (o e-mail cadastrado) e **Senha** (mascarada, com botão de olho para exibir/ocultar).
- Botão **Entrar**. Enter no campo de usuário pula para a senha; Enter na senha efetua o login.
- Caixa de acessos de demonstração no rodapé da tela, listando um gestor e um monitor ativos com suas senhas (recurso de protótipo).

**3.2 Regras de validação**
- Usuário inexistente ou senha incorreta → toast "Usuário ou senha incorretos", permanece no login.
- Usuário **inativo**, mesmo com senha correta → toast orientando a procurar um gestor.
- Sucesso → define `currentUser`/`currentRole`, aplica as regras de perfil e vai para Início.

**3.3 Logout:** menu ⋮ → "Sair" → limpa os campos e volta para o login.

---

## 4. Estrutura de Navegação

- **Barra de status:** horário + etiqueta do perfil atual (Gestor/Monitor).
- **Appbar (barra azul sólida):** presente em todas as telas internas exceto o Login. Ícone mini da marca + "Servio" (sem "Kids") em branco à esquerda; botão ⋮ à direita.
- **Menu ⋮:** Gestor vê "Cadastrar brinquedos", "Cadastrar pacotes", "Cadastrar usuários", "Configurações gerais", divisor, "Sair"; Monitor vê só "Sair".
- **Tab bar inferior:** Início / Atendim. / Locações / Caixa (Gestor); Início / Atendim. (Monitor — os outros dois ficam ocultos).
- Telas de fluxo/detalhe (detalhe do atendimento, pagamento, telas de cadastro, configurações) escondem a tab bar e usam link "‹ Voltar" no topo.

---

## 5. Regras de Negócio Detalhadas

### 5.1 Atendimento — criação

Tela **Novo Atendimento** abre com um filtro no topo: **"Brinquedos"** (padrão) ou **"Pacote"**.

- **Modo Brinquedos:** grade de brinquedos ativos (com foto real cadastrada) + seção "Tempo" com chips **10 / 15 / 20 / 30 min** e um quinto chip **"Outro"** que revela um campo numérico para o operador digitar qualquer duração. Valor estimado = preço/min do brinquedo × minutos escolhidos, recalculado ao vivo.
- **Modo Pacote:** lista os pacotes que estão **Ativos** e com a flag **"Exibir pacote no atendimento"** (`exibe_pacote_atendimento`) ligada — cada card mostra nome do pacote, valor, tempo (com unidade) e os brinquedos inclusos. Tempo é convertido internamente para minutos (minutos ×1, horas ×60, dias ×1440, meses ×43200).
- **Monitor responsável:** quando quem cria é um **Gestor**, aparece um seletor com "Eu (nome)" + os monitores ativos; o atendimento é gravado no nome do escolhido (padrão: o próprio gestor). O Monitor não vê esse seletor — é sempre o responsável pelos próprios atendimentos.
- Nome da criança, **telefone do cliente** e **nome do responsável** são opcionais em ambos os modos (o responsável cobre o caso de a criança ser atendida sem o responsável presente). Telefone e responsável aparecem na tela de detalhe do atendimento, não nos cards resumidos.
- Um único toque em "Iniciar atendimento" cria o registro (`id`, `toy`, `cliente`, `op` = usuário logado, `horaInicio`, `data`, `minutos`, `valor`, `status:'ativo'`) e o cronômetro real começa a contar a partir de `Date.now()`.

### 5.2 Atendimento — ciclo de vida e ações

- **Cronômetro real:** baseado em `elapsedMs` + `lastResumeAt`, atualizado a cada 1 segundo (não é um contador ingênuo — resiste a pausar/retomar sem perder precisão).
- **Ordenação nas listas (Início e Atendimentos ativos):** os atendimentos mais perto de acabar (ou já com tempo esgotado) ficam no **topo**; os com mais tempo restante ficam mais **embaixo**. Atendimentos de pacote (Tempo Livre) sempre ficam por último, já que não competem contra o relógio.
- **Etiquetas informativas** (linha própria, abaixo de "Cliente · horário · tempo"): "Tempo finalizado" (vermelho, tempo=0), "Quase Acabando" (amarelo escuro, 1 min restante), "Pausado" (cinza, sobrepõe as anteriores). Não aparecem para atendimentos de pacote.
- **Anel de progresso:** cor dinâmica conforme tempo restante (verde → amarelo escuro → vermelho). Para atendimentos de **pacote**, o anel normal é substituído por um **círculo verde sólido com ícone de infinito e o texto "Tempo Livre"** — tanto nas listas quanto no anel grande da tela de detalhe.
- **Nome do operador** aparece em **negrito e azul** logo após o tempo, tanto nos cards das listas (Início/Atendimentos) quanto na tela de detalhe.
- **Pausar/Retomar:** disponível na tela de detalhe **e** como botão direto em cada card da lista de Atendimentos ativos (atalho, sem precisar abrir o detalhe). Ao pausar, o cronômetro congela e a etiqueta muda para "Pausado".
- **Estender tempo:** chips +5/+10/+15 min na tela de detalhe, com abertura/fechamento suave (evita "pulos" de layout) e trava contra duplo-toque acidental. Em atendimentos por tempo, estender **recalcula o valor total** (preço/min × minutos totais); pacotes/Tempo Livre não têm preço por minuto e ficam de fora dessa regra.
- **Pago Parcial:** se o atendimento já tinha pagamento antecipado e a extensão fez o valor subir, a etiqueta verde/azul "✓ Pago antecipado" vira **"Pago Parcial" em âmbar** e o detalhe passa a exibir a linha **"Restante a cobrar"**. Ao finalizar, a cobrança é apenas desse restante (`valor` total − `valorPagoAntecipado`), nunca o valor cheio de novo.
- **Pausa:** o cronômetro congela em `elapsedMs` e nada mais é somado enquanto pausado; ao retomar, `lastResumeAt` reinicia a contagem do ponto exato. O **horário de início nunca muda** após a criação — pausas, retomadas e extensões não o alteram.
- **Trocar brinquedo:** botão na tela de detalhe (oculto para atendimentos de pacote) — abre uma grade dos demais brinquedos ativos e um interruptor **"Recalcular valor"** (ligado por padrão: usa o preço do novo brinquedo × tempo total; desligado: mantém o valor original).
- **Antecipar pagamento:** leva à cobrança sem encerrar o atendimento; ao confirmar, volta ao detalhe com a etiqueta "✓ Pago antecipado" e o botão de antecipar some.
- **Finalizar:** leva à cobrança normalmente; se já pago antecipado, encerra direto sem pedir pagamento de novo. Ao finalizar, o registro muda para `status:'finalizado'`, some da lista de ativos e passa a aparecer no Histórico.
- **Alerta sonoro:** toca um bipe duplo (gerado via Web Audio API, sem arquivo externo) na primeira vez que o tempo de um atendimento chega a zero. Não repete a cada segundo, não toca para atendimentos de pacote nem pausados, e é reativado se o tempo for estendido depois de já ter tocado.

### 5.3 Pagamento (Pix / Dinheiro / Cartão de Crédito / Cartão de Débito)

Os meios seguem exatamente os 4 da modelagem: `PIX`, `CREDITO`, `DEBITO`, `DINHEIRO` — cada um é um botão próprio na grade de formas de pagamento (não há mais um "Cartão" genérico com sub-opções).

**Área de valores — apenas dois campos:**
- **Desconto:** editável, começa em **R$ 0,00** (nunca vazio).
- **Valor a pagar:** calculado ao vivo (`valor total − desconto`).

**Pagamento dividido por parcelas confirmadas:**
- Tocar em um método abre o campo **"Valor desta parcela"** (pré-preenchido com o restante) e o botão **"Adicionar parcela"** — nada entra na conta sem essa confirmação explícita.
- Cada parcela confirmada vira uma linha em **"Pagamentos registrados"** (método + valor), removível pelo "×".
- **Total já pago** fica sempre visível, mesmo em R$ 0,00. **Restante a pagar** aparece só enquanto for maior que zero — ao quitar, a linha e a grade de métodos somem.
- O botão de confirmação final só libera quando o restante zera. Em **locações**, com pelo menos uma parcela registrada e saldo em aberto, ele vira **"Registrar pagamento parcial"**.
- **Dinheiro:** o campo de valor recebido calcula o troco em cima da **parcela** (não do total) e trava a adição se o recebido for menor.
- **Pix:** QR ilustrativo com o valor da parcela. **Crédito/Débito:** orientação da maquininha.
- Os pagamentos são gravados como lista `{metodo, valor}`; onde antes havia um método único, exibe-se a combinação legível (ex.: "Pix + Cartão de Débito").

**Autorização do gestor:** se a configuração "Permitir que o monitor receba pagamentos" estiver **desligada** e quem estiver logado for Monitor, ele preenche tudo normalmente, mas a confirmação abre um **modal de autorização** — escolher um gestor ativo + digitar a senha dele. Senha errada mantém a tela como está (toast de erro); senha correta conclui o pagamento. Para Gestor, ou com a configuração ligada, o fluxo segue direto.

### 5.4 Cadastro de Brinquedos

Campos: foto (upload com pré-visualização), nome, faixa etária, capacidade máxima, valor por minuto, valor para locação, **cor predominante** (chips com cores comuns + opção "Outra" para digitar) e ativo/inativo. Edição ao tocar no item da lista (toggle ativo/inativo tem toque isolado, não abre edição). Imagem cadastrada aparece em todas as telas que referenciam aquele brinquedo (Novo Atendimento, cards de atendimento, detalhe).

### 5.5 Cadastro de Pacotes

Campos: **imagem do pacote** (upload, aparece como miniatura nos atendimentos criados a partir dele), nome, brinquedos inclusos (seleção múltipla, com opção especial "Todos os brinquedos"), valor, **tempo numérico + unidade** (Minutos/Horas/Dias/Meses — substituiu o campo de texto livre original), interruptor **"Exibir pacote no atendimento"** (`exibe_pacote_atendimento` — só pacotes com essa flag ligada aparecem no modo "Pacote" de Novo Atendimento) e ativo/inativo. Edição ao tocar no item.

Pacote de exemplo: **"Pulseira Vale Tudo"** — R$ 30,00, acesso a "Todos os brinquedos", 8 horas, com imagem própria cadastrada, flag de exibição no atendimento ativa.

### 5.6 Locações (CRUD completo)

Tela redesenhada como "Gestão de Locações":
- Abas de filtro: **Todas / Confirmadas / Em análise**.
- **Status:** Em análise (laranja), Confirmada (verde), Cancelada (vermelho) e **Finalizado (navy)**. "Finalizado" **não é selecionável no formulário** — é atribuído automaticamente quando o pagamento total da locação é confirmado.
- **Pagamento da locação:** botão de moeda no card abre a mesma tela de cobrança dos atendimentos (mesmos 4 meios, desconto e divisão entre métodos). Pagamentos parciais são acumulados na locação, que continua no status atual até o total ser atingido; ao quitar, o status muda sozinho para "Finalizado" e o botão de pagamento some do card.
- **Campos do cliente:** nome do evento, cliente, telefone, **e-mail** (opcional), data e horário.
- **Pacote + brinquedos avulsos:** a locação pode referenciar **um pacote** (seletor "Nenhum" + pacotes ativos) e, adicionalmente, brinquedos avulsos em seleção múltipla. O valor total sugerido soma o valor do pacote com o valor de locação de cada brinquedo avulso. O card mostra o pacote e os avulsos juntos.
- Cada card tem faixa colorida à esquerda + selo de status preenchido, nome do evento (campo próprio, separado do nome do cliente), cliente, data/horário, caixa cinza com pacote/brinquedos, linha de pagamentos (quando houver) e "Valor total" em destaque.
- Botão de cancelar (ícone) em cada card ativo, com **modal de confirmação próprio do app** (não usa `window.confirm`, que é bloqueado no ambiente de preview).
- Botão flutuante azul **"+"** fixo no canto inferior direito, **acima da tab bar**, que não se move com o scroll da lista (corrigido — antes ele fazia parte da área rolável).
- Tocar no card (quando não cancelado) abre edição.

### 5.7 Histórico de Atendimentos

Dentro da aba "Atendimentos", alternância **Ativos / Histórico**. Na aba Histórico:
- **Filtro de período:** Hoje / Últimos 7 dias / Personalizado (revela dois campos de data).
- **Filtro de monitor** (só para Gestor): "Todos" + um chip dinâmico para cada monitor/gestor que tem pelo menos um atendimento finalizado.
- **Agrupamento por dia:** quando o período cobre mais de 1 dia, a lista aparece separada por cabeçalhos de data ("Hoje", "Ontem", ou dia da semana + data), cada grupo mostrando o total faturado daquele dia.
- Cada linha do histórico mostra: brinquedo, cliente, tempo utilizado, horário de início, valor final (já com desconto, sinalizado quando houver) e a lista de meios de pagamento — sem imagem nem anel (removidos para priorizar leitura rápida em texto).
- Botão **"Gerar PDF do histórico"** (simulado — gera nome de arquivo baseado no período filtrado).

### 5.8 Fechamento de Caixa (só Gestor)

Filtros de período e monitor (mesmo padrão do Histórico), indicadores (faturado, atendimentos, ticket médio), gráfico de barras por dia, tabela "Separado por monitor" e botão "Gerar PDF do relatório".

---

### 5.9 Configurações Gerais (só Gestor)

Tela acessível pelo menu ⋮, montada a partir de uma **lista de configurações** (`appSettings`) — para criar uma nova opção basta acrescentar um item com `key`, `label`, `desc` e `value`; a tela se redesenha sozinha, sem layout dedicado por opção.

| Configuração | Padrão | Efeito |
|---|---|---|
| Permitir que o monitor receba pagamentos | Ligado | Desligada, a confirmação de pagamento feita por um Monitor exige autorização de um gestor (ver 5.3) |


## 6. Fluxo de Telas (ordem de navegação)

1. **Login** (usuário + senha) → **Início**
2. **Início** → prévia de até 3 atendimentos ativos (ordenados por urgência) → "Novo atendimento" ou "ver todos"
3. **Novo Atendimento** → filtro Brinquedos/Pacote → (grade + tempo) ou (lista de pacotes) → iniciar → **Atendimentos ativos**
4. **Atendimentos** → alterna Ativos/Histórico → toque no card (ativos) → **Detalhe do Atendimento**
5. **Detalhe do Atendimento** → pausar/estender/trocar brinquedo/antecipar/finalizar → **Pagamento** (quando aplicável)
6. **Pagamento** → Pix/Dinheiro/Cartão → confirmação → retorno automático
7. **Locações** (Gestor) → filtro de status → cadastrar/editar/cancelar
8. **Fechamento de Caixa** (Gestor) → filtros → indicadores/gráfico/tabela → PDF
9. **Cadastro de Brinquedos / Pacotes / Usuários** (Gestor, via menu ⋮) → lista com edição ao toque + botão "+ Novo"
10. **Configurações gerais** (Gestor, via menu ⋮) → lista de interruptores

---

## 7. Padronização Visual (Design System)

### 7.1 Paleta de Cores

| Papel | Hex |
|---|---|
| Fundo geral | `#F3F5F8` |
| Superfície (cards) | `#FFFFFF` |
| Texto principal | `#171B22` |
| Texto secundário | `#68707C` |
| Bordas | `#E4E7EC` |
| Navy (marca, avatares, destaque) | `#16294D` |
| Azul de ação (botões, appbar, links) | `#3B6FE0` |
| Sucesso (pago, ativo, tempo livre) | `#1D8A5C` |
| Alerta (quase acabando) | `#C9861D` |
| Erro/cancelamento (tempo finalizado, cancelado) | `#D6484A` |

### 7.2 Tipografia
- **Sora** (500–800): títulos, valores monetários, números de indicadores, cronômetros.
- **Inter** (400–700): corpo de texto, labels, campos de formulário.

### 7.3 Iconografia
**Font Awesome**, autohospedado dentro do próprio HTML (fonte woff2 em base64 embutida no CSS — sem depender de CDN externo, que se mostrou bloqueado no ambiente de preview). Nenhum emoji é usado na interface funcional.

### 7.4 Componentes padrão
Botão primário (gradiente azul), botão fantasma, botão suave (pares de ação), chips de seleção única/múltipla, cards de listagem com borda e cantos arredondados, anel de progresso com cor dinâmica, círculo "Tempo Livre", toggle switch, etiquetas de status coloridas, toast de notificação (rodapé, ~2,4s), **modal de confirmação próprio** (usado no lugar de `window.confirm`/`alert`, que são bloqueados em iframes/sandboxes de preview).

### 7.5 Princípios de UX
Poucos toques para ações centrais, botões grandes para uso em pé/ao ar livre, cor sempre acompanhada de texto/ícone (nunca só cor), uma ação primária por tela, estados vazios explicativos, feedback imediato via toast, transições suaves em painéis que expandem/recolhem (evita "saltos" de layout).

---

## 8. Alinhamento com a Modelagem de Banco de Dados v1

A partir do documento `Servio_Modelagem_BD_v1`, a modelagem passa a ser a fonte de verdade para campos e regras. O protótipo foi alinhado nos seguintes pontos:

| Ajuste | Onde |
|---|---|
| Flag do pacote renomeada para "Exibir pacote no atendimento" (`exibe_pacote_atendimento`) | Cadastro de Pacotes, Novo Atendimento |
| Telefone do cliente e nome do responsável no atendimento (opcionais) | Novo Atendimento, detalhe |
| Status "Finalizado" em locações, atribuído só pelo pagamento total | Locações |
| E-mail do cliente | Cadastro de Locações |
| Pacote (referência única) + brinquedos avulsos (N:N) na mesma locação | Cadastro de Locações |
| E-mail, telefone e data de nascimento | Cadastro de Usuários |
| Cor predominante | Cadastro de Brinquedos |
| Desconto na cobrança | Pagamento |
| 4 meios de pagamento: `PIX`, `CREDITO`, `DEBITO`, `DINHEIRO` | Pagamento, histórico |
| Pagamento dividido — lista de `{metodo, valor}` no lugar de um método único | Pagamento, histórico, locações |

Rodada seguinte:

| Ajuste | Onde |
|---|---|
| Login por usuário (e-mail) e senha, no lugar da seleção de perfil + PIN | Login, Cadastro de Usuários |
| Gestor atribui o monitor responsável pelo atendimento | Novo Atendimento |
| Extensão de atendimento pago recalcula o valor e gera "Pago Parcial" | Detalhe do atendimento, Pagamento |
| Desconto padrão R$ 0,00 e apenas "Desconto" + "Valor a pagar" | Pagamento |
| Parcelas confirmadas uma a uma, com total pago sempre visível | Pagamento |
| Tela de Configurações gerais + autorização do gestor para o monitor receber | Configurações, Pagamento |

**Fora de escopo:** o sistema dinâmico de perfis e permissões (`tipos_usuarios` + `acessos_sistemas` + `tipos_usuarios_acessos`). Os perfis seguem fixos como Gestor/Monitor.

### 8.1 Correção a levar para o documento de modelagem

> A tabela `pagamentos` está com as colunas `atendimento` e `locacao` marcadas como **ambas obrigatórias (Não nulo)** simultaneamente, o que é inconsistente — um pagamento pertence a um atendimento OU a uma locação, nunca aos dois ao mesmo tempo. A correção sugerida é tornar as duas colunas **nulas**, com uma regra de negócio exigindo que **pelo menos uma** esteja preenchida — exatamente o mesmo padrão já usado corretamente na tabela `atendimentos` entre as colunas `brinquedo` e `pacote`.

---

## 9. Observações sobre o Protótipo

Este documento descreve o comportamento **tal como implementado no protótipo interativo**, que roda inteiramente no navegador sem backend. Para uma implementação de produção, ainda seriam necessários:

- Persistência real de dados (hoje vive em memória, reseta a cada reload).
- Geração real de PDF (hoje é simulada com nome de arquivo + notificação).
- QR Code Pix funcional, gerado por integração bancária real.
- Autenticação segura (hoje a senha é comparada em texto puro na memória do navegador, sem hash nem sessão real).
- Sincronização multi-dispositivo e fila de sincronização offline.
- Cálculo de "tempo livre" em pacotes de longa duração (dias/meses) considerando fechamento de caixa por turno/dia, não coberto em detalhe neste protótipo.
