Shell Remotas. Parte I

--> Introdução 
	
Em um mondo conectado, o acesso remoto a computadores acontece o tempo todo. Você usa serviços como ssh ou telnet para essa finalidade, mas, às vezes, eles não estão disponíveis ou não é possível até mesmo implantar esses serviços no dispositivo de destino. Nesses casos, você pode facilmente escrever seu próprio programa...

[ NÍVEL: INICIANTE ]

--> Casos de Uso de Shell remota

Eu escrevi e implantei shells remotas algumas vezes por razões completamente legítimadas em minhas próprias CPUs. Por exemplo, uma vez eu precisei de acesso shell ao meu Android para depurar algum outro aplicativo. Eu tentei servidores SSH, mas eles estavam com um desempenho ruim, então acabei implantando um pequeno binário para acessá-lo remotamente.

Do ponto de vista de segurança, uma shell remota geralmente faz parte de um shellcode para permitir o acesso remoto não autorizado a um sistema.

--> Tipos de Shell remotas

Existem basicamente duas maneiras de obter acesso remoto a shell:

	* Shell Remotas Diretas: Se comporta como um servidor. Funciona como um servidor ssh/telnet. O usuário/atacante remoto se conecta a uma porta específica na máquina de destino e obtém acesso automático a uma shell.

	* Shell Remotas Reversas: Funcionam ao contrário. A aplicação em execução na máquina de destino se conecta de volta (chamadas de homeback/callback) a um servidor e porta específicos em uma máquina que pertence ao usuário/atacante.

O método inverso da shell tem algumas vantagens.
	
	* Os firewalls geralmente bloqueiam conexões de entrada, mas permitem conexões de saída para fornecer acesso à Internet aos usuários da máquina.
	* O usuário/atacante não precisa saber o IP da máquona que está executando a shell remota, mas precisa possuir um sistema com um IP fixo, para permitir que o alvo "ligue para casa" (callback).
	* Geralmente há muitas conexões de saída em uma máquina e apenas alguns servidores (se houver) em execução. Isso torna a detecção um pouco mais difícil, especialmente se a shell se conectar a algo que esteja executando na porta 80...


--> Networking -> O Cliente

Começaremos escrevendo um "servidor" mínimo e funções do "cliente" para que possamos experimentar os shells diretos e reversos. 

<a href="https://github.com/carloserocha/Exploits/blob/master/main_socket.c#L7">client</a><br>

Primeiros estabelecemos uma conexão com um endereço IP e porta específicos. A função recebe como parâmetros um endereço IP para conexão e uma porta. Em seguida, ele cria um socket TCP (SOCK_STREAM) e preenche os dados para conexão. É efetivamente estabelecida a conexão após uma execução bem-sucedida de conexão. Em caso de qualquer erro (criar o socket ou conexão), nós simplesmente paramos a aplicação.

Esta função nos permitirá implementar uma shell remota reversa.

--> Networking -> O Servidor

A função do servidor é um pouco mais longa... mas é tão simples quanto a do cliente. 

<a href="https://github.com/carloserocha/Exploits/blob/master/main_socket.c#L24">server</a><br />

O começo da função é praticamente o mesmo para o código do cliente. Ele cria um socket, preenche os dados da rede, mas ao invés de tentar se conectar a um servidor remoto, ele liga o socket a uma porta específica. Observer que o endereço passado para "vincular" é constante INADDR_ANY. Este é, na verdade, o IP 0.0.0.0 e significa que o socket estará escutando em todas as interfaces.

O "ligamento" da chamada de sistema realmente não faz a tomada de uma "escuta" socket (você pdoe realmente chamar "ligamento" em um socket client). É o sistema de "escuta" que chama aquele que faz do Socket, um Socket de servidor. O segundo parâmetro passado para "escutar" é o backlog. Basicamente, ele indica quantas conexões serão enfileiradas para serem aceitar antes que o servidor comece a rejeitar conexões. No nosso caso, isso realmente não importa.

Neste ponto, nosso servidor está configurado e podemos aceitar conexões. A chamada para a chamada de sistema "aceita" faŕa com que o nosso servidor aguarde por uma conexão de entrada. Sempre que chegar, um novo socket será criado para trocar dados com o novo cliente.

--> Iniciando uma Shell

A última parte do nosso exemplo de shell remota é uma função para iniciar uma shell...

<a href="https://github.com/carloserocha/Exploits/blob/master/main_socket.c#L54">start</a><br />

OBS: ESSE CÓDIGO NÃO IRÁ FUNCIONAR SOZINHO. É APENAS UMA DEMONSTRAÇÃO DELE.

Mais uma vez, a função é bem simples. Ele faz uso de duas chamadas de sistema dup2 e execv . O primeiro duplica um determinado descritor de arquivo. Neste caso, as três chamadas no início da função, atribui o descritor de arquivo recebido como parâmetro à entrada padrão (descritor de arquivo 0), saída padrão (descritor de arquivo 1) e erro padrão (descritor de arquivo 3).

Portanto, se o descritor de arquivo que passamos como parâmetro for um dos Sockets criados com nossas funções de cliente e servidor anteriores, estaremos efetivamente enviando e recebendo dados pela rede toda vez que gravarmos dados no console e lermos dados de stdin.

Agora apenas executamos um shell com o sinalizador -i (modo interativo). A chamada de sistema execv substituirá o processo atual (cujo stdin, stdout e stderr estão associados a uma conexão de rede) pelo que foi passado como parâmetro.

Isso é basicamente isso. Nós só precisamos de uma função principal para testar nosso aplicativo shell remoto.

--> A Main

A principal função é bem simples. Observe que não estou verificando os argumentos da linha de comando. Isso significa que, se você não passar os argumentos corretos, o aplicativo falhará. É assim que a função principal se parece:

<a href="https://github.com/carloserocha/Exploits/blob/master/main_socket.c#L71">main</a><br />

O programa espera ter um primeiro argumento de uma letra. Se o argumento for o caractere 'c', ele iniciará um shell remoto reverso (executando o código do cliente) conectando-se novamente ao endereço IP passado como segundo argumento e a porta passada como terceiro argumento.

Caso contrário, ele é executado no modo de servidor (shell remoto direto) e usa o segundo argumento como a porta a ser ligada.

--> Testando...

Então, vamos testar nosso pequeno programa. Você pode apenas compilar usando make.

	make main_socket

Para os testes, precisaremos de dois terminais.

Primeiro, vamos testar o shell remoto direto. Em um terminal, você precisa iniciar o aplicativo no modo de servidor:

	# ./main_socket s 9000

Isso iniciará um servidor TCP aguardando conexões na porta 5000. Agora, a partir de outro terminal, use o netcat para conectar-se ao servidor:

	# nc localhost 9000

É isso aí. É melhor se você executar o comando netcat a partir de um diretório diferente, caso contrário, parecerá que nada aconteceu.
Deixe a sessão digitando exit no seu terminal netcat ou pressionando CTRL + D e vamos tentar o shell remoto reverso.

Agora começamos em um dos terminais um netcat no modo de servidor. Quando executamos nosso aplicativo no sistema de destino, ele se conectará a essa janela do netcat .

	# nc -l -p 9000

No outro terminal, iniciaremos a shell reversa dessa forma:

	# ./main_socket c 127.0.0.1 9000

Você receberá imediatamente um aviso no seu terminal netcat e acessará a máquina de destino que acabou de chamar de volta para casa.


OBS: <a href="https://github.com/carloserocha/Exploits/blob/master/shell_remote.c">Pode ser usado para criar o servidor LISTEN somente</a><br />
 <h3>--> FIM.</h3>

													09/07/2018


