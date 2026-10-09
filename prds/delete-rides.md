Permitir Deletar Carona Somente se For a Dona
Depende dos PRDs 1 e 3
Contexto
O Soft Go hoje não tem nenhuma forma de excluir uma carona publicada por engano ou que não vai mais acontecer. Precisamos adicionar essa opção — mas só quem publicou pode excluir.

User Story
Como dona de uma carona, quero poder excluí-la caso ela não vá mais acontecer, para que ela suma da listagem de todo mundo.
Requisitos Funcionais
A ação de excluir aparece apenas para quem é a dona da carona (comparar owner_id com a usuária logada).
O endpoint verifica no backend se quem está pedindo a exclusão é realmente a dona — não basta esconder o botão no front.
Se não for a dona, retornar erro, mesmo que a pessoa tente chamar o endpoint diretamente.
Método	Rota	Resposta
DELETE	/rides/:id	204 sucesso · 403 se não for dona · 404 se a carona não existe
Decisão em aberto (a mais importante deste card): o que acontece com quem já confirmou presença (rides_people) quando a carona é excluída? Duas opções razoáveis: apagar os participantes junto (cascade), ou impedir a exclusão se já houver gente confirmada. Essa é uma decisão de produto de verdade — proponha uma e justifique sua escolha, não implemente sem pensar a respeito.
Critérios de Aceite
Apenas a dona vê / consegue usar a opção de excluir no front
O backend valida a posse mesmo se o front for contornado (ex: chamando a API direto)
Tentativa de exclusão por quem não é dona retorna erro apropriado (403), não sucesso silencioso
Existe uma decisão clara — e documentada — sobre o que acontece com participantes já confirmados