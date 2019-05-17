JASmine WEB - CUPS

Esse tutorial foi realizado com a intenção de adicionar funcionalidades ao sistema já existente JASmine, que gera relatórios a partir das impressoras instaladas no CUPS.

Nessa versão o JASmine tem a opção de listar os usuários ,impressoras, servidores e mostrar qual foi o usuário, impressora ou servidor que mais imprimiu, além de estatísticas de impressão.

Versão atual:

        Usuários;
        Impressoras;
        Servidores;

Não serão abordadas: a instalação do sistema operacional(Ubuntu 16.04) e configuração de rede. 

Instalação

1 - Executar os processos de instalação dos pacotes.

    $ sudo apt-get install apache2 vim build-essential cups cups-pdf apparmor-utils pkpgcounter samba mysql-server-5.7 php-mbstring php7.0-mbstring php-gettext phpmyadmin libdbd-mysql-perl

2 - Configurando o cups.

    $ sudo cp /etc/cups/cupsd.conf /etc/cups/cupsd.conf.bkp
    $ sudo vim /etc/cups/cupsd.conf

Em "Listen localhost:631" altere para:

    Listen 631

Vamos agora dar permissão para acesso ao CUPS,  altere as linhas conforme abaixo:

    Restrict access to the server...
    <Location />
    Allow all
      Order allow,deny
    </Location>

    Restrict access to the admin pages...
    <Location /admin>
    Allow all
      Order allow,deny
    </Location>

    Restrict access to configuration files...
    <Location /admin/conf>
    Allow all
      AuthType Default
      Require user @SYSTEM
      Order allow,deny
    </Location>

3 - Configurando o samba.

    $ sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.bkp

Criando uma configuração simples no Samba para o funcionamento correto da impressora:

    $ sudo vim /etc/samba/smb.conf

    configuração do samba aqui.

Ajustando as permissões de acesso

    $ sudo smbpasswd -a root

obs: colocar o hostname do servidor em maiusculo.

    $ sudo net -S localhost -U root -W HOSTNAME-DO-SERVIDOR rpc rights grant 'root' SePrintOperatorPrivilege

Edite o arquivo cups-pdf

    $ sudo vim /etc/cups/cups-pdf.conf

Comente a linha 45

    Out ${HOME}/PDF

Altere para: 

    Out /var/spool/cups-pdf/${USER}

Descomente as linhas:

    Label 1  - Linha 99 
    Log /var/log/cups  - Linha 202
    DecodeHexStrings 1 - Linha 280 
 
 4 - Jasmine
 
    $ sudo wget http://nayco3.free.fr/Jasmine/Releases/0.0.3/JASmine-MySQL-0.0.3.tar.bz2
    $ sudo wget http://nayco3.free.fr/Jasmine/Releases/0.0.3/JASmine-Backend-0.0.3.tar.bz2
    $ sudo wget http://nayco3.free.fr/Jasmine/Releases/0.0.3/JASmine-Web-0.0.3.tar.bz2 

A primeira etapa é criar o banco que vai armazenar os dados de impressão.

    sudo mysql -u root -p
    password: *****
    mysql> CREATE DATABASE print;
    mysql> exit

Agora vamos utilizar o script contido no arquivo JASmine-MySQL.

    $ sudo tar -jxvf JASmine-MySQL-0.0.3.tar.bz2
    cd JASmine-MySQL-0.0.3
    $ sudo mysql -u root -p print < jasmine.sql

Crie um usuário no MySQL para gerenciar o banco de impressão.

    $ sudo mysql -u root -p
    mysql> GRANT ALL ON print.* TO jasmine@localhost identified by 'senha';
    mysql> FLUSH PRIVILEGES;
    mysql> exit
 
 Dentro do JASmine-Backend existem os programas auxiliares que irão monitorar as impressões e gerar os dados a serem salvos no banco. Na pasta do JASmine-Backend.

    $ sudo tar -jxvf JASmine-Backend-0.0.3.tar.bz2
    cd JASmine-Backend-0.0.3

Existe um script em Perl chamado jasmine que deverá ser copiado para dentro do CUPS.

    $ sudo cp jasmine /usr/lib/cups/backend/
    cd /usr/lib/cups/backend
    $ sudo chmod 755 jasmine
    $ sudo chown root jasmine

Agora temos que editar nosso script para colocar as informações referentes ao banco de dados.

    $ sudo vim jasmine

    my $DBhost="localhost";
    my $DBlogin="jasmine";
    my $DBpassword="senha";
    my $Dbdatabase="print";
 
 O JASmine-Web é a página que coleta as informações e as exibe na Web, volte para pasta onde você baixou os arquivos.

    cd Impressao/
    $ sudo mkdir /var/www/html/Impressao
    $ sudo cp -r * /var/www/html/Impressao

Neste momento iremos editar o arquivo com as configurações de acesso a banco.

    cd /var/www/html/Impressao
    $ sudo vim conexao.php

    $DB_host="localhost";
    $DB_login="jasmine";
    $DB_pass="senha";
    $DB_db="print";

5 - IonCube

        cd /var/www

        # wget http://downloads3.ioncube.com/loader_downloads/ioncube_loaders_lin_x86-64.tar.gz

        # tar xvfz ioncube_loaders_lin_x86-64.tar.gz

        cd ioncube/

        http://IP-DO-SERVIDOR/ioncube/loader-wizard.php

Faça:

        # cp /var/www/ioncube/ioncube_loader_lin_5.3.so /usr/lib/php5/20090626/

Clique no inten 3;

Save this 20-ioncube.ini file and put it in your ini files directory, /etc/php5/apache2/conf.d

Faça o download do arquivo, e copie seu conteudo.

zend_extension = /usr/lib/php5/20090626/ioncube_loader_lin_5.3.so

Cole no arquivo abaixo:

        # vim /etc/php5/apache2/conf.d/20-ioncube.ini

        # service apache2 restart

Teste o loader novamente na pagina, se tudo correu bem, continue.

        cd ..
        sudo rm -rf ioncube
        sudo rm ioncube_loaders_lin_x86-64.tar.gz

6 - Configuração após reinicialização.

    $ sudo vim /etc/restart.sh

    #!/bin/bash
    #Faz uma pausa aguardando o inicio do sistema.
    sleep 10
    #Reinicia o samba
    /etc/init.d/smbd restart && /etc/init.d/cups restart

    $ sudo chmod +x /etc/restart.sh

    $ sudo chmod 777 /etc/restart.sh

Edite o arquivo abaixo:

    $ sudo vim /etc/rc.local

cole o conteúdo abaixo em cima da linha exit 0

    /etc/restart.sh &

Reinicie o sistema.

    $ sudo shutdown -r now

7 - Instalando a impressora no servidor.

Acesse o CUPS:

http://ip_do_servidor:631

Em "Administration", selecione:

    "Show printers shared by other systems"
    "Share printers connected to this system"
    "Allow printing from the Internet"
    "Allow remote administration"
    "Allow users to cancel any job (not just their own)" 

Em "Administration", clique em "Add Printer" Faça a instalação da impressora prestando atenção em "connection".

    connection: jasmine:socket://IP-DA-IMPRESSORA

