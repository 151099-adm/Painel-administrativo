# Painel Administrativo de Obras — SEEL

Dashboard de gestão de tarefas administrativas por obra, com priorização (matriz Eisenhower), SLA, rotina semanal, diretrizes da empresa e autenticação real via Firebase.

## Stack

- **Frontend**: HTML/CSS/JS puro, arquivo único (`dashboard_tarefas.html`), sem build/bundler.
- **Backend**: Firebase (projeto `painel-obras-seel`, plano Spark/gratuito)
  - **Authentication**: e-mail/senha, restrito a domínio `@seel.com.br`, com **verificação de e-mail obrigatória** (`email_verified`) antes de liberar acesso ao board
  - **Firestore**: banco de dados NoSQL, usado como armazenamento principal
- **Hospedagem atual**: Netlify Drop (`delicate-malabi-e89ee8.netlify.app`) — deploy manual por arrastar-e-soltar. Não há CI/CD configurado ainda.

## Estrutura de dados (Firestore)

Quatro coleções:

- **`boards/shared`** — documento único com o quadro **compartilhado** entre todos os usuários, e liberado para qualquer usuário autenticado ler/escrever. Campo principal:
  - `tarefas_matriz_compartilhado` (ou `tarefas_matriz_privado` no modo privado): JSON com `{tasks, categoryColors, rotinaHistorico}`
    - `rotinaHistorico`: mapa `"NomeDaObra::idDoItem" -> [chaves de semana marcadas]` — **isso é o único dado de rotina que qualquer usuário pode gravar**; fica separado de propósito dos dados administrativos da obra (ver abaixo)
  - `perfis`: JSON com `{nomeDaPessoa: {setor, funcao, email, telefone}}` — gravado por qualquer usuário durante o próprio cadastro, ou editado por admin

- **`boards/admin_settings`** — documento único, **leitura liberada pra todos, escrita travada por regra do Firestore a quem está na coleção `admins`**. Contém:
  - `slaConfig`: `{q1, q2, q3, q4}` — dias-alvo de SLA por quadrante
  - `obraInfo`: mapa `nome_da_obra -> {administrativo, lider, status, un, engenheiro, coordenador, visibilidade, rotina: [{id, text}]}` — dados da obra em si (não confundir com `rotinaHistorico`, que é separado)
  - `obrasSeeded` / `rotinaSeeded`: flags de inicialização (29 obras pré-carregadas da planilha real da empresa)
  - `diretrizes[]`: `{id, titulo, link, descricao, anexoId, anexoNome}`
  - `customSections[]`: `{id, titulo, items: [{id, texto, link, anexoId, anexoNome}]}`
  - `pessoasExtra[]`: nomes cadastrados manualmente pelo admin (sem conta própria ainda)

- **`users/{uid}`** — um documento por usuário autenticado, para o quadro **privado** dele (mesma estrutura do board compartilhado) e a preferência de tema (`tarefas_tema`).

- **`attachments/{id}`** — um documento por arquivo anexado (Diretrizes/Seções), até ~700KB em base64. Leitura liberada a todos; escrita travada a admin pela regra do Firestore.

- **`admins/{uid}`** — a existência de um documento com o UID da pessoa é o que define se ela é administradora. **Não existe mais senha de promoção a admin** — a promoção é feita manualmente pelo Firebase Console (Firestore → coleção `admins` → criar documento com o UID da pessoa, pego em Authentication → Users).

Toda leitura/escrita passa pelas funções `storageGet(key, isShared)` / `storageSet(key, value, isShared)`, mais funções dedicadas para os dados administrativos (`loadAdminSettings()` / `saveAdminSettings()`), perfis (`loadPerfis()` / `savePerfis()`) e anexos (`saveAttachment()` / `loadAttachment()`).

## Regras de segurança do Firestore (versão atual, publicada)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    function isSeelUser() {
      return request.auth != null &&
             request.auth.token.email.matches('.*@seel[.]com[.]br$') &&
             request.auth.token.email_verified == true;
    }
    function isAdmin() {
      return isSeelUser() &&
             exists(/databases/$(database)/documents/admins/$(request.auth.uid));
    }
    match /boards/shared {
      allow read, write: if isSeelUser();
    }
    match /boards/admin_settings {
      allow read: if isSeelUser();
      allow write: if isAdmin();
    }
    match /users/{userId} {
      allow read, write: if isSeelUser() && request.auth.uid == userId;
    }
    match /attachments/{attachmentId} {
      allow read: if isSeelUser();
      allow write: if isAdmin();
    }
    match /admins/{uid} {
      allow read: if isSeelUser() && request.auth.uid == uid;
      allow write: if false;
    }
  }
}
```

**Importante**: esse controle de admin é reforçado no **banco de dados**, não só na tela — mesmo alguém tentando escrever direto via console do navegador não consegue burlar (exceto nos campos de `boards/shared`, que são intencionalmente abertos a todos — ver "Limitações" abaixo).

## Modelo de dados principal

- `tasks[]`: `{id, text, q (q1-q4, quadrante de prioridade), done, cat (obra/categoria), due (prazo), responsavel, createdAt, completedAt}`
- `categoryColors{}`: mapa `nome_da_obra -> {bg, text, mid}` (cores da paleta atribuídas por ordem de criação)

## Funcionalidades implementadas

- Matriz de priorização (4 quadrantes, drag-and-drop, com "Mover para ▾" como alternativa em touch/mobile)
- 31 obras pré-carregadas a partir de planilha real da empresa (`OBRAS_SEED`, hardcoded no JS)
- SLA configurável por quadrante, com alerta visual de atraso (admin-only)
- Rotina **semanal** por obra (reseta toda segunda-feira), com marcação liberada a todos e gestão de itens (admin-only)
- Relatório: tarefas concluídas por pessoa/obra (30 dias), cumprimento de rotina (últimas 4 semanas) — admin-only
- "Meu Time": lista curada de pessoas lideradas diretamente pelo admin logado, com resumo de pendências — dado pessoal, não visível a outros admins
- Diretrizes da empresa e seções personalizadas (links/anexos até ~700KB, guardados em documentos próprios no Firestore)
- Autenticação e-mail/senha restrita a `@seel.com.br`, com **verificação de e-mail obrigatória** e cadastro self-service (nome, setor, função, telefone)
- Recuperação de senha via fluxo oficial do Firebase (e-mail de redefinição real)
- Promoção a administrador **apenas via Firebase Console** (coleção `admins`), sem senha compartilhada
- Backup manual: botão "Exportar backup" (admin) baixa um `.json` com todo o board compartilhado
- Tema claro/escuro, cartão de perfil (iniciais + nome) no topo, ícones no menu
- Obras "encerradas" somem da lista pra usuários comuns; obras podem ser marcadas como visíveis só a admin

## Limitações conhecidas / dívidas técnicas

- **Arquivo único monolítico** (~2900 linhas HTML+CSS+JS inline). Recomendado dividir em módulos antes de crescer muito mais.
- **Sem testes automatizados.**
- **Sem CI/CD**: deploy é manual via Netlify Drop.
- **`tasks` e `categoryColors` continuam abertos a qualquer usuário autenticado** (intencional — todo mundo precisa criar/mover tarefas). Não há distinção de permissão dentro desses campos.
- **Dono único da infraestrutura**: hoje só uma pessoa tem acesso ao Firebase, Netlify e GitHub — ponto único de falha para a operação.
- **Sem trilha de auditoria**: não há registro de quem alterou o quê.
- **Sem política formal de LGPD**: o sistema guarda nome, telefone e e-mail de pessoas reais sem termo de consentimento ou política de retenção definida.
- Anexos em base64 dentro do próprio documento Firestore. Para uso mais pesado, migrar para o Firebase Storage.

## Firebase Console

Projeto: `painel-obras-seel` — acesso via [console.firebase.google.com](https://console.firebase.google.com)
