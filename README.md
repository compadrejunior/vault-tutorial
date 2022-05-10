# Hashicorp Vault Tutorial

Esse tutorial visa simplificar os passos de instalação, configuração e utilização do Hashicorp Vault.

# Índice
- [Hashicorp Vault Tutorial](#hashicorp-vault-tutorial)
- [Índice](#índice)
- [Visão Geral](#visão-geral)
- [Development Mode](#development-mode)
  - [:hammer: Instando o Vault](#hammer-instando-o-vault)
  - [:arrow_forward: Iniciando o servidor em modo desenvolvimento](#arrow_forward-iniciando-o-servidor-em-modo-desenvolvimento)
  - [:key: Key/Value secrets engine](#key-keyvalue-secrets-engine)
    - [Criando um segredo](#criando-um-segredo)
    - [Lendo um segredo](#lendo-um-segredo)
  - [:closed_lock_with_key: Dynamic Secrets](#closed_lock_with_key-dynamic-secrets)
    - [Habilitando o mecanismo de segredos da AWS](#habilitando-o-mecanismo-de-segredos-da-aws)
  - [:ticket: Autenticação com Token](#ticket-autenticação-com-token)
- [Production mode](#production-mode)
  - [Configurando o servidor](#configurando-o-servidor)
#  Visão Geral 
O Hashicorp Vault permite gerenciar credenciais de todos os tipos de maneira segura, facilitando a criação e obtenção de chaves e implementando diversos engines de gerenciamento de credenciais. 

O Vault vem com vários componentes conectáveis chamados mecanismos de segredos e métodos de autenticação que permitem a integração com sistemas externos. O objetivo desses componentes é gerenciar e proteger seus segredos em infraestrutura dinâmica (por exemplo, credenciais de banco de dados, senhas, chaves de API).

![Componentes do Vault](./assets.png "Componentes do Vault")

# Development Mode
Os passos abaixo guiam na instalação, configuração e utilização do Vault na máquina do desenvolvedor. 

  >**Importante**: Não execute os comandos abaixo em um ambiente de produção.

## :hammer: Instando o Vault
Instale o Vault seguindo as instruções da página de [Download](https://www.vaultproject.io/downloads) da Hashicorp. 

## :arrow_forward: Iniciando o servidor em modo desenvolvimento

1. Execute o comando abaixo:

    ```bash
    vault server -dev
    ```

    > :warning: O servidor do vault é executado no foreground, ou seja, as entradas no terminal ficarão bloqueadas até que você pressione o comando CTRL+C para interromper o servidor. 

2. Abra outra janela do terminal para executar os próximos passos

3. Habilite a comunicação do cliente CLI do vault com o servidor executando o seguinte comando: 

    ```bash
    export VAULT_ADDR='http://127.0.0.1:8200'
    ```

## :key: Key/Value secrets engine
O Vault trabalha com diversos gerenciadores de credenciais como AWS, Github, KV (Key/Value secrets engine) e outros. 

O engine KV é o modo padrão do vault e vem habilitado por padrão. 

Os segredos são armazenados no engine KV dentro do caminho secrets. 

Para listar os engines habilitados execute o comando abaixo:

```bash
vault secrets list
```

Para uma lista de comandos do KV digite:

```bash
vault kv -help 
```

O resultado exibe os seguintes campos:

    - Path: caminho onde as chaves são armazenadas.
    - Type: tipo de engine
    - Access: uma chave que identifica o engine habilitado.
    - Description: a descrição do engine.

Exemplo:

```bash
Path          Type         Accessor              Description
----          ----         --------              -----------
cubbyhole/    cubbyhole    cubbyhole_e9a67f8f    per-token private secret storage
identity/     identity     identity_493cfa79     identity store
secret/       kv           kv_cbb9c9f2           key/value secret storage
sys/          system       system_aeaf18d4       system endpoints used for control, policy and debugging
```

### Criando um segredo

1. Digite o comando abaixo para ver a lista de opções do comando put. 

    ```bash
    vault kv put -help
    ```

2. Vamos criar um segredo chamado hello com vários pares chave/valor. Para isso execute o comando abaixo:

    ```bash
    vault kv put secret/hello foo=world excited=yes
    ```

    Onde: 
      - **secret**: caminho padrão de segredos do KV.
      - **hello**: caminho definido pelo usuário. Você pode determinar esse caminho para agrupar vários segredos, por exemplo, de uma mesma aplicação. 
      - **foo**: chave a ser armazenada (definida pelo usuário)
      - **world**: valor da chave a ser armazenada (definida pelo usuário)
      - **excited**: outra chave definida pelo usuário
      - **yes**: o valor da segunda chave definida pelo usuário.

### Lendo um segredo

Para ler o valor da chave armazenada anteriormente utilize o comando abaixo:

```bash
vault kv get secret/hello
```

O resultado será exibido semelhante ao exemplo abaixo:

```bash
======= Metadata =======
Key                Value
---                -----
created_time       2022-01-15T01:40:09.888293Z
custom_metadata    <nil>
deletion_time      n/a
destroyed          false
version            2

===== Data =====
Key        Value
---        -----
excited    yes
foo        world
```

O campo version indica a versão do segredo. 

Para imprimir apenas o valor de uma chave específica use o comando vault kv get informando o parâmetro -field, o caminho e o segredo. 

Exemplo:

```bash
vault kv get -field=excited secret/hello
```

## :closed_lock_with_key: Dynamic Secrets
Segredos dinâmicos são gerados quando são acessados. Segredos dinâmicos não existem até que sejam lidos, portanto, não há risco de alguém roubá-los ou outro cliente usando os mesmos segredos. Como o Vault possui mecanismos de revogação integrados, os segredos dinâmicos podem ser revogados imediatamente após o uso, minimizando o tempo de existência do segredo.

### Habilitando o mecanismo de segredos da AWS

1. Digite o comando abaixo:

    ```bash
    vault secrets enable -path=aws aws
    ```

2. Defina as credenciais de acesso da AWS

    ```bash
    export AWS_ACCESS_KEY_ID=YOUR_AWS_ACCESS_KEY_ID
    export AWS_SECRET_ACCESS_KEY=YOUR_AWS_SECRET_ACCESS_KEY
    ```

3. Configure o mecanismo de segredos da AWS.
    ```bash
    vault write aws/config/root \
        access_key=$AWS_ACCESS_KEY_ID \
        secret_key=$AWS_SECRET_ACCESS_KEY \
        region=us-east-1
    ```
4. Crie uma role com o comando abaixo.: 
    ```bash
    vault write aws/roles/my-role \
        credential_type=iam_user \
        policy_document=-<<EOF
        {
          "Version": "2012-10-17",
          "Statement": [
            {
              "Sid": "Stmt1426528957000",
              "Effect": "Allow",
              "Action": [
                "ec2:*"
              ],
              "Resource": [
                "*"
              ]
            }
          ]
        }
        EOF
    ```
5. Para gerar o segredo execute o comando abaixo:

    ```bash
    vault read aws/creds/my-role
    ```

    >**Importante**: copie o valor do campo lease_id. Esse valor será usado no comando abaixo para revogar o segredo. 

6. Para revogar o segredo execute o comando abaixo, substituindo o campo <lease_id> pelo valor copiado no passo anterior:

    ```bash
    vault lease revoke aws/creds/my-role/<lease_id>
    ```

7. Para uma lista completa de comandos do AWS Secret Engine execute o comando abaixo:

    ```bash
    vault path-help aws
    ```

## :ticket: Autenticação com Token

A autenticação de token é habilitada automaticamente. Quando você iniciou o 
servidor de desenvolvimento, a saída exibiu um token raiz. A CLI do Vault lê o 
token raiz da variável de ambiente $VAULT_TOKEN. Esse token raiz pode executar 
qualquer operação no Vault porque é atribuído à política raiz. Um recurso é 
criar novos tokens.

1. Para criar um token digite:

    ```bash
    vault token create
    ```

2. O resultado exibido deverá ser semelhante ao exemplo abaixo:


    ```bash
    Key                  Value
    ---                  -----
    token                s.iyNUhq8Ov4hIAx6snw5mB2nL
    token_accessor       maMfHsZfwLB6fi18Zenj3qh6
    token_duration       ∞
    token_renewable      false
    token_policies       ["root"]
    identity_policies    []
    policies             ["root"]
    ```

3. Use o token para efetuar o login no Vault.

    ```bash
    vault login
    ```

4. Informe o token no prompt abaixo>

    ```bash
    Token (will be hidden):
    ```

5. Se bem sucedido o resultado será semelhante ao exibido abaixo:

    ```
    Success! You are now authenticated. The token information displayed below
    is already stored in the token helper. You do NOT need to run "vault login"
    again. Future Vault requests will automatically use this token.

    Key                  Value
    ---                  -----
    token                s.iyNUhq8Ov4hIAx6snw5mB2nL
    token_accessor       maMfHsZfwLB6fi18Zenj3qh6
    token_duration       ∞
    token_renewable      false
    token_policies       ["root"]
    identity_policies    []
    policies             ["root"]
    ```

6. Para revogar o token execute o comando vault revoke, informando o token 
   obtido no passo anterior. 

    Exemplo: 

    ```bash 
    vault token revoke s.iyNUhq8Ov4hIAx6snw5mB2nL
    ```

# Production mode
Essa etapa irá guia-lo no deploy do Hashicorp Vault em ambiente de produção. 

  > Se o servidor do Vault estiver sendo executado no modo de desenvolvimento, é necessário parar o servidor pressionando CTRL+C no terminal. Também remova a variável de ambiente VAULT_TOKEN através do comando ```unset VAULT_TOKEN```

## Configurando o servidor

1. Crie o arquivo config.hcl com o seguinte conteúdo.
    ```bash
    storage "raft" {
      path    = "./vault/data"
      node_id = "node1"
    }

    listener "tcp" {
      address     = "127.0.0.1:8200"
      tls_disable = "true"
    }

    api_addr = "http://127.0.0.1:8200"
    cluster_addr = "https://127.0.0.1:8201"
    ui = true
    ```
2. Crie o diretório de storage.
    ```bash
    mkdir -p ./vault/data
    ```
3. Inicie o servidor definindo o parâmetro -config para apontar para o caminho adequado onde você salvou a configuração acima.
    ```bash
    vault server -config=config.hcl
    ```
4. Exporte a variável VAULT_ADDR
    ```bash
    export VAULT_ADDR='http://127.0.0.1:8200'
    ```
5. Inicialize o operador do Vault

    ```bash
    vault operator init
    ```
6. Salve as chaves informadas no console.
7. Execute o comando abaixo informando a primeira chave. 
    ```bash
    vault operator unseal
    ```
8. Continue com a deslacração do operador do vault para concluir a abertura do Vault. Para deslacrar o cofre você deve usar três chaves diferentes de unseal, a mesma chave repetida não funcionará. 
9. O resultado será parecido com a saída abaixo:

    ```bash
    Key                     Value
    ---                     -----
    Seal Type               shamir
    Initialized             true
    Sealed                  false
    Total Shares            5
    Threshold               3
    Version                 1.7.0
    Storage Type            raft
    Cluster Name            vault-cluster-0ba62cae
    Cluster ID              7d49e5fd-a1a4-c1d1-55e2-7962e43006a1
    HA Enabled              true
    HA Cluster              n/a
    HA Mode                 standby
    Active Node Address     &lt;none&gt;
    Raft Committed Index    24
    Raft Applied Index      24
    ```
10. Quando o valor de Sealed muda para false, o Vault é deslacrado.
11. Por fim, autentique como o token raiz inicial (foi incluído na saída com as chaves de unseal).
    ```bash
    vault login
    ```
12. Insira o valor do token quando solicitado.
13. O resultado será parecido com a saída abaixo:

    ```bash
    Success! You are now authenticated. The token information displayed below
    is already stored in the token helper. You do NOT need to run "vault login"
    again. Future Vault requests will automatically use this token.

    Key                  Value
    ---                  -----
    token                s.spAZOi7SlpdFTNed50sYYCIU
    token_accessor       OevFmMXjbmOCQ8rSubY84vVp
    token_duration       ∞
    token_renewable      false
    token_policies       ["root"]
    identity_policies    []
    policies             ["root"]
    ```
