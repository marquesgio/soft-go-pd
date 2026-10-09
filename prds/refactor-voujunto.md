Contexto
Hoje o modal "Quero ir junto!" pede nome e telefone toda vez, mesmo que a pessoa já tenha usado o Soft Go antes. Com login, isso não faz mais sentido — o sistema já sabe quem é a usuária.

User Story
Como usuária logada, quero confirmar presença em uma carona com um clique, sem preencher meus dados de novo, para confirmar mais rápido.
Requisitos Funcionais
Remover os campos de nome/telefone do modal — usar diretamente os dados da usuária logada.
O botão "Vou junto" só funciona para quem está autenticada (se não estiver logada, redirecionar para o login).
Impedir duplicidade: a mesma usuária não pode confirmar presença duas vezes na mesma carona.
Modelo de Dados
-- rides_people passa a referenciar um usuário real
ALTER TABLE rides_people ADD COLUMN user_id INTEGER REFERENCES users(id);
-- (name e phone digitados manualmente deixam de ser usados aqui)
ALTER TABLE rides_people ADD CONSTRAINT uq_ride_user UNIQUE (ride_id, user_id);
Por que isso é novo agora: antes, sem login, não tinha como saber se duas pessoas com o mesmo nome eram a mesma pessoa — não dava pra bloquear duplicidade de verdade. Com usuário autenticado, virou uma regra de negócio que só faz sentido a partir daqui.
Decisão em aberto: o telefone continua sendo exibido para a dona da carona entrar em contato? Se sim, de onde ele vem — de um campo novo em users, ou continua sendo digitado à parte no momento da confirmação? Vale decidir e documentar a escolha antes de prompar a IA.
Critérios de Aceite
Modal não pede mais nome/telefone — usa a usuária logada
Usuária não logada não consegue confirmar presença (é redirecionada ou bloqueada)
A mesma usuária não consegue confirmar presença duas vezes na mesma carona
O participante aparece corretamente associado à usuária certa na listagem