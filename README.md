🚀 ChatTasker - Bot de Gerenciamento de Tarefas para o Google Chat
O ChatTasker é um bot para o Google Chat (Workspace) que ajuda equipes a organizar, acompanhar e concluir tarefas diretamente na conversa. Ele envia lembretes automáticos, cobra usuários que não enviaram suas tarefas e garante que todos estejam alinhados ao longo do dia.

Com comandos simples, você pode manter a produtividade do time sem precisar cobrar manualmente cada membro do grupo.

📌 Passo a Passo para Implementação

1️⃣ Criar o Código no Google Apps Script
Acesse o Google Apps Script.
Clique em Novo Projeto.
No editor de código, crie um arquivo chamado Codigo.gs.
Cole o código do bot no arquivo Codigo.gs (veja o código completo abaixo).

const CHAT_WEBHOOK_URL = "SEULINKWEBHOOK";

function onMessage(event) {
  Logger.log("== Evento recebido ==");
  Logger.log(JSON.stringify(event));

  if (event.type === "REMOVED_FROM_SPACE") {
    onRemovedFromSpace(event);
    return;
  }

  let nome = event.user && event.user.displayName
    ? normalizarNome(event.user.displayName)
    : "Usuário Desconhecido";

  let userId = event.user && event.user.name ? event.user.name : null;

  Logger.log("Nome capturado: " + nome);
  Logger.log("ID capturado: " + (userId || "Nenhum ID encontrado"));

  adicionarUsuarioNaLista(nome, userId);

  const mensagem = event.message ? event.message.text : "";
  
  const mencion = userId
    ? `<${userId}>`
    : `@${nome}`;

  if (mensagem.length < 160) {
    return { text: `🚨 ${mencion}, sua descrição deve conter no mínimo 160 caracteres! Ou 2,5 linhas.` };
  }

  const usuariosEnviaram = listarUsuariosEnviaram();
  if (usuariosEnviaram.includes(nome)) {
    return { text: `✅ ${mencion}, você já enviou sua task hoje!` };
  }

  salvarUsuario(nome);

  const todosEnviaram = verificarTodosEnviaram();
  if (todosEnviaram) {
    enviarParaGoogleChat("🚀 Todos os membros já enviaram suas tarefas de hoje. Bom trabalho!");
    const props = PropertiesService.getScriptProperties();
    props.setProperty("allDoneForToday", "true");
  }

  return { text: `✅ Obrigado, ${mencion}, sua task foi registrada com sucesso!` };
}

function testeLocal() {
  const mockEvent = {
    user: { displayName: "Fulano", name: "users/12345678" },
    message: { text: "Teste local com 200 caracteres" },
    type: "MESSAGE"
  };
  onMessage(mockEvent);
}

//Adiciona o usuário na lista de para comparação de []
function adicionarUsuarioNaLista(nome, userId) {
  const props = PropertiesService.getScriptProperties();
  let todosUsuarios = listarTodosUsuarios();
  let usuariosMap = listarMapeamentoUsuarios();

  if (!todosUsuarios.includes(nome)) {
    todosUsuarios.push(nome);
    props.setProperty("todosUsuarios", JSON.stringify(todosUsuarios));
  }

  // Mapeamento de nome e ID
  if (userId && !usuariosMap[nome]) {
    usuariosMap[nome] = userId;
    props.setProperty("usuariosMap", JSON.stringify(usuariosMap));
    Logger.log(`ID "${userId}" salvo para "${nome}"`);
  }
}

// Lista de usuários para cobrança
function listarTodosUsuarios() {
  const props = PropertiesService.getScriptProperties();
  const usuariosString = props.getProperty("todosUsuarios");
  return usuariosString ? JSON.parse(usuariosString) : [];
}

// nome + ID
function listarMapeamentoUsuarios() {
  const props = PropertiesService.getScriptProperties();
  const usuariosMapString = props.getProperty("usuariosMap");
  return usuariosMapString ? JSON.parse(usuariosMapString) : {};
}

// Lista de quem já enviou
function listarUsuariosEnviaram() {
  const props = PropertiesService.getScriptProperties();
  const usuariosString = props.getProperty("usuariosEnviaram");
  return usuariosString ? JSON.parse(usuariosString) : [];
}

//Nome de usuário de enviou
function salvarUsuario(nome) {
  const props = PropertiesService.getScriptProperties();
  let usuariosEnviaram = listarUsuariosEnviaram();

  if (!usuariosEnviaram.includes(nome)) {
    usuariosEnviaram.push(nome);
    props.setProperty("usuariosEnviaram", JSON.stringify(usuariosEnviaram));
  }
}

// Compara os arrays para ver se todos enviaram
function verificarTodosEnviaram() {
  const todosUsuarios = listarTodosUsuarios();
  const usuariosEnviaram = listarUsuariosEnviaram();
  return todosUsuarios.every(usuario => usuariosEnviaram.includes(usuario));
}

// envia mensagem ao gchat
function enviarParaGoogleChat(mensagem) {
  const payload = JSON.stringify({ text: mensagem });
  const options = {
    method: "post",
    contentType: "application/json",
    payload: payload,
  };
  UrlFetchApp.fetch(CHAT_WEBHOOK_URL, options);
}

//Função que ouve quando alguém é removido do grupo
function onRemovedFromSpace(event) {
  if (event.user && event.user.displayName) {
    const nome = normalizarNome(event.user.displayName);
    removerUsuarioDaLista(nome);
  }
}

//Se for removido o bot remove das listas salvas
function removerUsuarioDaLista(nome) {
  const props = PropertiesService.getScriptProperties();
  let todosUsuarios = listarTodosUsuarios();
  let usuariosEnviaram = listarUsuariosEnviaram();
  let usuariosMap = listarMapeamentoUsuarios();

  // Remove o usuário de todas as listas
  todosUsuarios = todosUsuarios.filter(usuario => usuario !== nome);
  usuariosEnviaram = usuariosEnviaram.filter(usuario => usuario !== nome);
  delete usuariosMap[nome];

  // Atualiza as propriedades
  props.setProperty("todosUsuarios", JSON.stringify(todosUsuarios));
  props.setProperty("usuariosEnviaram", JSON.stringify(usuariosEnviaram));
  props.setProperty("usuariosMap", JSON.stringify(usuariosMap));
}

// Mensagem de bom dia, com condição para não enviar fins de semana.
function enviarMensagemBomDia() {
  const diaDaSemana = new Date().getDay();
  if (diaDaSemana === 0 || diaDaSemana === 6) {
    return; // Evita envio aos fins de semana
  }

  const mensagem = `
☀️ Bom dia! Lembre-se de enviar sua task hoje.
O formato é: @ChatBot seguido de sua descrição (mínimo 160 caracteres).
Envie até meio-dia! 😊
  `;
  enviarParaGoogleChat(mensagem);
}

//Cobrança ao meio dia e as 14 da tarde.
function enviarCobranca() {
  const diaDaSemana = new Date().getDay();
  if (diaDaSemana === 0 || diaDaSemana === 6) return;

  const props = PropertiesService.getScriptProperties();
  if (props.getProperty("allDoneForToday") === "true") return;

  const todosUsuarios = listarTodosUsuarios();
  const usuariosEnviaram = listarUsuariosEnviaram();
  const usuariosMap = listarMapeamentoUsuarios();

  const usuariosPendentes = todosUsuarios.filter(usuario => !usuariosEnviaram.includes(usuario));

  if (usuariosPendentes.length > 0) {
    const mencoes = usuariosPendentes.map(nome => {
      const id = usuariosMap[nome];
      return id ? `<${id}>` : `@${nome}`;
    }).join(", ");

    const mensagem = `🚨 Atenção! Estes membros ainda não enviaram a tarefa de hoje: ${mencoes}. Por favor, enviem o quanto antes!`;
    enviarParaGoogleChat(mensagem);
  }
}

//Aviso as 19 da noite de quem não enviou no dia.
function enviarCobranca19() {
  const diaDaSemana = new Date().getDay();
  if (diaDaSemana === 0 || diaDaSemana === 6) return;

  const props = PropertiesService.getScriptProperties();
  if (props.getProperty("allDoneForToday") === "true") return;

  const todos = listarTodosUsuarios();
  const enviados = listarUsuariosEnviaram();
  const usuariosMap = listarMapeamentoUsuarios();

  const pendentes = todos.filter(nome => !enviados.includes(nome));

  if (pendentes.length > 0) {
    const mencoes = pendentes.map(nome => {
      const id = usuariosMap[nome];
      return id ? `<${id}>` : `@${nome}`;
    }).join(", ");

    const mensagem = `🚨 Estes membros não enviaram a tarefa de hoje: ${mencoes}`;
    enviarParaGoogleChat(mensagem);
  }
}

// Limpeza diária as 23 da noite.
function limparUsuarios() {
  const props = PropertiesService.getScriptProperties();
  props.deleteProperty("usuariosEnviaram");

  // Reseta o "allDoneForToday" para permitir cobranças no novo dia
  props.deleteProperty("allDoneForToday");
}

// Formata os nomes de usuário para evitar duplicatas
function normalizarNome(nome) {
  return nome.trim().toLowerCase();
}

//Gatilhos para os acionadores
function configurarGatilhos() {

  // Bom dia às 7h
  ScriptApp.newTrigger("enviarMensagemBomDia")
    .timeBased()
    .everyDays(1)
    .atHour(7)
    .create();

  // Cobrança (manhã) às 12h
  ScriptApp.newTrigger("enviarCobranca")
    .timeBased()
    .everyDays(1)
    .atHour(12)
    .create();

  // Cobrança (tarde) às 14h
  ScriptApp.newTrigger("enviarCobranca")
    .timeBased()
    .everyDays(1)
    .atHour(14)
    .create();

  // Cobrança (noite) às 19h, mensagem diferente
  ScriptApp.newTrigger("enviarCobranca19")
    .timeBased()
    .everyDays(1)
    .atHour(19)
    .create();

  // Limpeza diária às 23h
  ScriptApp.newTrigger("limparUsuarios")
    .timeBased()
    .everyDays(1)
    .atHour(23)
    .create();
}

Vá até Configurações e ative o Manifesto, isso adicionará o arquivo appsscript.json.
No arquivo appsscript.json, cole o seguinte código:
json
Copiar
Editar
{
  "timeZone": "America/Sao_Paulo",
  "dependencies": {
    "enabledAdvancedServices": [
      {
        "userSymbol": "Chat",
        "version": "v1",
        "serviceId": "chat"
      }
    ]
  },
  "exceptionLogging": "STACKDRIVER",
  "runtimeVersion": "V8",
  "chat": {},
  "webapp": {
    "executeAs": "USER_ACCESSING",
    "access": "DOMAIN"
  }
}

Salve todas as alterações.

2️⃣ Criar um Projeto no Google Cloud
Acesse o Google Cloud Console.
Crie um novo projeto e escolha um nome para ele (exemplo: ChatTasker).
Certifique-se de estar utilizando o mesmo e-mail que usa no Google Apps Script e no Google Chat.
Copie o ID do projeto recém-criado.
Volte ao Google Apps Script, vá para Configurações e cole o ID do Projeto na seção "Projeto do Google Cloud Platform (GCP)".

3️⃣ Configurar a Tela de Permissão OAuth
No Google Cloud, acesse APIs e Serviços > Tela de Permissão OAuth.
Configure:
Nome do aplicativo: ChatTasker (mesmo nome do projeto para evitar confusão).
E-mail de suporte: Insira seu e-mail.
Logotipo: Pode ser qualquer imagem da sua escolha.
Usuários autorizados: Adicione o seu próprio e-mail.
Clique em Salvar e continuar.

4️⃣ Ativar e Configurar a Google Chat API
Vá para APIs e Serviços > APIs ativadas.
Ative a Google Chat API.
Acesse Configuração e preencha os seguintes campos:
Nome do Bot: ChatTasker
URL do Avatar: Insira a URL de uma imagem (exemplo: um robô ou um ícone personalizado).
Descrição: "Bot de tarefas para o Google Chat".
Recursos Interativos: Ative "Receber mensagens individuais" e "Participar de espaços e grupos".
Em Configurações de Conexão, selecione Apps Script.
Vá até o Google Apps Script, copie o ID de Implantação e cole no campo ID de Implantação do Google Cloud.
Marque a opção "Disponibilizar o bot para pessoas e grupos específicos".
Ative o registro de erros no Logging.
Salve as configurações.

5️⃣ Criar a Implantação do Bot
No Google Apps Script, clique em Implantar > Nova Implantação.
Escolha App da Web.
No campo Quem pode acessar, selecione:
"Qualquer pessoa no grupo" (se for interno).
"Qualquer pessoa com o link" (se for público).
Ao implantar, será gerado um Código de Implantação.
Copie este código e cole na Configuração do Google Cloud (ID de Implantação).
Clique em Salvar e ativar o bot.

6️⃣ Adicionar o Bot no Google Chat
Vá até o grupo do Google Chat onde deseja adicionar o bot.
Clique em Configurações > Apps e Integrações.
Procure pelo nome do bot (ChatTasker) e adicione ao grupo.
Vá até Webhooks, crie um novo Webhook e configure:
Nome do Webhook: ChatTasker
Imagem do Avatar: A mesma utilizada para o bot.
Copie o link gerado pelo Webhook.
No código Codigo.gs, substitua:
javascript
Copiar
Editar
const CHAT_WEBHOOK_URL = "SEUWEBHOOK";
por:

javascript
Copiar
Editar
const CHAT_WEBHOOK_URL = "LINK_DO_WEBHOOK_COPIADO";
Salve e execute o código.

7️⃣ Configurar Acionadores
No Google Apps Script, vá para Gatilhos e configure:

enviarMensagemBomDia() → 7h.
enviarCobranca() → 12h e 14h.
enviarCobranca19() → 19h.
limparUsuarios() → 23h.
Exclua acionadores antigos e configure a versão atual do projeto:

Vá até Implantar > Gerenciar Implantações e copie a versão atual.
Configure o acionador limparUsuarios() para rodar entre 23h e 00h.

8️⃣ Testando o Bot
No Google Chat, envie uma mensagem marcando o bot @ChatTasker.
Se for a primeira vez, clique em Configuração e permita o acesso.
Envie uma tarefa com pelo menos 160 caracteres.
O bot responderá confirmando o registro.
Teste removendo um usuário do grupo para verificar se ele é removido da lista de controle.
Aguarde os horários de cobrança para ver se os lembretes são enviados.

🎯 Configurações Avançadas
Personalização de Mensagens
Altere os textos de alerta e notificações no código.
Reduza o número mínimo de caracteres para tarefas alterando:
javascript
Copiar
Editar
if (mensagem.length < 160)
para um valor menor (exemplo: if (mensagem.length < 100)).

Alteração de Horários
Modifique os horários de cobrança no final do código.
Ajuste os acionadores conforme necessário.
Automação Completa
Após configurar e testar, o bot funcionará automaticamente: 
✅ Envia mensagem de bom dia e lembra da tarefa.

✅ Cobra os membros do grupo ao longo do dia.

✅ Notifica quem não enviou no final do dia.

✅ Limpa os dados à noite para o próximo dia.

🎉 Conclusão

Agora você tem um bot de gerenciamento de tarefas 100% funcional no Google Chat! 🚀

Esse bot ajuda no controle de produtividade de equipes, reduz a necessidade de cobranças manuais e organiza melhor o fluxo de trabalho.
