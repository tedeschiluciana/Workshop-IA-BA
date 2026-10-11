# Prompts Lovable – Jornada Perioperatória

Oct 11, 2026 · @Luciana

São 13 prompts para colar no Lovable em ordem, um por vez: o primeiro cria todo o banco e a segurança, os seguintes constroem um módulo cada, e o último só revisa a segurança.

## Como usar

1. Conecte o projeto ao seu Supabase (integração nativa do Lovable) antes do Prompt 1.
2. Cole o texto da seção "Knowledge do projeto" em Project Settings → Knowledge. Ele é lido em todo prompt, então as regras fixas não precisam ser repetidas e você economiza créditos.
3. Troque `[SEU_EMAIL]` no Prompt 1 pelo e-mail que nascerá como anestesista e `[TELEFONE]` / `[NOME_DR]` no Knowledge.
4. Envie um prompt por vez. Só avance quando o teste indicado ao final de cada prompt passar.
5. Se algo quebrar, corrija com um prompt curto que comece com "Sem alterar mais nada:" em vez de reenviar o prompt inteiro.
6. Use o modo Chat/Plan do Lovable para dúvidas; ele não altera código e custa menos.

Cada prompt traz três blocos fixos: **Vê / Faz / Clica** (o que o usuário enxerga e a resposta de cada botão), **Dados** (tabelas e funções com que a tela conversa) e **Não alterar** (o que fica intocado).

## Knowledge do projeto

Cole uma única vez em Project Settings → Knowledge.

```text
PRODUTO
SaaS de jornada perioperatória. Usuários: anestesista (acesso total), cirurgião e secretária (só agenda), paciente (só páginas públicas por link, sem login). Idioma: pt-BR. Banco: Supabase. Única integração externa: Google Calendar.

REGRAS FIXAS DE SEGURANÇA (valem em todo prompt; nunca remover nem afrouxar)
1. RLS ativa em todas as tabelas. Toda tabela nova nasce com RLS e sem acesso para anon.
2. Páginas públicas (/f/:token, /csat/:token, /agenda/:token) nunca leem tabelas: usam só funções RPC que recebem o token e devolvem apenas os campos necessários.
3. Papel e situação (ativo/inativo) ficam na tabela papeis_usuario, separada de perfis, e só mudam pela função admin_definir_perfil, restrita a anestesista ativo. Ninguém altera o próprio papel.
4. Só o e-mail semente nasce anestesista ativo. Demais cadastros nascem inativos. O anestesista pode fechar novos cadastros.
5. respostas_paciente é lida só por anestesista. Secretária e cirurgião geram e enviam o link, mas não veem respostas.
6. Anestesista só acessa dados clínicos com sessão em duas etapas (aal2).
7. registros_acesso é gravada só por funções do banco (login por registrar_login, sem gatilho em auth.sessions), nunca pelo navegador; sem UPDATE nem DELETE para ninguém.
8. Exames e PDFs em buckets privados, acessados só por link temporário (60 s) emitido pela função de servidor arquivo-link, que exige aal2; tipo e tamanho validados no servidor.
9. Nenhuma chave secreta no código do navegador. Dados de paciente nunca em console.log, URL, localStorage, sessionStorage ou mensagens de erro.

REGRAS DE TRABALHO
- Altere somente o que o prompt pede. Não renomeie tabelas, colunas, funções ou rotas existentes.
- Não crie tabelas fora do esquema do Prompt 1 sem que o prompt peça.
- Reutilize componentes já criados. Sem dados fictícios em produção.
- Ao terminar, responda em até 5 linhas: o que foi criado e o que testar.

DESIGN
Clean e minimalista. Agenda no estilo Google Calendar; prontuário e formulários no estilo Notion (blocos, muito espaço em branco, tipografia sóbria); dashboard no estilo Power BI (cartões de indicador e gráficos simples). Fundo claro, uma cor de destaque, cantos suaves, sem sombras pesadas nem gradientes. Responsivo; páginas públicas pensadas primeiro para celular.

CONTATO PADRÃO NAS ORIENTAÇÕES
Dúvidas: [TELEFONE], [NOME_DR].
```

## Fundação: banco, segurança e acesso

### Prompt 1 — Banco de dados e segurança

```text
Crie SOMENTE o banco no Supabase (migrações SQL), os buckets e uma função de servidor. Não crie telas, rotas nem componentes.
Todas as decisões de arquitetura abaixo já estão tomadas: implemente como descrito, sem pedir confirmação.

TABELAS (use exatamente estes nomes)
- perfis: id (= auth.users.id), nome, email, crm, telefone, criado_em. Não contém papel nem situação.
- papeis_usuario: usuario_id (único, = auth.users.id), papel (anestesista|cirurgiao|secretaria), ativo (padrão false), atualizado_por, atualizado_em. Tabela separada de perfis, única fonte de papel e situação.
- config_plataforma (linha única): cadastros_abertos, telefone_contato, nome_medico, modelo_orientacoes (jsonb)
- locais: id, nome, tipo (consultorio|hospital)
- agendamentos: id, local_id, tipo (consulta|procedimento), modalidade (presencial|telemedicina), inicio, fim, procedimento, anestesista_id, cirurgiao_id (opcional), paciente_nome, paciente_telefone, status (agendado|formulario_enviado|formulario_respondido|atendido|cancelado), google_event_id, criado_por, criado_em
- links_publicos: id, tipo (formulario|csat|agenda), token_hash (único), agendamento_id (nulo se agenda), cirurgiao_id (só agenda), criado_por, expira_em, usado_em, revogado
- respostas_paciente: id, agendamento_id (único), dados (jsonb), enviado_em
- pacientes: id, nome, email, telefone, nascimento, sexo, convenio, carteirinha, criado_em
- apas: id, agendamento_id (único), paciente_id, status (pre_consulta|em_atendimento|finalizada), dados_paciente (jsonb), dados_clinicos (jsonb), scores (jsonb), asa, liberado (bool), conclusao, tecnica_anestesica, planejamento, condutas (jsonb), orientacoes (jsonb), pdf_path, finalizada_em, atualizado_em
- exames: id, apa_id, categoria (hematologia|bioquimica|rx_torax|ecg|eco|outros), arquivo_path, texto_ocr, enviado_por, enviado_em
- csat_respostas: id, agendamento_id (único), notas (jsonb), comentario, respondido_em
- registros_acesso: id, usuario_id, acao (login|abrir_apa|abrir_exame|baixar_pdf|enviar_pdf|alterar_perfil), alvo_id, sessao_id, detalhe, criado_em. Índice único em sessao_id quando acao = login.
- google_tokens: usuario_id, refresh_token, calendario_id

FUNÇÕES AUXILIARES (security definer, search_path fixo; leem papeis_usuario sem recursão de RLS)
- sessao_registrada(): existe em registros_acesso um login com sessao_id = auth.jwt()->>'session_id'.
- eh_equipe(): papeis_usuario do usuário com ativo = true E sessao_registrada().
- eh_anestesista(): eh_equipe() E papel = anestesista E auth.jwt()->>'aal' = 'aal2'.

RLS (ativa em todas; anon sem acesso a nenhuma tabela)
- papeis_usuario: cada um lê a própria linha; eh_anestesista() lê todas. Nenhuma política de INSERT, UPDATE ou DELETE: só as funções abaixo escrevem.
- perfis: cada um lê o próprio; eh_anestesista() lê todos; o próprio usuário altera só nome, telefone e crm.
- config_plataforma, locais: eh_equipe() lê; eh_anestesista() altera.
- agendamentos: eh_equipe() lê, cria e altera.
- respostas_paciente, pacientes, apas, exames, csat_respostas: somente eh_anestesista().
- links_publicos, google_tokens: nenhuma política (só funções e service_role).
- registros_acesso: SELECT só eh_anestesista(); nenhuma política de INSERT, UPDATE ou DELETE; gatilho na própria tabela que bloqueia UPDATE e DELETE.

FUNÇÕES (security definer, search_path fixo)
- Gatilho em auth.users (INSERT): cria a linha em perfis e em papeis_usuario. E-mail [SEU_EMAIL] nasce anestesista ativo. Outros: se cadastros_abertos = false, recusa o cadastro; senão nasce secretaria inativo.
- admin_definir_perfil(usuario, papel, ativo): só eh_anestesista(); recusa se usuario = quem chama; atualiza papeis_usuario e grava alterar_perfil.
- admin_definir_cadastros(aberto): só eh_anestesista().
- gerar_link(tipo, agendamento_id, cirurgiao_id): só eh_equipe(); gera token aleatório de 32 bytes, guarda só o hash SHA-256, devolve o token uma única vez. Validade: formulario 14 dias, csat 30, agenda 90. Revoga link anterior do mesmo tipo e agendamento.
- revogar_link(id): só eh_equipe().
- publico_abrir_formulario(token): devolve só primeiro nome, procedimento e data. publico_enviar_formulario(token, dados): valida, grava em respostas_paciente, marca usado_em, muda status para formulario_respondido. Uso único.
- publico_abrir_csat(token) e publico_enviar_csat(token, notas, comentario): mesma lógica, uso único.
- publico_agenda(token, de, ate): devolve só inicio, fim, local, procedimento, nome do anestesista e iniciais do paciente.
- abrir_apa(apa_id): só eh_anestesista(); grava abrir_apa e devolve a APA.
- registrar_acesso(usuario, acao, alvo_id, detalhe): executável só por service_role.
Funções publico_* são as únicas executáveis por anon; token inválido, vencido, usado ou revogado devolve sempre o mesmo erro genérico.

AUDITORIA DE LOGIN (decisão final: NÃO usar gatilho em auth.sessions nem em nenhuma outra tabela dos esquemas auth ou storage, exceto o gatilho de cadastro em auth.users)
- registrar_login(): sem parâmetros, executável por usuário autenticado. Lê auth.uid() e auth.jwt()->>'session_id' dentro do banco e insere um registro login; repetição na mesma sessão é ignorada pelo índice único. O navegador não envia nenhum dado, só dispara a função.
- Como eh_equipe() exige sessao_registrada(), uma sessão sem login registrado não lê nem grava nada. O registro não pode ser pulado.

ARMAZENAMENTO
- Bucket privado exames: PDF, JPEG, PNG; máx. 10 MB. Bucket privado apas: PDF; máx. 5 MB. Limites de tipo e tamanho configurados no próprio bucket.
- Upload só por eh_anestesista(). Sem política de leitura para o navegador.

FUNÇÃO DE SERVIDOR arquivo-link(tipo, id, acao) (decisão final: função segura do aplicativo, executada só no servidor; use o mecanismo de função de servidor do projeto)
1. Exige usuário autenticado e valida o JWT no servidor.
2. Exige aal = aal2 no JWT e, consultando o banco, papel anestesista com ativo = true e sessão registrada. Qualquer falha devolve 403 sem detalhe.
3. Confere que id existe em exames ou apas e obtém o caminho do arquivo no banco; nunca aceita caminho vindo do navegador.
4. Chama registrar_acesso com acao abrir_exame, baixar_pdf ou enviar_pdf.
5. Devolve URL assinada com validade de 60 segundos.
A chave service_role fica só no ambiente do servidor.

TESTE AO FINAL: liste tabelas com RLS ativa; confirme que anon não lê nenhuma e que um usuário autenticado não consegue alterar papeis_usuario.
```

### Prompt 2 — Login, duas etapas e estrutura

```text
Crie o acesso à plataforma e a estrutura de navegação.

VÊ / FAZ / CLICA
/login mostra e-mail, senha e os botões:
- Entrar → autentica no Supabase. Erro: "E-mail ou senha inválidos" (mensagem única). Sucesso: chama registrar_login(), lê a própria linha em papeis_usuario e segue a regra de destino abaixo.
- Entrar com Google / Entrar com Apple → OAuth do Supabase; ao voltar, mesma regra de destino.
- Criar conta → pede nome, e-mail e senha (mín. 10 caracteres). Resposta: "Cadastro recebido, aguarde liberação" ou, se o banco recusar, "Cadastros fechados".
- Esqueci a senha → envia e-mail de redefinição; resposta sempre igual, exista ou não a conta.

Regra de destino:
- ativo = false → tela "Aguardando liberação" com botão Sair.
- anestesista sem fator cadastrado → /2fa/configurar: mostra QR code TOTP e campo de código; Confirmar → cadastra e verifica o fator → /dashboard.
- anestesista com fator → /2fa: campo de 6 dígitos; Verificar → eleva a sessão para aal2 → /dashboard. Sem aal2 não entra em nenhuma outra rota.
- cirurgiao ou secretaria → /agenda.

Estrutura após login: menu lateral com Dashboard, Agenda, Prontuário, Configurações e Perfis de acesso para anestesista; só Agenda e Configurações para cirurgiao e secretaria. No rodapé do menu: nome, papel e Sair (encerra a sessão → /login). Cada item abre uma página vazia com título; o conteúdo vem nos próximos prompts. Rota não permitida ao papel redireciona para /agenda.

DADOS: papeis_usuario e perfis (leitura do próprio registro), função registrar_login, Supabase Auth e MFA.

NÃO ALTERAR: nenhuma tabela, política, função ou bucket do Prompt 1. Nenhuma migração nova.

TESTE: entrar com [SEU_EMAIL] exige 2FA; uma segunda conta cai em "Aguardando liberação".
```

## Agenda e formulário do paciente

### Prompt 3 — Agenda e novo agendamento

```text
Preencha a página Agenda.

VÊ
Calendário com alternância Dia / Semana / Mês, setas, botão Hoje, filtros por anestesista e local. Eventos coloridos por tipo (consulta, procedimento); cancelados riscados. Botão "Novo agendamento".

FAZ / CLICA
- Novo agendamento → painel lateral em 3 passos: (1) local e tipo; se consulta, modalidade presencial ou telemedicina; (2) data, hora de início e duração; (3) procedimento, anestesista, cirurgião (opcional), nome e telefone do paciente. Salvar → insere em agendamentos com status agendado → fecha o painel, o evento aparece e seu cartão abre.
- Clicar em um evento → cartão com dados e status e os botões:
  - Gerar link do formulário → chama gerar_link('formulario') → mostra o endereço /f/<token> uma única vez com Copiar e Abrir no WhatsApp (mensagem padrão contendo só o link) → status vira formulario_enviado.
  - Editar → reabre o painel preenchido → Salvar atualiza.
  - Cancelar → pede confirmação → status cancelado.
  - Abrir pré-consulta → visível só para anestesista quando status = formulario_respondido; por ora desabilitado.
- Arrastar um evento → atualiza inicio e fim.
- Horário sobreposto para o mesmo anestesista → aviso amarelo, sem bloquear.
- "+ Novo local" na lista de locais aparece só para anestesista → insere em locais.

DADOS: agendamentos, locais, função gerar_link.
ÚNICA MIGRAÇÃO PERMITIDA: função listar_equipe() para eh_equipe(), devolvendo só id, nome e papel dos perfis ativos (para as listas de anestesista e cirurgião).

NÃO ALTERAR: login, 2FA, menu, demais tabelas, políticas e funções.

TESTE: secretária cria agendamento e gera link; o token não aparece de novo ao reabrir o cartão.
```

### Prompt 4 — Google Agenda e link do cirurgião

```text
Adicione a sincronização com o Google Agenda e a agenda compartilhada por link.

VÊ / FAZ / CLICA
- Configurações ganha o bloco "Google Agenda" (só anestesista): botão Conectar → consentimento Google com escopo calendar.events → a função de servidor google-oauth troca o código e grava o refresh token em google_tokens → o bloco mostra "Conectado" e o botão Desconectar (apaga o token).
- Criar, editar ou cancelar um agendamento → a função de servidor google-sync cria, atualiza ou remove o evento no calendário do anestesista escolhido e grava google_event_id. Título do evento: procedimento + local + iniciais do paciente. Nunca enviar nome completo, telefone ou dado clínico ao Google. Falha de sincronização → ícone de alerta no evento com botão Tentar de novo; o agendamento não é perdido.
- Sincronização em mão única: plataforma → Google.
- Agenda ganha o botão "Compartilhar com cirurgião" → escolhe o cirurgião (ou todos) → gerar_link('agenda') → mostra /agenda/<token> uma vez com Copiar. Abaixo, lista dos links ativos com validade e botão Revogar → revogar_link.
- /agenda/<token> (pública, sem login): calendário somente leitura Dia / Semana / Mês alimentado apenas por publico_agenda. Token inválido → "Link inválido ou expirado".

DADOS: google_tokens (só nas funções de servidor), agendamentos.google_event_id, links_publicos via funções.
SEGREDOS: client id e client secret do Google apenas nos secrets do servidor. Use funções de servidor do aplicativo, como arquivo-link; não pergunte qual mecanismo usar.

NÃO ALTERAR: fluxo de novo agendamento, cartão do evento, login, banco (nenhuma migração).

TESTE: criar agendamento gera evento no Google; link revogado deixa de abrir.
```

### Prompt 5 — Formulário público do paciente

```text
Crie a página pública /f/<token>, sem login, pensada para celular.

VÊ / FAZ / CLICA
- Ao abrir → publico_abrir_formulario. Inválido, vencido ou já respondido → mensagem única "Link inválido ou já utilizado". Válido → saudação com primeiro nome, procedimento e data, e caixa obrigatória de consentimento LGPD.
- Formulário em passos com barra de progresso e botões Voltar / Continuar (Continuar valida o passo):
  1. Identificação: nome, e-mail, telefone, data de nascimento (idade calculada na tela), sexo, convênio, carteirinha.
  2. Cirurgia: diagnóstico ou motivo, peso (kg), altura (cm).
  3. Comorbidades: mapa corporal interativo. Silhueta com regiões clicáveis (cabeça/neurológico, coração, pulmões, fígado, rins, endócrino, sangue, ossos e articulações, outros). Clicar em uma região → abre lista de condições comuns com caixas de seleção e campo "outra". Região com item marcado fica destacada; abaixo, resumo do que foi marcado. Opção "Não tenho doenças".
  4. Medicamentos: linhas repetíveis com nome, dose e posologia; botão "+ Adicionar"; opção "Não uso".
  5. Alergias: medicamentosa (quais), látex, esparadrapo, contraste.
  6. Hábitos e histórico: fumante, uso de substâncias ilícitas, complicações anestésicas prévias, cirurgias prévias (lista), realizou exames pré-operatórios, resfriado ou covid nas últimas duas semanas.
  7. Revisão: resumo de tudo com Editar por bloco e botão Enviar → publico_enviar_formulario → tela "Recebido, obrigado".
- Respostas ficam só na memória da página até o envio.

DADOS: somente as funções publico_abrir_formulario e publico_enviar_formulario. O objeto dados usa chaves fixas: identificacao, diagnostico, peso_kg, altura_cm, comorbidades, medicamentos, alergias, tabagismo, substancias, complicacoes_anestesicas, cirurgias_previas, exames_realizados, infeccao_respiratoria_recente, consentimento_em.

NÃO ALTERAR: área logada, agenda, banco (nenhuma migração). Esta página não consulta nenhuma tabela diretamente.

TESTE: enviar muda o status do agendamento para formulario_respondido; reabrir o link mostra a mensagem única.
```

## Consulta e APA

### Prompt 6 — Prontuário e pré-consulta

```text
Preencha a página Prontuário (só anestesista) e crie a tela da pré-consulta.

VÊ / FAZ / CLICA
- Prontuário: duas abas, "Pré-consultas" (formulário respondido ou atendimento em andamento) e "APAs realizadas" (finalizadas). Tabela com paciente, procedimento, data e status; busca por nome; ordenação por data.
- Clicar em uma linha, ou em "Abrir pré-consulta" no cartão da agenda (habilite o botão) → chama iniciar_apa se ainda não existir APA, depois abrir_apa → abre /prontuario/<id da apa>.
- Tela da pré-consulta, em blocos estilo Notion:
  - Cabeçalho: paciente, idade, procedimento, data, seletor de modalidade (presencial | telemedicina), indicador "Salvo".
  - Respostas do paciente: tudo o que veio do formulário, com etiqueta "informado pelo paciente", sempre editável.
  - Diagnóstico: código CID-10 e descrição (descrição pré-preenchida com o diagnóstico do paciente).
  - Hospital: pré-preenchido com o local do agendamento.
  - Exame físico: lateralidade; via aérea (Mallampati, abertura bucal, distância tireomentoniana, mobilidade cervical, protrusão mandibular, dentição); cardiovascular (PA, FC, ausculta, observações); neuroeixo (inspeção da coluna, observações).
  - Capacidade funcional (METs) e classificação ASA (I a VI, com marcador E).
  - Conclusão para o procedimento, técnica anestésica proposta e planejamento.
  - Blocos vazios com título: Scores, Exames, Condutas e orientações.
- Qualquer edição salva sozinha após 1 s → atualiza apas (dados_paciente ou dados_clinicos) → indicador muda para "Salvo". A primeira edição muda o status para em_atendimento.

DADOS: apas, pacientes, agendamentos, função abrir_apa.
ÚNICA MIGRAÇÃO PERMITIDA: função iniciar_apa(agendamento_id), só eh_anestesista(): cria o registro em pacientes a partir de respostas_paciente, cria a apa com status pre_consulta copiando as respostas para dados_paciente e devolve o id.

NÃO ALTERAR: agenda (exceto habilitar o botão), formulário público, login, demais funções e políticas.

TESTE: abrir uma pré-consulta grava abrir_apa em registros_acesso; secretária não vê o menu Prontuário nem recebe dados pela API.
```

### Prompt 7 — Scores automáticos

```text
Preencha o bloco Scores da tela da pré-consulta.

VÊ / FAZ / CLICA
- Oito cartões: IMC, Caprini, STOP-Bang, DASI, Índice de Lee (RCRI), ARISCAT, Apfel, El-Ganzouri. Cada cartão mostra total e interpretação.
- Clicar em um cartão → expande a lista de critérios daquele score como caixas de seleção ou campos.
- Itens deriváveis de dados já existentes (idade, sexo, peso, altura, tabagismo, comorbidades, via aérea, tipo de cirurgia) vêm pré-marcados com a etiqueta "pré-preenchido" e continuam editáveis.
- Marcar ou desmarcar um item → total e interpretação recalculam na hora → salva em apas.scores no formato { nome_do_score: { itens, total, interpretacao } }.
- IMC é calculado de peso e altura, sem itens.
- Os scores apenas informam: não alteram ASA, conclusão nem condutas.

IMPLEMENTAÇÃO: todas as fórmulas em um único arquivo de funções puras, seguindo os critérios e pontos de corte das publicações originais de cada score, com a referência em comentário e um teste unitário por score.

DADOS: apas.scores (gravação), apas.dados_paciente e apas.dados_clinicos (leitura).

NÃO ALTERAR: demais blocos da pré-consulta, salvamento automático, banco (nenhuma migração).

TESTE: alterar peso recalcula IMC e os itens dependentes.
```

### Prompt 8 — Exames com foto e leitura de texto

```text
Preencha o bloco Exames da tela da pré-consulta.

VÊ / FAZ / CLICA
- Abas por categoria: hematologia, bioquímica, RX de tórax, ECG, ECO, outros. Cada aba lista os exames enviados com data.
- Enviar PDF → seletor de arquivo. Tirar foto → abre a câmera do dispositivo.
- Após escolher → confere tipo (PDF, JPEG, PNG) e tamanho (10 MB); fora disso mostra o motivo e não envia. Dentro → envia ao bucket exames em <apa_id>/<uuid> → insere em exames → o item aparece na lista.
- Em seguida a leitura de texto roda no próprio navegador (OCR em português para imagem; extração de texto para PDF) → grava em exames.texto_ocr → mostra ao lado do exame o campo editável "Texto extraído — confira com o original".
- Clicar em um exame → chama a função de servidor arquivo-link (acao abrir_exame) → abre o arquivo em nova aba pelo link de 60 s.
- Excluir → pede confirmação → remove arquivo e registro.
- Nenhum serviço externo de OCR ou IA.

DADOS: exames, bucket exames, função de servidor arquivo-link.

NÃO ALTERAR: demais blocos, limites do bucket, banco (nenhuma migração).

TESTE: arquivo de 15 MB é recusado também pelo servidor; abrir exame grava abrir_exame em registros_acesso.
```

### Prompt 9 — Condutas, orientações e APA em PDF

```text
Preencha o bloco Condutas e orientações e crie a finalização do atendimento.

VÊ / FAZ / CLICA
- Liberação: liberado para cirurgia (sim | não) e "necessita de recomendações especiais" (texto).
- Condutas (caixas com detalhe): novos exames (quais), reavaliação posterior (data), aguarda avaliação de especialista (qual), reagendar cirurgia.
- Medicações: uma linha por medicamento informado pelo paciente; o anestesista escolhe manter, suspender N dias antes, tomar na manhã da cirurgia ou ajustar (texto). A escolha desenha uma linha do tempo até a data da cirurgia. Nenhuma sugestão automática de conduta.
- Jejum: painel visual em linha do tempo, com horários calculados a partir do início do procedimento e das regras em config_plataforma.modelo_orientacoes (padrão: 8 h refeição completa, 6 h refeição leve ou leite não humano, 4 h leite materno, 2 h líquidos claros), cada faixa com exemplos de alimentos e o horário-limite. Editável por paciente.
- Orientações gerais: joias, piercings, maquiagem, cílios postiços, unhas, mega-hair, levar documentos e exames. Texto vindo do modelo, editável.
- Contato: "Dúvidas, entre em contato" com telefone_contato e nome_medico.
- Tudo salva em apas (liberado, condutas, orientacoes) pelo salvamento automático existente.
- Botão Finalizar atendimento → confere ASA e conclusão preenchidos; faltando, destaca o campo. Completo → abre a pré-visualização da APA: identificação, respostas do paciente, exame físico, scores, exames, ASA, conclusão, técnica e planejamento, liberação, condutas e orientações, em formato estruturado.
- Na pré-visualização, a pergunta "Há alguma alteração final a ser feita?" com dois botões: Voltar e editar → retorna à tela; Confirmar e gerar PDF → gera o PDF, envia ao bucket apas, grava pdf_path, status finalizada e finalizada_em, e muda o agendamento para atendido.
- Tela final: Baixar PDF → arquivo-link (baixar_pdf). Enviar ao paciente → arquivo-link (enviar_pdf) e abre o compartilhamento do dispositivo com o arquivo; em computador, baixa o arquivo e mostra o telefone e o e-mail do paciente para envio.
- APA finalizada fica somente leitura; botão Reabrir para correção → volta a em_atendimento; nova finalização substitui o PDF.
- O PDF tem duas partes: a APA completa (prontuário) e a folha de orientações ao paciente; só a folha de orientações é enviada ao paciente.

DADOS: apas, agendamentos.status, config_plataforma, bucket apas, função de servidor arquivo-link.

NÃO ALTERAR: scores, exames, formulário público, banco (nenhuma migração).

TESTE: baixar e enviar gravam baixar_pdf e enviar_pdf em registros_acesso; a APA aparece em "APAs realizadas".
```

## CSAT, dashboard e administração

### Prompt 10 — Avaliação CSAT

```text
Crie a avaliação de satisfação (CSAT) do paciente.

VÊ / FAZ / CLICA
- O cartão de um agendamento do tipo procedimento ganha o botão Gerar link CSAT → gerar_link('csat') → mostra /csat/<token> uma vez com Copiar e Abrir no WhatsApp.
- /csat/<token> (pública, sem login, celular primeiro): publico_abrir_csat → mostra primeiro nome e procedimento. Link inválido ou já respondido → mensagem única.
- Cinco perguntas com escala de 1 a 5: consulta pré-anestésica, clareza das orientações, acolhimento no dia da cirurgia, controle de dor e náusea, satisfação geral. Comentário opcional. Enviar → publico_enviar_csat → tela "Obrigado".
- Cálculo automático: CSAT = percentual de notas 4 ou 5 sobre o total de notas, geral e por pergunta.
- No cartão do agendamento, anestesista vê a nota média e o comentário; cirurgiao e secretaria veem só "respondido" ou "pendente".

DADOS: csat_respostas via funções publico_*, links_publicos via gerar_link.
ÚNICA MIGRAÇÃO PERMITIDA: função csat_resumo(de, ate), só eh_anestesista(), devolvendo CSAT geral, por pergunta e por mês; e função csat_situacao(agendamento_id) para eh_equipe(), devolvendo só respondido ou pendente.

NÃO ALTERAR: formulário do paciente, pré-consulta, APA, demais botões do cartão.

TESTE: três respostas com notas conhecidas produzem o percentual esperado.
```

### Prompt 11 — Dashboard

```text
Preencha a página Dashboard (só anestesista), no estilo Power BI.

VÊ / FAZ / CLICA
- Filtros no topo: período (padrão: mês atual) e local. Mudar um filtro → recalcula tudo.
- Cartões de indicador: agendamentos hoje, agendamentos na semana, formulários enviados sem resposta, pré-consultas aguardando atendimento, APAs finalizadas no período, percentual de liberados para cirurgia, CSAT do período.
- Clicar em um cartão → abre a Agenda ou o Prontuário já filtrado para aquele conjunto.
- Gráficos: atendimentos por semana (barras), distribuição por ASA (barras), consultas x procedimentos por local (barras agrupadas), taxa de resposta do formulário (linha), CSAT por mês (linha).
- Lista "Próximos atendimentos" com os 5 seguintes; clicar em um → abre o cartão na Agenda.
- Sem dados no período → estado vazio com texto, nunca números fictícios.

DADOS: agendamentos, apas (somente contagens e asa/liberado/status), csat_resumo. Somente leitura.

NÃO ALTERAR: todas as demais páginas, banco (nenhuma migração).

TESTE: os números dos cartões batem com as listas abertas ao clicar.
```

### Prompt 12 — Configurações, perfis e registros de acesso

```text
Preencha as páginas Configurações e Perfis de acesso.

CONFIGURAÇÕES (todos os papéis)
- Meus dados: nome, telefone, CRM → Salvar → atualiza o próprio registro em perfis → aviso "Salvo".
- Trocar e-mail → pede a senha atual → envia confirmação ao novo e-mail → só troca após confirmar.
- Trocar senha → senha atual, nova (mín. 10 caracteres) e repetição → Salvar → encerra as outras sessões.
- Verificação em duas etapas: Ativar → QR code e código; Trocar dispositivo → novo QR code após código atual. Desativar existe só para cirurgiao e secretaria; anestesista não pode desativar.
- Sair de todos os dispositivos → encerra todas as sessões.
Somente anestesista vê também: Contato das orientações (telefone_contato, nome_medico), Modelos de orientações (regras de jejum e texto das orientações gerais) e Locais (adicionar, renomear) → Salvar grava em config_plataforma ou locais. O bloco Google Agenda continua como está.

PERFIS DE ACESSO (só anestesista), duas abas
- Usuários: tabela com nome, e-mail, papel, situação e data de cadastro. Seletor de papel e chave ativo/inativo em cada linha → pede confirmação → admin_definir_perfil → a linha atualiza. A linha do próprio usuário fica travada. Acima da tabela, chave "Aceitar novos cadastros" → admin_definir_cadastros.
- Registros de acesso: tabela somente leitura com data e hora, usuário, ação e alvo; filtros por usuário, ação e período; paginação. Sem botões de editar ou excluir.

DADOS: perfis, papeis_usuario (leitura), config_plataforma, locais, registros_acesso (leitura), funções admin_definir_perfil e admin_definir_cadastros, Supabase Auth e MFA.

NÃO ALTERAR: fluxo de login e 2FA, bloco Google Agenda, banco (nenhuma migração). O navegador nunca grava em registros_acesso.

TESTE: liberar uma secretária gera um registro alterar_perfil; ela entra e vê só Agenda e Configurações.
```

## Revisão de segurança

### Prompt 13 — Revisão final, sem funcionalidades novas

```text
Faça uma revisão de segurança. NÃO adicione funcionalidades, telas, campos nem mude o visual. Corrija apenas o que falhar e só no ponto da falha.

Confira item por item e responda em tabela (item | OK ou FALHA | evidência | correção aplicada):
1. Todas as tabelas com RLS ativa; anon sem SELECT, INSERT, UPDATE ou DELETE em qualquer tabela.
2. /f, /csat e /agenda públicas usam somente funções publico_*; nenhuma consulta direta a tabela nessas páginas. Cada função devolve só os campos previstos e erro genérico único.
3. Tokens: 32 bytes aleatórios, só o hash no banco, com validade, uso único (formulário e CSAT) e revogação.
4. papel e ativo existem só em papeis_usuario (não em perfis), sem INSERT, UPDATE ou DELETE direto para nenhum usuário; admin_definir_perfil recusa quem não é anestesista e recusa alterar a si mesmo.
5. Só o e-mail semente nasce anestesista ativo; os demais nascem inativos; cadastros fechados recusam novo cadastro.
6. respostas_paciente, pacientes, apas, exames e csat_respostas inacessíveis a cirurgiao e secretaria, também via API direta e via funções.
7. Anestesista sem aal2 não lê dado clínico (conferir eh_anestesista() em todas as políticas e funções); anestesista não consegue desativar o 2FA.
8. registros_acesso: nenhum INSERT direto do navegador; sessão sem login registrado por registrar_login não acessa dados; nenhum gatilho em auth.sessions; UPDATE e DELETE bloqueados para todos; existem registros de login, abrir_apa, abrir_exame, baixar_pdf, enviar_pdf e alterar_perfil.
9. Buckets exames e apas privados, com limite de tipo e tamanho no servidor; nenhum link público ou permanente; links assinados de 60 s emitidos só pela função de servidor arquivo-link, que exige aal2 e papel anestesista ativo.
10. Nenhuma chave service_role, client secret do Google ou outro segredo no código do navegador ou no repositório.
11. Nenhum dado de paciente em console.log, parâmetros de URL, localStorage, sessionStorage, títulos de aba ou mensagens de erro. Remova todo console.log restante.
12. Funções security definer com search_path fixo e permissão de execução restrita ao papel previsto.
13. Funções de servidor validam o JWT e o papel antes de qualquer ação; CORS restrito ao domínio do app.
14. Eventos enviados ao Google contêm só procedimento, local e iniciais.

Ao final, liste o que não pôde ser verificado por você e precisa de teste manual.

NÃO ALTERAR: qualquer comportamento visível ao usuário que não seja uma falha da lista.
```

## Configuração manual fora do Lovable

**Painel do Supabase**

- [ ] Criar o projeto na região São Paulo (sa-east-1) e conectá-lo ao Lovable antes do Prompt 1.
- [ ] Authentication → Providers: ativar Google e Apple com as credenciais abaixo.
- [ ] Authentication → MFA: ativar TOTP.
- [ ] Authentication → URL Configuration: definir o domínio final do app em Site URL e Redirect URLs.
- [ ] Authentication: exigir confirmação de e-mail, senha mínima de 10 caracteres e proteção contra senhas vazadas.
- [ ] Authentication → SMTP: configurar um servidor de e-mail próprio; o padrão do Supabase tem limite baixo de envios.
- [ ] Secrets do servidor (no Lovable ou no Supabase, onde o projeto guardar): gravar client id e client secret do Google.
- [ ] Storage: conferir que os buckets exames e apas estão privados, com os limites de tipo e tamanho.
- [ ] Rodar o Security Advisor do Supabase e zerar os alertas.
- [ ] Ativar 2FA na sua própria conta do Supabase e do Lovable.

**Google Cloud**

- [ ] Criar um projeto, ativar a Google Calendar API e configurar a tela de consentimento OAuth.
- [ ] Criar credencial OAuth "Web application" com a URL de retorno do Supabase (login) e da função google-oauth (agenda).
- [ ] Adicionar o escopo calendar.events; por ser escopo sensível, o app precisa de verificação do Google para sair do modo de teste (limitado a 100 usuários cadastrados).

**Apple**

- [ ] Conta Apple Developer (paga, anual), Services ID, chave de "Sign in with Apple" e domínio verificado. A chave expira a cada 6 meses e precisa ser renovada no Supabase.

**Backups**

- [ ] Contratar um plano do Supabase com backup diário; avaliar Point-in-Time Recovery.
- [ ] Os backups do banco não incluem os arquivos do Storage: definir rotina própria de cópia dos buckets.
- [ ] Testar uma restauração antes de usar com pacientes reais.

**LGPD e regulação**

- [ ] Dados de saúde são dados sensíveis: definir base legal, texto do consentimento do formulário e política de privacidade com um advogado.
- [ ] Nomear o encarregado (DPO) e publicar o canal de contato do titular.
- [ ] Assinar o acordo de tratamento de dados (DPA) com Supabase, Lovable e Google.
- [ ] Definir prazo de guarda do prontuário (a regra do CFM é de no mínimo 20 anos) e o processo para pedidos de acesso, correção e exclusão.
- [ ] Escrever o plano de resposta a incidentes, com comunicação à ANPD e aos titulares.
- [ ] Verificar com o CFM/CRM e a SBIS os requisitos de prontuário eletrônico e de assinatura digital (ICP-Brasil) para a APA e para telemedicina.
- [ ] No Lovable, manter o projeto privado e não usar dados reais de pacientes durante os testes.

Confirme prazos e exigências legais com um advogado e com o seu conselho: não sou advogada nem consultora regulatória, e a lista acima é um ponto de partida.

## Decisões que tomei e pontos para validar

Onde o seu texto deixava em aberto, escolhi a opção mais simples e segura. Troque no prompt indicado se preferir outra.

| Ponto | O que os prompts fazem | Onde mudar |
| --- | --- | --- |
| Envio de links e do PDF | Sem integração de e-mail ou WhatsApp: a plataforma copia o link, abre o WhatsApp ou usa o compartilhamento do aparelho | Prompts 3, 9 e 10 |
| Google Agenda | Mão única, da plataforma para o Google; o evento leva só procedimento, local e iniciais | Prompt 4 |
| Agenda do cirurgião por link | Mostra horário, local, procedimento, anestesista e iniciais do paciente, não o nome | Prompt 1 (publico\_agenda) |
| OCR dos exames | Roda no navegador, sem serviço externo; a leitura de foto é imperfeita e o texto fica editável | Prompt 8 |
| CID | Campo de código e descrição, sem busca na tabela CID-10 completa | Prompt 6 |
| Telemedicina | Só registra a modalidade; não há videochamada embutida | Prompt 6 |
| Perguntas do CSAT | Cinco perguntas que eu propus, de 1 a 5; CSAT = % de notas 4 e 5 | Prompt 10 |
| Novos cadastros | Nascem como secretária inativa; o anestesista define o papel ao liberar | Prompt 1 |
| PDF enviado ao paciente | Só a folha de orientações; a APA completa fica no prontuário | Prompt 9 |

Três pontos pedem a sua conferência clínica antes do uso com pacientes:

- **Scores.** O Lovable escreve as fórmulas a partir das publicações; confira cada um dos oito com casos de resultado conhecido.
- **Jejum.** Os tempos padrão (8 / 6 / 4 / 2 h) são um ponto de partida editável em Configurações; ajuste ao seu protocolo.
- **Medicações.** A plataforma não sugere conduta; a linha do tempo só desenha o que você escolher.

Três decisões de arquitetura já vão fechadas no Prompt 1, para o Lovable não perguntar: papel e situação ficam na tabela papeis\_usuario, separada de perfis; arquivo-link é uma função de servidor que exige duas etapas e emite link de 60 segundos; e o login é registrado pela função registrar\_login, sem gatilho em auth.sessions, sendo obrigatório para qualquer acesso a dados.
