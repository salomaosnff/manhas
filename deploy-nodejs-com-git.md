# Deploy de Aplicações NodeJS com Git Hooks

> Neste tutorial você irá aprender como automatizar o deploy de suas aplicações NodeJS utilizando o git

## Requerimentos:
- Uma VPS com acesso SSH
- Git instalado na VPS e na máquina Local
- NodeJS na VPS e na máquina Local
- Conhecimentos básicos de git e shellscript

## Criando um usuário

Primeiramente, iremos criar um usuário no servidor que será responsável por fazer o deploy.
Neste caso irei chamar de `git` mas fique à vontade para escolher o nome.
Este usuário pertencerá ao grupo sudo.

```bash
$ sudo useradd git -mG sudo
```

Para maior segurança, é recomendado definirmos uma senha para o usuário `git`.

```bash
$ sudo passwd git
```

Faça Login:

```bash
$ sudo su - git
```

Altere as permissões da pasta home do usuário `git`.

```bash
$ sudo chmod 770 -R /home/git/
```

Crie a pasta `.ssh`:

```bash
$ mkdir -p .ssh
```

## Criando uma chave SSH

Na sua máquina local, crie uma chave SSH para ter acesso ao usuário `git`, 
se você já possui, você deve pular este comando.

```bash
$ ssh-keygen -t rsa
```

Adicione sua chave pública às chaves autorizadas do usuário `git` na VPS.
Execute na sua máquina local:

```bash
$ cat ~/.ssh/id_rsa.pub | ssh root@HOST_DA_VPS "cat > /home/git/.ssh/authorized_keys"
```

> Se não tiver acesso ao usuário root de seu servidor, você pode copiar manualmente o conteúdo do arquivo `~/.ssh/id_rsa.pub` de sua máquina local, criar o arquivo `/home/git/.ssh/authorized_keys` no servidor, colar e salvar.

Tente conectar-se ao SSH com o usuário git:

```bash
ssh git@HOST
```

Se conseguir logar, você está pronto para prosseguir. 
Se não conseguir, tente refazer os passos anteriores.

## Instalando o PM2 no servidor

O PM2 é um módulo javascript que permite 
a execução de aplicações NodeJS em segundo plano. 
Com ela é possível iniciar sua aplicação após o Boot da VPS.

Para instalar, execute no seu servidor:

```bash
$ sudo npm install -g pm2
```

## Estrutura de diretórios

Neste tutorial, esta será a estrutura de pastas:
```
/home/git/
  aplicacao/
    aplicacao.git/
    producao/
```

Na pasta `aplicacao.git` ficará armazenada o repositório, 
na `produção` ficará armazenada os arquivos do projeto.

Para criar execute:

```bash
$ mkdir -p ~/aplicacao/{aplicacao.git,producao}
```

## Criando um repositório bare

Para enviar atualizações para o servidor de produção(VPS),
é necessário criarmos um repositório bare, que nada mais é 
que um repositório que fica do lado do servidor.

Leia mais em [Git no Servidor](https://git-scm.com/book/pt-br/v1/Git-no-Servidor-Configurando-o-Servidor).

Para cria-lo, entre na pasta `aplicacao` criada no passo anterior:

```bash
$ cd ~/aplicacao
```

Iniciamos um repositório bare na pasta `aplicacao.git/`.

```bash
$ git init aplicacao.git --bare
```

## Criando um Hook

Para automatizar algumas tarefas, o git inclui no seu 
repositório uma pasta `hook` que contém script que são
chamados de dependendo das ações que são feitas no repositório.
Leia mais em [Git Hooks](https://git-scm.com/book/gr/v2/Customizing-Git-Git-Hooks).

Iremos criar um script `post-receive` dentro da pasta `aplicacao.git/hooks/`.
Ele será executado toda vez que o reposítório receber atualizações.

Utilizando um editor de sua preferência, crie o arquivo `aplicacao.git/hooks/post-receive` contendo:

```shellscript
# Define a pasta de destino dos arquivos
GIT_WORK_TREE=/home/git/aplicacao/producao

echo "--> Fazendo checkout...";
GIT_WORK_TREE=$APP_DIR git checkout -f;

cd $APP_DIR;

echo "--> Instalando dependências...";

# Reinstalando módulos
rm -rf node_modules/;

# Instala somente módulos de produção
npm install --only=production

echo "--> Iniciando aplicação...";

# Inicia ou reinicia a aplicação
PORT=8000 pm2 startOrRestart app.yml

echo "--> Deploy efetuado com sucesso!"
```

Dê permissão de execução ao script:

```bash
$ sudo chmod +x aplicacao.git/hooks/post-receive
```

## Adicionando um repositório remoto ao seu projeto

Na pasta de seu projeto, execute:

```bash
$ git remote add deploy git@<SUA_VPS>:aplicacao/aplicacao.git
```

## Arquivo de inicialização do PM2

Crie um arquivo `app.yml` na pasta de seu projeto contendo:

```yml
apps:
  - script   : app.js     # Script que irá inicializar sua aplicação
    name     : app        # Nome da aplicação
    instances: 1          # Quantidade máxima de instâncias
    exec_mode: cluster    # Modo de execução (Leia mais em http://pm2.keymetrics.io/docs/usage/cluster-mode)
```

Comite as alterações executando:

```bash
$ git add -A
$ git commit -m "Deploy com Git Hooks"
```

E envie para sua VPS executando:

```bash
$ git push deploy
```

Se tudo ocorrer bem, serão instalados os módulos e o pm2 (re)iniciará sua aplicação.

## Iniciando a aplicação no boot

No seu servidor, execute:

```bash
sudo pm2 save
```

Este comando irá salvar as aplicações abertas no PM2.

Agora temos que habilitar o serviço do PM2, para isso execute:

```bash
$ pm2 startup
```

Será exibido uma saída semelhante a esta:

```
[PM2] Init System found: systemd
[PM2] You have to run this command as root. Execute the following command:
sudo env PATH=$PATH:/usr/bin /usr/lib/node_modules/pm2/bin/pm2 startup systemd -u git --hp /home/git
```

Execute o comando que tem logo no fim, e pronto!

Agora toda vez que iniciar o servidor, sua aplicação também será inicializada.

Para desativar o serviço execute: 

```bash
$ pm2 unstartup
```

Será mostrada uma saída semelhante à do `pm2 startup`, e basta executa o comando que ele pede.

## Conclusão

Neste tutorial vimos como é possível aumentar a nossa produtividade 
utilizando recursos nativos do git e o módulo PM2 para executar a aplicação 
em segundo plano. Agora você não irá perder tempo logando no SSH e iniciando a 
aplicação(e muito menos instalando os módulos) toda vez que houver uma atualização no projeto.

Isso é tudo que queria apresentar, até a próxima!
