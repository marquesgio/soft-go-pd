Revogar Sessões ao Sair
Depende do PRD 1 (auth)
Contexto
Hoje o "Sair" só apaga o token do navegador. A API aceita qualquer token com assinatura válida até ele expirar (7 dias), porque não guarda nenhuma informação sobre sessões. Uma cópia do token (roubada por um script malicioso, já que ele fica no localStorage, ou esquecida num computador emprestado) continua publicando caronas, inscrevendo e vendo telefones mesmo depois de a usuária sair. Agora que o token libera ações e dados pessoais (features vou-junto e save-rideowner), isso precisa mudar.

User Story
Como usuária, quero que, ao clicar em "Sair", nenhum token antigo da minha conta continue funcionando, em nenhum aparelho, para que ninguém use minha sessão depois que eu saí.
Requisitos Funcionais
Cada conta ganha uma versão de sessão. Todo token emitido (cadastro e login) carrega a versão atual da conta.
A API só aceita um token se a versão dentro dele for igual à versão atual da conta. Vale para as rotas que exigem login e para as que só identificam quem está logada.
"Sair" chama a API, que incrementa a versão da conta: todos os tokens emitidos antes deixam de valer, em todos os aparelhos.
Entrar de novo funciona normalmente e emite um token com a versão nova. A conta, as caronas, as inscrições e o telefone não mudam.
Trocar de conta continua sendo sair e entrar.
Modelo de Dados
ALTER TABLE users ADD COLUMN token_version INTEGER NOT NULL DEFAULT 0;
Decisão tomada: "Sair" encerra a sessão em todos os aparelhos. Não existe "sair só deste aparelho" (isso exigiria guardar cada sessão no servidor).
Atenção no deploy: tokens emitidos antes desta feature não têm versão. Eles deixam de valer, e todas as usuárias precisam entrar de novo uma vez.
Fora de escopo: tela de troca de senha (quando existir, deve incrementar a versão), lista de aparelhos conectados, sair de um aparelho só, refresh token, cookie httpOnly.
Critérios de Aceite
Depois de "Sair", o token antigo recebe 401 nas rotas que exigem login
Depois de "Sair", o token antigo é tratado como deslogado no mural (sem telefones)
Sair num aparelho derruba a sessão dos outros aparelhos da mesma conta
Entrar de novo depois de sair funciona, sem perder nada da conta
Sair de uma conta não afeta os tokens de outra conta
