# Configuração de SSH e UFW no Ubuntu 24.04 usando Docker

## 1. Objetivo

Este documento descreve como criar um container Ubuntu 24.04, instalar e
configurar o servidor SSH (`OpenSSH Server`) e o firewall UFW
(`Uncomplicated Firewall`), além de testar o acesso remoto e validar as
regras de firewall.

> **Importante sobre Docker:** containers compartilham o kernel do host
> e normalmente não executam um sistema de inicialização completo. O UFW
> gerencia regras de firewall no ambiente Linux, mas em um container
> Docker suas regras podem não funcionar como funcionariam em uma
> máquina virtual ou física, devido às capacidades, namespaces e regras
> de rede administradas pelo Docker. Este exercício serve para
> demonstrar instalação e configuração; para validar o firewall de forma
> realista, prefira uma VM Ubuntu 24.04. Não altere o firewall do host
> sem compreender o impacto.

## 2. Pré-requisitos

-   Docker instalado e em execução no computador.
-   Terminal com acesso ao comando `docker`.
-   Cliente SSH instalado no computador host.
-   Conexão com a internet para baixar a imagem e pacotes.

Verifique o Docker:

``` bash
docker --version
docker info
```

Se `docker info` apresentar erro de conexão, inicie o serviço Docker no
host antes de continuar.

## 3. Baixar a imagem Ubuntu 24.04

``` bash
docker pull ubuntu:24.04
```

Confirme que a imagem foi baixada:

``` bash
docker image ls ubuntu
```

## 4. Criar e iniciar o container

Crie o container com a porta TCP 2222 do host encaminhada para a porta
22 do container. O SSH ainda não estará disponível até ser instalado e
iniciado.

``` bash
docker run -dit \
  --name ubuntu-ssh-ufw \
  -p 2222:22 \
  ubuntu:24.04 \
  bash
```

Confira o estado do container:

``` bash
docker ps
```

Abra um shell dentro dele:

``` bash
docker exec -it ubuntu-ssh-ufw bash
```

Os próximos comandos desta seção devem ser executados **dentro do
container**, salvo indicação contrária.

## 5. Atualizar os pacotes

``` bash
apt update
apt upgrade -y
```

Instale utilitários úteis para diagnóstico:

``` bash
apt install -y iproute2 iputils-ping nano sudo
```

## 6. Instalar o servidor SSH e o UFW

Como a imagem base é mínima, instale explicitamente os pacotes:

``` bash
apt install -y openssh-server ufw
```

Verifique as versões e a instalação:

``` bash
ssh -V
ufw version
dpkg -l openssh-server ufw
```

Crie o diretório necessário ao serviço SSH, caso não exista:

``` bash
mkdir -p /run/sshd
```

## 7. Configurar o SSH

### 7.1 Criar um usuário para acesso remoto

Evite usar o login direto como `root`. Crie um usuário próprio e defina
uma senha:

``` bash
adduser administrador
```

Siga as perguntas do comando e defina uma senha forte. Se for necessário
conceder privilégios administrativos no laboratório, adicione o usuário
ao grupo `sudo`:

``` bash
usermod -aG sudo administrador
```

> Em um ambiente real, conceda privilégios administrativos apenas quando
> forem necessários. A senha digitada durante `adduser` não aparece na
> tela.

### 7.2 Ajustar a configuração do servidor

Faça uma cópia de segurança do arquivo original:

``` bash
cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

Abra o arquivo:

``` bash
nano /etc/ssh/sshd_config
```

Garanta que as diretivas abaixo estejam definidas, sem o caractere `#`
no início. Se já existirem, edite-as em vez de duplicá-las:

``` text
Port 22
PermitRootLogin no
PubkeyAuthentication yes
PasswordAuthentication yes
```

`PasswordAuthentication yes` é útil para o laboratório. Em servidores
expostos à internet, prefira autenticação por chave pública e desative
senhas depois de confirmar que o acesso por chave funciona. Não exponha
o serviço publicamente com credenciais fracas.

Valide a sintaxe:

``` bash
sshd -t
```

Se o comando não exibir mensagens, a validação foi concluída sem erros.

### 7.3 Iniciar o servidor SSH

Containers Ubuntu podem não executar `systemd`; por isso,
`systemctl start ssh` pode não funcionar. Inicie o daemon diretamente:

``` bash
mkdir -p /run/sshd
/usr/sbin/sshd
```

Confira se está escutando na porta 22:

``` bash
ss -lntp | grep ':22'
```

Confira os processos:

``` bash
pgrep -a sshd
```

Se precisar reiniciar o daemon após alterar a configuração, em
laboratório pode encerrá-lo e iniciá-lo novamente:

``` bash
pkill sshd
/usr/sbin/sshd
```

Execute isso somente depois de validar a configuração com `sshd -t`.

## 8. Configurar o UFW

### 8.1 Conferir o estado inicial

``` bash
ufw status verbose
```

### 8.2 Definir políticas padrão

Permita conexões de saída e bloqueie conexões de entrada que não tenham
regra explícita:

``` bash
ufw default deny incoming
ufw default allow outgoing
```

### 8.3 Permitir SSH antes de habilitar o firewall

Como o SSH usa a porta 22 dentro do container, adicione a regra antes de
ativar o UFW:

``` bash
ufw allow 22/tcp
```

Confira as regras configuradas:

``` bash
ufw show added
```

Ative o UFW:

``` bash
ufw enable
```

Confirme o estado:

``` bash
ufw status numbered
ufw status verbose
```

A saída deve indicar que o UFW está ativo e apresentar uma regra para
`22/tcp`. Se o UFW alertar que pode interromper conexões SSH, confirme
que a regra SSH foi adicionada antes de prosseguir.

> **Limitação no Docker:** o encaminhamento `-p 2222:22` é administrado
> pelo Docker no host. O UFW dentro do container não controla
> necessariamente esse encaminhamento. O resultado efetivo depende da
> configuração de rede e das regras do host; portanto, `ufw status` não
> comprova, por si só, que a porta publicada esteja protegida.

## 9. Testar o acesso SSH

Saia do container:

``` bash
exit
```

No terminal do **host**, verifique se o container continua em execução:

``` bash
docker ps
```

Tente conectar-se ao SSH publicado na porta 2222:

``` bash
ssh -p 2222 administrador@localhost
```

Na primeira conexão, confira a impressão digital apresentada e aceite-a
somente se estiver de acordo com o ambiente esperado. Digite a senha
definida ao criar o usuário.

Após conectar, confirme a identidade e o sistema:

``` bash
whoami
hostname
```

Para sair da sessão SSH:

``` bash
exit
```

Se o cliente SSH não estiver instalado no host, instale um cliente
OpenSSH apropriado ao sistema operacional ou faça o teste a partir de
outra máquina que tenha o comando `ssh`.

## 10. Testes e diagnóstico

### Verificar o estado do container

No host:

``` bash
docker ps
docker logs ubuntu-ssh-ufw
```

### Verificar SSH e UFW dentro do container

``` bash
docker exec -it ubuntu-ssh-ufw bash
ss -lntp
pgrep -a sshd
ufw status verbose
```

### Problemas comuns

-   **`Connection refused`:** confirme que o container está em execução,
    que `sshd` foi iniciado e que a porta 2222 está publicada.
-   **`Connection timed out`:** confira as regras de rede e firewall do
    host, além do mapeamento de portas do Docker.
-   **`Permission denied`:** confira o nome do usuário, a senha e as
    diretivas de autenticação no SSH.
-   **`sshd -t` apresenta erro:** revise `/etc/ssh/sshd_config` e
    corrija a linha indicada antes de reiniciar o serviço.
-   **`systemctl` não funciona:** isso é esperado em muitos containers
    mínimos; use o comando direto `/usr/sbin/sshd` neste laboratório.
-   **UFW não aparece como esperado ou não controla a porta publicada:**
    o firewall dentro do container tem limitações. Faça o teste de
    firewall em uma VM Ubuntu ou configure regras no host com cautela.

## 11. Comandos úteis de referência

  ---------------------------------------------------------------------------
  Finalidade                          Comando
  ----------------------------------- ---------------------------------------
  Entrar no container                 `docker exec -it ubuntu-ssh-ufw bash`

  Ver logs do container               `docker logs ubuntu-ssh-ufw`

  Validar configuração SSH            `sshd -t`

  Iniciar o SSH                       `/usr/sbin/sshd`

  Ver portas em escuta                `ss -lntp`

  Ver status do UFW                   `ufw status verbose`

  Listar regras do UFW                `ufw status numbered`

  Permitir SSH                        `ufw allow 22/tcp`

  Bloquear conexões de entrada por    `ufw default deny incoming`
  padrão                              

  Permitir conexões de saída por      `ufw default allow outgoing`
  padrão                              
  ---------------------------------------------------------------------------

## 12. Limpeza do ambiente de laboratório

Quando terminar, execute no host:

``` bash
docker stop ubuntu-ssh-ufw
docker rm ubuntu-ssh-ufw
```

Opcionalmente, remova a imagem se não for mais necessária:

``` bash
docker rmi ubuntu:24.04
```

A remoção da imagem só será possível se nenhum outro container depender
dela.

## 13. Checklist final

-   [ ] Imagem `ubuntu:24.04` baixada.
-   [ ] Container criado com a porta `2222:22`.
-   [ ] Pacotes `openssh-server` e `ufw` instalados.
-   [ ] Usuário não-root criado.
-   [ ] Configuração SSH validada com `sshd -t`.
-   [ ] SSH iniciado e escutando na porta 22 do container.
-   [ ] Regra `22/tcp` adicionada antes de habilitar o UFW.
-   [ ] Estado e regras do UFW conferidos.
-   [ ] Conexão SSH testada a partir do host.
-   [ ] Limitações de firewall no Docker compreendidas.

## 14. Referências oficiais

-   Ubuntu Server --- OpenSSH:
    https://ubuntu.com/server/docs/openssh-server
-   Ubuntu Server --- Firewall:
    https://ubuntu.com/server/docs/security-firewall
-   Docker --- `docker run`:
    https://docs.docker.com/reference/cli/docker/container/run/
-   Docker --- Port publishing:
    https://docs.docker.com/engine/network/port-publishing/
-   OpenSSH --- `sshd_config`: https://man.openbsd.org/sshd_config
