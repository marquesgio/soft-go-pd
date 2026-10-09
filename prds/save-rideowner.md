Salvar o Dono da Carona ao Criar
Depende do PRD 1
Contexto
Hoje quem publica uma carona digita nome e telefone manualmente no formulário — não existe vínculo real entre a carona e uma usuária do sistema. Isso precisa mudar: toda carona publicada deve ficar associada a quem a criou.

User Story
Como usuária logada, quero que a carona que eu publico fique registrada no meu nome automaticamente, sem eu precisar digitar meus dados de novo.
Requisitos Funcionais
Remover os campos nome/telefone do formulário de publicar carona — usar os dados da usuária logada.
Ao criar a carona, salvar automaticamente quem é a dona.
Exibir o nome da dona no card da carona na listagem (hoje já mostra um nome, mas vindo de outro lugar).
Modelo de Dados
-- rides ganha um dono de verdade
ALTER TABLE rides ADD COLUMN owner_id INTEGER REFERENCES users(id);
-- os campos name/phone que existiam antes deixam de ser usados na criação
Atenção na migration: já podem existir caronas publicadas sem owner_id (dados de teste, criados sem login). Vale pensar se a migration limpa esses dados antigos ou se tenta popular owner_id de outra forma. Para este exercício, está tudo bem assumir uma base zerada.
Critérios de Aceite
Publicar carona não pede mais nome/telefone no formulário
Carona criada fica corretamente associada à usuária logada
Listagem mostra o nome da dona vindo da tabela users, não mais de um campo solto
Usuária não logada não consegue publicar carona