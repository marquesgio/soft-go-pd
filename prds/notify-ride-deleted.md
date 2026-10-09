Avisar Quando a Carona ou a Inscrição Mudar
Depende dos PRDs delete-rides e leave-ride
Contexto
Três ações tiram alguém de uma carona sem que a outra parte saiba: a dona exclui a carona (as inscrições saem junto), a passageira sai da carona, ou a dona remove uma passageira. Hoje o único aviso é por fora, pelo WhatsApp, e isso só funciona para quem cadastrou telefone. Quem não tem telefone só percebe ao abrir o mural, às vezes já no dia de ir para a Soft.

User Story
Como participante, quero ser avisada quando a carona for excluída ou quando eu for removida dela, para ter tempo de procurar outra forma de ir até a Soft.
Como dona, quero ser avisada quando uma passageira sair da minha carona, para saber que a vaga abriu.
Requisitos Funcionais
Carona excluída: cada participante recebe um aviso, tenha ou não telefone cadastrado.
Passageira removida pela dona: a passageira recebe um aviso.
Passageira saiu: a dona recebe um aviso.
O aviso identifica a carona (data, hora, cidade de saída, dona) e, nos casos de saída/remoção, a pessoa.
O aviso é enviado pelo sistema no momento da ação, sem depender de ninguém.
Uma falha ao enviar o aviso não impede a ação nem devolve erro para quem a fez.
Decisão em aberto: qual canal usar? Opções razoáveis: e-mail (toda conta já tem e-mail de login, mas exige um serviço de envio), aviso dentro do app (exibido no próximo acesso) ou os dois. Proponha um e justifique.
Critérios de Aceite
Os três eventos geram aviso para a pessoa certa, inclusive quem não tem telefone
O aviso identifica a carona e, quando for o caso, a pessoa
A ação continua funcionando mesmo se o envio do aviso falhar
Existe uma decisão documentada sobre o canal do aviso
