Permitir que a Passageira Saia da Carona (e a Dona Remova Passageiras)
Depende dos PRDs vou-junto e delete-rides
Contexto
Hoje, depois de confirmar presença em uma carona ("Vou junto"), não existe forma de desistir. Se os planos mudam, a vaga fica ocupada à toa, a dona conta com alguém que não vai, e a regra "já confirmou presença em uma carona neste dia" impede a passageira de entrar em outra carona no mesmo dia. A dona também não consegue tirar da carona quem avisou que não vai.

User Story
Como passageira inscrita em uma carona, quero poder sair dela caso não vá mais, para liberar a vaga e poder escolher outra carona.
Como dona de uma carona, quero poder remover uma passageira, para liberar a vaga de quem não vai.
Requisitos Funcionais
A lista de passageiros do card fica recolhida; quem quiser abre para ver os nomes.
Na lista aberta, a passageira vê uma lixeira no próprio nome para sair da carona.
Na lista aberta, a dona vê uma lixeira em cada nome para remover a passageira.
Antes de sair ou remover, um modal pede confirmação.
O endpoint só aceita o pedido da própria passageira ou da dona da carona — ninguém mais, mesmo chamando a API direto.
Ao sair ou ser removida, a vaga volta a ficar disponível e a passageira pode se inscrever em outra carona no mesmo dia.
Método	Rota	Resposta
DELETE	/ride-users/:rideId/users/:userId	204 sucesso · 401 sem login · 403 se não for a passageira nem a dona · 404 se a carona não existe ou a pessoa não está inscrita
Avisos: fora deste card. Quem sai não avisa a dona, e quem é removida não é avisada, até o PRD notify-ride-deleted ser feito (ele cobre os dois avisos).
Critérios de Aceite
A lista de passageiros começa recolhida e abre ao clicar
Apenas a própria passageira e a dona veem / conseguem usar a lixeira
O backend recusa (403) quem não é a passageira nem a dona
Depois de sair ou ser removida, a vaga aparece livre no mural e a passageira pode entrar em outra carona no mesmo dia
