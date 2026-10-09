Contexto
Hoje o Soft Go não tem login. Quem publica uma carona ou confirma presença digita nome e telefone manualmente toda vez, sem nenhuma validação de identidade. Isso precisa mudar: precisamos de cadastro e login antes de liberar as próximas funcionalidades.

User Story
Como usuária do Soft Go, quero criar uma conta e fazer login, para que o sistema saiba quem eu sou sem eu precisar digitar meus dados toda vez.
Requisitos Funcionais
Tela de Cadastro (Sign Up): nome, e-mail, senha e confirmação de senha — validar e-mail único e um tamanho mínimo de senha.
Tela de Login (Sign In): e-mail + senha — em caso de erro, mensagem genérica (não revelar se o problema foi o e-mail ou a senha).
Sessão: usuária autenticada permanece logada (token/JWT) sem precisar logar de novo a cada ação.
Logout: precisa existir um jeito de sair da conta.
Modelo de Dados
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(120) NOT NULL,
  email VARCHAR(160) UNIQUE NOT NULL,
  password_hash VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT NOW()
);
Atenção na revisão: senha nunca deve ser salva em texto puro — sempre com hash (ex: bcrypt). Se a IA gerar código salvando password direto no banco, isso é motivo de reprovar a review mesmo que "funcione".
Fora de escopo: recuperação de senha, verificação de e-mail e login social (Google etc). Deixe isso explícito no seu prompt para a IA não tentar implementar por conta própria.
Critérios de Aceite
Cadastro rejeita e-mail duplicado com mensagem clara
Senha é armazenada com hash, nunca em texto puro
Login com credenciais erradas retorna erro sem revelar se foi o e-mail ou a senha
Depois do login existe um token/sessão que identifica a usuária nas próximas requisições
Existe uma forma de deslogar