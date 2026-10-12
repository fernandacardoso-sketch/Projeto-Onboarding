# Distribuidor de Carteira · Atribuição, Agendamento e Fechamento de Onboarding

**Projeto:** New Business / Sales Ops · Hotmart
**Última atualização:** 12/10/2026
**Responsável:** Fernanda Cardoso

Automação que substitui a atribuição manual de Onboarder, o agendamento da reunião de welcome e a marcação de Closed Won.
Este arquivo **não vai para o GitHub Pages** (o `.gitignore` só libera `index-fer.html` e `painel.html`) e não deve conter senha, token ou chave de API.

---

## 1. Como funciona

O Closer abre o **Onboarding Assignment** (`index-fer.html`), entra com e-mail e senha e:

1. **Escolhe a Opportunity** no **Funil**, um kanban que espelha a visão "Funil" do Salesforce: uma coluna por estágio (Issues identified, SQL, Offer presented, Contract Sent, Closed Won), com o valor total de cada coluna, e busca em tempo real por nome, cliente ou Hotmart ID. Clicar num card elegível abre um **painel lateral** com o resto do fluxo. Se o **Amount 12 months** estiver vazio ou zero, o painel pede o valor antes de atribuir.
2. **Atribui:** o sistema escolhe o Onboarder (regras na seção 4), grava `Onboarding_Owner__c` na Opportunity e atualiza o cache.
3. **Agenda o welcome:** escolhe um horário livre na agenda do Onboarder; o evento é criado com o Closer e o cliente como convidados.
4. **Preenche o que faltar** (formulário com os campos obrigatórios e a Conta da Opportunity) e **marca Closed Won**.
5. No Closed Won, o **Salesforce cria sozinho o `Onboarding__c`** do Onboarder, e ele entra na carteira no próximo Sync.

Quem atribui é o sistema: o Closer não escolhe o Onboarder.

### Stack
- **n8n Cloud** (`hotmart.app.n8n.cloud`, projeto *Sales Agent*): back-end, todos os webhooks.
- **GitHub Pages:** só o front-end, repositório público `fernandacardoso-sketch/Projeto-Onboarding`.
  - Onboarding Assignment: `https://fernandacardoso-sketch.github.io/Projeto-Onboarding/index-fer.html`
  - Painel: `https://fernandacardoso-sketch.github.io/Projeto-Onboarding/painel.html`
- **Salesforce:** origem dos dados e destino das gravações.
- **Google Sheets** (planilha `188cY1Ti6anYxcehneMgZpXeqOXMtQYSrK2fjIzzrqls`): abas `PODS`, `Cache` e `Hist`.
- **Google Sheets de acessos** (planilha própria, restrita à Fernanda): aba `Acessos`.
- **Upstash Redis** (banco *Projeto Onboarding*, só da Fernanda): sessões, limite de tentativas e log de login.
- **Google Calendar:** uma única conta operadora, `sales.bizops@hotmart.com`.

---

## 2. Workflows (n8n)

| # | Nome | Endpoint | Quem chama | O que faz |
|---|---|---|---|---|
| 0 | Login | `POST /carteira-login` | tela | Valida e-mail e senha, cria a sessão (8h) |
| 1 | Sync Diário | agendado, 06:00 BRT | n8n | Reconstrói a aba `Cache` a partir do PODS e do Salesforce |
| 2 | Webhook de Atribuição | `POST /atribuicao-onboarder` | tela | Escolhe o Onboarder, grava no Salesforce, no cache e no Hist; devolve horários |
| 3 | Agendamento de Reunião | sub-workflow (do 2) + `POST /confirmar-reuniao-onboarding` | W2 e tela | Calcula horários livres e cria o evento na agenda do Onboarder |
| 4 | Closed Won | `POST /closed-won-onboarding` | tela | Confere campos obrigatórios e muda o Stage por chamada direta à API (devolve o motivo real quando o Salesforce recusa) |
| 5 | Painel de Acompanhamento | `GET /painel-acompanhamento` | painel | Carteira por Onboarder e últimas atribuições |
| 6 | Minhas Oportunidades | `GET /minhas-oportunidades` | tela | Lista as Opportunities do Closer logado (nome, **conta**, estágio, valor, Hotmart ID) |
| 7 | Campos da Oportunidade | `GET /campos-oportunidade?opp_id=` | tela | Campos obrigatórios com tipo, opções e Conta |
| 8 | Salvar Campos | `POST /salvar-campos` | tela | Grava os campos e a Conta na Opportunity |
| 9 | Buscar Contas | `GET /buscar-contas?q=` | tela | Busca Contas por nome, e-mail ou User ID |
| 10 | Carteira do Salesforce | `GET /carteira-salesforce` | admin | Prévia/validação da contagem de carteira (só leitura) |
| 11 | Testar Agendas | `GET /testar-agendas` | admin | Confere se a agenda de cada Onboarder está acessível (só leitura) |
| 12 | _(não usado)_ | | | Número reservado |
| 13 | Diagnóstico Closed Won | `GET /diagnostico-closed-won?opp_id=` | admin | Registro completo da Opportunity (tenta ler as regras de validação; a credencial atual não tem acesso) |
| 14 | Simular Etapa | `GET /simular-etapa?opp_id=&etapa=` | admin | Mostra o que o Salesforce exigiria numa etapa, **sem alterar a Opp** (transação sempre desfeita). Desative depois de usar |

Todos os webhooks, **exceto o Login**, exigem o header `Authorization: Bearer <token>` e respondem **401** sem sessão válida. O CORS é restrito a `https://fernandacardoso-sketch.github.io`.

### Checagens de dono (servidor)
- Workflows 7 e 8: só o dono da Opportunity (`OwnerEmail__c` = e-mail da sessão) mexe nela; `admin` passa.
- Workflows 2 e 4: a mesma regra (**403** "Esta Opportunity não pertence a você").
- Workflow 2: exige **Amount 12 months maior que zero** (**422**, código `amount_obrigatorio`).
- Workflow 3 (confirmar reunião) **ainda não confere dono**: ver pendências.

---

## 3. Dados

### Aba `PODS` (roster, fonte: `PODs_Q4_2026.xlsx`)
Colunas `Level | Img | POD | Email | Time`. `Time` = `Closer`, `Onboarding` ou vazio (Guardião). `Level` = `N2-N3`, `N4-N5`, `N6+` ou `Guardian`.
Níveis com **hífen simples** (`N2-N3`), nunca travessão.

| POD | Nível | Líder Closer | Líder Onboarding |
|---|---|---|---|
| 1, 2, 3 | N2-N3 | Renata | Raissa |
| 4, 5 | N4-N5 | Matheus | Giovane |
| 7 | N6+ | Matheus | Júlia |

Todo Onboarder do PODS entra na distribuição. A linha `fernanda.cardoso` (Closer, POD 1) é **de teste** e deve sair antes de produção.

### Aba `Cache`
Reconstruída todo dia pelo Sync e atualizada na hora pelo W2 (+1 na atribuição). Campos: `email, pod, level, time, carteira_atual, pre_onboarding_atual, sg{1,2,3}_carteira, sg{1,2,3}_pre_onboarding, ultima_atribuicao_em, atualizado_em`.

### Aba `Hist`
Uma linha por atribuição (e por alerta de capacidade), com o racional em texto.

### Como a carteira é contada (Sync Diário)
Consulta ao objeto **`Onboarding__c`**:
- `Onboarding_Status__c` **fora** de `Accomplished` e `Unaccomplished`;
- `RecordTypeId` em `012Ia0000004HwoIAE` ou `012SG00000KIyKHYA1`;
- Onboarder = `Owner.Email`; valor = `Amount_12_Months__c` (para o subgrupo); pré-onboarding = status `Pre Onboarding`.

**Donos excluídos da contagem** (lista `EXCLUIR_DONOS` no topo do código do Sync e do workflow 10): `bkf-team`, `pamella.martins`, `guilherme.baeta`, `fernanda.cardoso`.
Onboardings de donos que não estão no PODS ficam fora do cálculo de atribuição.
Atribuídas que ainda não chegaram ao Closed Won **não** entram na contagem (o cache do W2 cobre o dia).

**Travas do Sync:** se o Salesforce não devolver nenhum onboarding, ou o PODS vier vazio, o Sync **para antes de limpar o cache**.

### Upstash (prefixos `carteira:`)
`session:<token>` (8h), `rl:<email>` (limite de tentativas, 15 min), `login_log` (últimos 1.000 logins).

### Variáveis do n8n
`UPSTASH_ONB_URL`, `UPSTASH_ONB_TOKEN`, `CARTEIRA_SENHA_HASH` (formato `salt:hash scrypt`). Nunca coloque esses valores em arquivo do projeto.

### Credenciais do n8n
`Salesforce - Fernanda`, `Sheets Fer`, `GCal - sales.bizops`, `Fer email` (Gmail do alerta ao coordenador).

---

## 4. Regras de atribuição (W2)

**Subgrupo do cliente** (pelo nível do Closer e pelo `Amount_12_months__c` da Opportunity; limite inferior inclusivo, superior exclusivo):

| Nível | sg1 | sg2 | sg3 |
|---|---|---|---|
| N2-N3 | < R$ 300 mil | R$ 300 mil a < 600 mil | ≥ R$ 600 mil |
| N4-N5 | < R$ 2 mi | R$ 2 mi a < 3 mi | ≥ R$ 3 mi |
| N6+ | < R$ 10 mi | R$ 10 mi a < 15 mi | ≥ R$ 15 mi |

**Cascata** (nunca cruza nível):
1. Onboarders do **mesmo POD** do Closer com carteira total abaixo do limite (**N2-N3: 75, N4-N5: 50, N6+: 25**).
2. **Guardião** (só N2-N3 e N4-N5; limite 150; vai para quem tem a menor carteira total).
3. **Outro POD do mesmo nível.**
4. Ninguém cabe: e-mail ao coordenador do nível e nenhuma atribuição. Coordenadores: N2-N3 `raissa.salles`, N4-N5 `giovane.jansen`, N6+ `julia.brito`.

**Escolha dentro do pool:** menor carteira **no subgrupo** (quem já tem 10 ou mais sai da disputa) → menor pré-onboarding no subgrupo → quem recebeu há mais tempo (nunca recebeu = primeiro). Se todos têm 10 ou mais no subgrupo, vale a menor carteira total, com os mesmos desempates.

---

## 5. Login e segurança

- Login por e-mail `@hotmart.com` na aba `Acessos` (colunas `email | nome | papel | ativo`) e **senha padrão compartilhada** (guardada só como hash). Papéis: `admin` e `usuario`.
- Token aleatório de 256 bits, validade de **8h**, guardado no Upstash e conferido pelo n8n em **toda** chamada.
- Máximo de **5 tentativas por e-mail a cada 15 minutos**; a mensagem de erro é sempre a mesma ("E-mail ou senha inválidos").
- A senha é compartilhada: quem sair do time deve ser desativado (`ativo = FALSE`) na aba `Acessos`. Trocar a senha vale para todos (novo `CARTEIRA_SENHA_HASH`).
- O front-end é público, mas **não contém dado nem segredo**: só as URLs dos webhooks, que exigem sessão.
- Nome de pessoa e e-mail vindos do Salesforce são exibidos como **texto**, nunca como HTML.

---

## 6. Agenda (modelo da conta única)

Uma credencial OAuth2 (`GCal - sales.bizops`) autorizada por `sales.bizops@hotmart.com`. Cada Onboarder **compartilha a agenda** com essa conta, com a permissão **"Fazer alterações nos eventos"** ("Ver detalhes" não basta: dá 403 ao criar). O n8n usa o **e-mail do Onboarder como `calendarId`**, sem ramo por pessoa.

- Horários: 1h cada, 9h às 18h (Brasília), dias úteis, próximos 30 dias; ignora eventos "transparentes" (Casa/Escritório) e convites recusados.
- Se a agenda não puder ser lida, **não oferece horário nenhum** e a tela mostra o aviso. Se o evento não puder ser criado, a tela mostra a causa.
- Quem entra no PODS precisa compartilhar a agenda antes. O workflow **11** confere a leitura (não a escrita) de todos.

---

## 7. Testes feitos (09 a 11/10/2026)

- Login, 401 sem token ou com token falso, CORS restrito: ok.
- Ponta a ponta com a Opportunity de teste: atribuição (cache +1, Hist, `Onboarding_Owner__c`), formulário de campos, **reunião criada na agenda real do Onboarder** (Closer e cliente convidados) e **Closed Won**; o Salesforce criou o `Onboarding__c` sozinho.
- Contagem do Sync conferida contra o Salesforce: 19 Onboarders, soma 703 na época; cache e painel batendo.
- Testes automatizados dos nós de código e da tela, rodados antes de cada entrega. A tela é testada em **Chrome real** (Kanban, painel lateral, teclado, temas, contraste AA e larguras) com dados de exemplo, sem chamar o n8n.

---

## 8. Pendências

1. **Revogar a API key do n8n** que foi exposta numa conversa e gerar outra, de preferência com validade curta.
2. **Importar no n8n** as versões novas dos workflows **2 e 4** (checagem de dono e Amount obrigatório) e do **4** (erro do Salesforce na tela), se ainda não estiverem publicadas.
3. **Checagem de dono no workflow 3** (confirmar reunião): hoje um usuário logado poderia pedir um evento em qualquer agenda com qualquer convidado.
4. Remover do PODS a linha de **teste da fernanda.cardoso** (Closer) antes de produção.
5. Apagar os `Onboarding__c` e eventos de teste no Salesforce e nas agendas.
6. **Redesenho do front-end:** login (split screen, tema claro/escuro) e atribuição em Kanban feitos; falta o **painel de acompanhamento** por dentro. **Para o card mostrar a conta, reimportar o workflow 6** (a consulta passou a trazer `Account.Name`); sem isso o card mostra "Conta não informada".
7. O workflow **11** só prova **leitura** das agendas; a escrita só aparece ao criar o evento. Avaliar uma checagem de escrita.
8. Limpar a resposta de confirmação da reunião (hoje devolve só status, ID e link do evento).
9. O texto do Hist para o sg1 ainda diz "120k–300k" (o sg1 não tem piso).
10. A gravação do cache no Sync usa "limpar e inserir": há alguns segundos com o cache vazio. Trocar por atualização por linha se virar problema.
11. Se o volume de atribuições simultâneas crescer, mover o cache para o Upstash (atualização atômica); hoje fica no Sheets por decisão.

---

## 9. Gotchas do n8n

- **Reimportar um workflow não troca o que está publicado:** salve e **publique**, senão o webhook continua com a versão antiga.
- Credenciais podem sumir no reimport: reselecione nos nós afetados (`Sheets Fer`, `Salesforce - Fernanda`, `GCal - sales.bizops`).
- Reimporte **dentro do workflow existente** para manter o ID e a referência do W2 para o W3. Importar como novo duplica o caminho do webhook.
- O nó `Execute Workflow` do W2 aponta para o W3 por ID: confira depois de reimportar.
- Resposta 200 vazia = nó quebrado: abra **Executions** e procure o nó vermelho ou cinza.
- O Salesforce devolve o nome do campo com a **caixa original** (`Amount_12_Months__c` no `Onboarding__c`, `Amount_12_months__c` na Opportunity). Leia sem diferenciar maiúsculas.
- `Merge` com 3 entradas precisa estar em modo *append*.
- Um nó com **saída de erro** (`continueErrorOutput`) é como o erro do Salesforce chega à tela, em vez de um 500 genérico.

---

## 10. Arquivos do projeto

| Arquivo | O que é |
|---|---|
| `0.` a `11. Carteira - *.json` | Os workflows do n8n |
| `index-fer.html` | Onboarding Assignment (vai para o GitHub Pages) |
| `painel.html` | Painel de Acompanhamento (vai para o GitHub Pages) |
| `PODs_Q4_2026.xlsx` | Roster do Q4 (nomes, POD, função, e-mail) |
| `PODS Q4 - novo roster.xlsx` | Roster no formato da aba PODS, com a lista de mudanças |
| `Onboardings fora da contagem.xlsx` | IDs dos onboardings dos donos excluídos, para conferência |
| `acessos-modelo.csv` | Modelo da aba `Acessos` |
| `BACKUP - *` e `PODS - backup *.csv` | Cópias de segurança antes das trocas |
| `TEMP - Atualizar PODS Q4 (...).json` | Workflow temporário, **não usado**; pode apagar |
| `Carteira Onboarding .xlsx` | Relatório antigo da carteira (histórico; o Sync não usa mais) |
| `README.md` | Este arquivo |

---

## 11. Histórico de bugs relevantes

1. Tiebreaker desatualizado (welcomes e ordem do roster): trocado por pré-onboarding e `ultima_atribuicao_em`.
2. `ultima_atribuicao_em` nunca era gravada.
3. Campo de receita errado (`Amount` padrão, sempre vazio): passou a usar `Amount_12_months__c`.
4. Eventos "transparentes" bloqueavam a agenda inteira.
5. Closed Won falhava em silêncio por validation rule: checagem prévia dos 9 campos.
6. **Closed Won com `opportunityId` indefinido:** o nó usava `$json.body.opp_id` depois de a checagem de campos ter sido inserida no meio do fluxo. Passou a usar o `Id` da Opportunity buscada.
7. **Carteira inflada pela planilha antiga:** a aba `Carteira Onboarding` estava defasada (mostrava Onboarders N6+ acima do limite que na verdade estavam abaixo). O Sync agora lê o Salesforce.
8. **Valor do `Onboarding__c` lido na caixa errada** (`Amount_12_months__c` x `Amount_12_Months__c`): tudo caía em sg1.


### Campos do passo 3 (12/10/2026)
- Obrigatórios sempre: Price Policy, Negotiated Fee (lista dependente da Price Policy), Amount 12 months, Onboarding Segmentation, WhatsApp Support Needed, Closed Reason Detail, Number of products to be migrated, Sales Start Date, **Proven revenue from the last year** (texto rico: aceita texto e até 3 prints colados, embutidos em base64 e comprimidos para caber no limite do campo; se já houver conteúdo no Salesforce, ele não é sobrescrito).
- Condicional: **Other Platform** é exigido só quando `Current_Platform__c = Other`.
- Os requisitos por etapa (SQL, Offer Presented, Contract Sent) não bloqueiam o fluxo, porque o app vai direto a Closed Won. O Closed Won exige o **Onboarding Owner** preenchido (feito pela atribuição); sem ele o gatilho do Salesforce falha ao criar o Onboarding.


### Modo demonstração (`index-fer.html?demo=1`)
Mostra o funil com 7 oportunidades fictícias (5 elegíveis, 1 já atribuída, 1 fechada), sem nenhuma chamada ao n8n ou ao Salesforce, com faixa amarela de aviso. Serve para validar o visual do card "Disponível para atribuir" quando o funil real não tem nenhum elegível. Atribuir é recusado nesse modo. O login continua sendo exigido.
