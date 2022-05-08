# Hashicorp Vault Tutorial
#  Visão Geral 
O Hasicorp Vault permite gerenciar credenciais de todos os tipos de maneira segura, facilitando a criação e obtenção de chaves e implementando diversos engines de gerenciamento de credenciais. 

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

## :arrow_forward: Key/Value secrets engine
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

### :arrow_forward: Criando um segredo

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

### :arrow_forward: Lendo um segredo

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

Para imprimir apenas o valor de uma chave específica use o comando vault kv get informando o parâmentro -field, o caminho e o segredo. 

Exemplo:

```bash
vault kv get -field=excited secret/hello
```

## :closed_lock_with_key: Dynamic Secrets
Segredos dinâmicos são gerados quando são acessados. Segredos dinâmicos não existem até que sejam lidos, portanto, não há risco de alguém roubá-los ou outro cliente usando os mesmos segredos. Como o Vault possui mecanismos de revogação integrados, os segredos dinâmicos podem ser revogados imediatamente após o uso, minimizando o tempo de existência do segredo.

### :arrow_forward: Habilitando o mecanismo de segredos da AWS

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

A autenticação de token é habilitada automaticamente. Quando você iniciou o servidor de desenvolvimento, a saída exibiu um token raiz. A CLI do Vault lê o token raiz da variável de ambiente $VAULT_TOKEN. Esse token raiz pode executar qualquer operação no Vault porque é atribuído à política raiz. Um recurso é criar novos tokens.

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

6. Para revogar o token execute o comando vault revoke, informando o token obtido no passo anterior. 

    Exemplo: 

    ```bash 
    vault token revoke s.iyNUhq8Ov4hIAx6snw5mB2nL
    ```