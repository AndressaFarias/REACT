## Manipulando Vários Ambientes no React with Docker
https://levelup.gitconnected.com/handling-multiple-environments-in-react-with-docker-543762989783


Muito dos times usam _React_ em combinação com o Docker e Docker Compose para desenvolvimento local e ubernetes para deploy da aplicação.

Na maioria dos casos temos diversos ambientes ( dev, hml, prd, ... ). Muitos desenvolvedores do React, em algum momento, enfrentam o desafio de como gerenciar esses ambientes e fazer com que o aplicativo React aponte para o endpoint da API correto. 


### Qual é o problema com variáveis ​​de ambiente no React?

As variáveis ​​de ambiente são obviamente variáveis ​​definidas no ambiente do aplicativo e podem ser usadas para configurá-lo adequadamente, por exemplo, aponte um frontend para o endpoint da API correto.

