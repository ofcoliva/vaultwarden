# Como Subir Seu Servidor Vaultwarden Localmente

Este guia detalha como implementei meu próprio servidor de senhas utilizando o **Vaultwarden** (uma implementação leve e em Rust da API do Bitwarden). O objetivo principal é ter controle total sobre os dados, centralizar senhas e TOTP sem custos de assinatura, e garantir que as informações não fiquem em mãos de terceiros.

## O que utilizaremos
* **Docker** (Containerização)
* **Tailscale** (Rede privada/VPN segura)
* **Vaultwarden** (Gerenciador de senhas)

## Por que Tailscale?
O Tailscale oferece uma VPN segura e gratuita para uso pessoal (até 3 usuários e 100 dispositivos na versão atual), o que atende perfeitamente à necessidade de acessar o Vaultwarden de qualquer lugar.

**Por que não Cloudflare Tunnel?**
Embora o Cloudflare Tunnel seja uma opção comum, ao aplicar políticas de acesso rigorosas (como mTLS ou filtragem de IP), o aplicativo mobile do Bitwarden costuma apresentar falhas de autenticação. Deixar o Vaultwarden exposto sem camadas extras de proteção no Cloudflare é arriscado; qualquer vulnerabilidade não corrigida (0-day) no painel de login poderia comprometer todo o cofre. O Tailscale resolve isso criando uma camada de rede privada, onde o servidor sequer "existe" na internet pública.

---

## Pré-requisitos
Certifique-se de ter o **Docker** e o **Git** instalados em sua máquina.

### 1. Preparação do Ambiente
Clone o repositório e entre no diretório:
```bash
git clone https://github.com/ofcoliva/vaultwarden.git
cd vaultwarden
```

Renomeie o arquivo de exemplo para configurar as variáveis de ambiente:
```bash
cp .env-example .env
```

### 2. Configuração do Tailscale
1. Acesse sua conta no [Tailscale](https://login.tailscale.com/).
2. Em **DNS**, habilite o **Magic DNS** e os **HTTPS Certificates**.
3. Vá em **Settings > Keys** e gere uma **Auth Key**.
4. Abra o arquivo `.env` e cole a chave em `TS_AUTHKEY=`.

### 3. Gerando o Token de Administrador
Para acessar o painel administrativo do Vaultwarden, você precisa gerar um hash de senha seguro (Argon2):

Pode ser feito criando um container temporário que irá executar o programa Vaultwarden com a função hash, ou pode optar por entrar no Vscode ou no terminal fazer o acesso remoto ao container do vaultwarden e executar o segundo comando. Fica a seu critério.

Comando 1
```bash
docker run --rm -it vaultwarden/server /vaultwarden hash
```

**Ou**

Comando 2
```bash
/vaultwarden hash
```


O terminal solicitará uma senha. Após digitar e confirmar, ele retornará uma string começando com `$argon2id$`. Copie essa string inteira e cole no seu `.env` na variável `ADMIN_TOKEN`.

Seu arquivo `.env` deve ficar parecido com isto:
```bash 
ADMIN_TOKEN='$argon2id$v=19$m=65540,t=3,p=4$xxxxxxxxxxxxxxxxxx...'
TS_AUTHKEY='tskey-auth-xxxxxxxxxxxxxxxxx'
```

---
### 4. Configuração de E-mail (SMTP)
O Vaultwarden utiliza o protocolo SMTP para enviar convites para novos usuários, alertas de segurança e notificações de mudança de senha. Como o Google (e outros provedores) bloqueiam o login direto por "aplicativos menos seguros", utilizaremos o método de **Senhas de App**.

#### Passos para smtp utilizando Gmail:
1.  Acesse sua [Conta Google](https://myaccount.google.com/).
2.  No menu lateral, clique em **Segurança**.
3.  Certifique-se de que a **Verificação em duas etapas** esteja **Ativada**.
4.  Na barra de busca da conta, digite **"Senhas de app"**.
5.  Em "Nome do app", digite algo para sua identificação (ex: `Vaultwarden Local`) e clique em **Criar**.
6.  O Google exibirá uma senha de **16 dígitos** dentro de um quadro amarelo. **Copie esta senha agora**, pois ela não será exibida novamente.

#### Preenchendo o arquivo `.env`
Agora, abra seu arquivo `.env` e insira as informações coletadas:

```bash
# Configurações de E-mail
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_SECURITY=starttls
SMTP_USERNAME=seu-email@gmail.com
SMTP_PASSWORD=aaaa bbbb cccc dddd
SMTP_FROM=seu-email@gmail.com
```

---
### Inicializando o Servidor

Execute o comando abaixo para subir os containers:
```bash
docker compose up -d
```

Após a inicialização, verifique os logs para encontrar a URL gerada pelo Tailscale:
```bash
docker logs tailscale_container
```
Você verá algo como:
```text
tailscale_container    | https://seu-servidor.palavra-palavra.ts.net/
tailscale_container    | |-- proxy http://localhost:80 
```


**IMPORTANTE**
> O certificado SSL é gerado automaticamente pelo Tailscale. Para acessar a URL, seu dispositivo (computador ou celular) **deve estar conectado à sua rede Tailscale**.

### Acessando o Painel
* **Interface do Usuário:** Acesse a URL gerada (ex: `https://seu-servidor.ts.net/`) para criar sua conta.
* **Painel de Admin:** Acesse a URL terminada em `/admin` (ex: `https://seu-servidor.ts.net/admin`).

---

## Observações Importantes
* **Configuração de SMTP:** As configurações de e-mail estão no `.env` para facilitar a configuração inicial.
* **Prioridade de Configuração:** Após o primeiro boot, o Vaultwarden passa a priorizar o arquivo `config.json` dentro do volume de dados. Mudanças posteriores devem ser feitas via **Painel Admin** ou editando o `config.json` diretamente.

---