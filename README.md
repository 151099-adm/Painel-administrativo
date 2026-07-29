# Painel Administrativo de Obras — SEEL

Dashboard de gestão de tarefas administrativas por obra, com priorização (matriz Eisenhower), SLA, rotina semanal, diretrizes da empresa e autenticação real via Firebase.

## Stack

- **Frontend**: HTML/CSS/JS puro, arquivo único (`dashboard_tarefas.html`), sem build/bundler.
- **Backend**: Firebase (projeto `painel-obras-seel`)
  - **Authentication**: e-mail/senha, restrito a domínio `@seel.com.br` (validado no client E nas regras do Firestore)
  - **Firestore**: banco de dados NoSQL, usado como armazenamento principal
- **Hospedagem atual**: Netlify Drop (`delicate-malabi-e89ee8.netlify.app`) — deploy manual por arrastar-e-soltar. Não há CI/CD configurado ainda.

## Estrutura de dados (Firestore)

Duas coleções:

- `boards/shared` — um único documento com os dados do quadro **compartilhado** entre todos os usuários. Campos (cada um é uma string JSON, exceto onde indicado):
  - `tarefas_matriz_compartilhado`: JSON com `{tasks, categoryColors, slaConfig, obraInfo, obrasSeeded, rotinaSeeded, adminUser, adminPassword, securityQuestion, securityAnswer, diretrizes, customSections}`
- `users/{uid}` — um documento por usuário autenticado, para o quadro **privado** dele. Campos:
  - `tarefas_matriz_privado`: mesma estrutura JSON acima, mas isolada por usuário
  - `tarefas_tema`: `"dark"` ou `"light"` (preferência de tema, sempre pessoal)
  - `isAdminFlag`: `true`/`false` — se essa conta tem acesso ao "modo administrador" dentro do app

Toda leitura/escrita passa pelas funções `storageGet(key, isShared)` / `storageSet(key, value, isShared)` no JS — são o único ponto de acesso ao Firestore. `isShared=true` → `boards/shared`; `isShared=false` → `users/{uid}` do usuário logado.

## Regras de segurança do Firestore

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isSeelUser() {
      return request.auth != null &&
             request.auth.token.email.matches('.*@seel[.]com[.]br$');
    }
    match /boards/{boardId} {
      allow read, write: if isSeelUser();
    }
    match /users/{userId} {
      allow read, write: if isSeelUser() && request.auth.uid == userId;
    }
  }
}
```

## Modelo de dados principal (dentro do JSON de cada board)

- `tasks[]`: `{id, text, q (q1-q4, quadrante de prioridade), done, cat (obra/categoria), due (prazo), responsavel, createdAt, completedAt}`
- `categoryColors{}`: mapa `nome_da_obra -> {bg, text, mid}` (cores da paleta atribuídas por ordem de criação)
- `obraInfo{}`: mapa `nome_da_obra -> {administrativo, lider, status, un, engenheiro, coordenador, rotina[]}`
  - `rotina[]`: `{id, text, history[]}` — `history` é lista de chaves de semana (segunda-feira, formato `AAAA-MM-DD`) em que o item foi marcado como feito
- `slaConfig`: `{q1, q2, q3, q4}` — dias-alvo de SLA por quadrante
- `diretrizes[]`: `{id, titulo, link, descricao, anexo (base64), anexoNome, anexoTipo}`
- `customSections[]`: `{id, titulo, items: [{id, texto, link, anexo, anexoNome, anexoTipo}]}`

## Funcionalidades já implementadas

- Matriz de priorização (4 quadrantes, drag-and-drop) por obra
- 29 obras pré-carregadas a partir de uma planilha real da empresa (`OBRAS_SEED`, hardcoded no JS)
- SLA configurável por quadrante, com alerta visual de atraso
- Rotina semanal por obra (checklist que reseta toda segunda-feira)
- Relatório: tarefas concluídas por pessoa/obra (30 dias), cumprimento de rotina (últimas 4 semanas)
- Diretrizes da empresa e seções personalizadas (links/anexos até 1,5MB em base64)
- Autenticação e-mail/senha restrita a @seel.com.br, com cadastro self-service
- Modo administrador ligado à conta (persistido em `isAdminFlag`), não precisa repetir login toda sessão
- Tema claro/escuro (toggle no canto superior direito do board)

## Limitações conhecidas / dívidas técnicas

- **Arquivo único monolítico** (~2200 linhas HTML+CSS+JS inline). Recomendado dividir em módulos (`index.html`, `styles.css`, `app.js`, talvez separar por feature) antes de crescer mais.
- **Sem testes automatizados.**
- **Sem CI/CD**: deploy é manual via Netlify Drop. Recomendado conectar o repositório Git ao Netlify para deploy automático a cada push.
- **Senha de "promoção a admin" (`adminUser`/`adminPassword`)** é um segredo compartilhado gravado em texto simples no documento `boards/shared` do Firestore — funciona, mas não é uma prática de segurança robusta. Uma melhoria seria usar Firebase Custom Claims ou uma Cloud Function para gerenciar papéis de administrador.
- **Anexos em base64** dentro do próprio documento Firestore (limite de 1,5MB por arquivo, ~1MB por documento no total). Para uso mais pesado de anexos, migrar para o Firebase Storage.
- **Sem paginação/otimização de leitura**: cada carregamento do board lê o documento inteiro. Funciona bem até algumas centenas de tarefas; se crescer muito, considerar reestruturar em subcoleções.

## Firebase Console

Projeto: `painel-obras-seel` — acesso via [console.firebase.google.com](https://console.firebase.google.com)
