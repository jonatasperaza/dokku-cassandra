# dokku-cassandra

Plugin de serviço Cassandra para [Dokku](https://dokku.com), no mesmo estilo dos plugins de datastore oficiais (postgres, mysql, redis): cada serviço roda como um container dedicado, com dados persistidos em volume, e pode ser linkado a apps via variável de ambiente `CASSANDRA_URL`.

## Requisitos

- Dokku 0.19+
- Docker

## Instalação

```bash
# no servidor Dokku
sudo dokku plugin:install https://github.com/<seu-usuario>/dokku-cassandra.git --name cassandra
```

Ou, sem publicar num repositório git, copiando os arquivos direto para o servidor:

```bash
scp -r dokku-cassandra root@servidor:/var/lib/dokku/plugins/available/cassandra
ssh root@servidor "dokku plugin:enable cassandra"
```

## Uso básico

```bash
# criar um serviço chamado "db1"
dokku cassandra:create db1

# ver informações de conexão
dokku cassandra:info db1

# linkar ao app (seta CASSANDRA_URL no app)
dokku cassandra:link db1 meuapp

# entrar no cqlsh
dokku cassandra:connect db1

# parar / iniciar
dokku cassandra:stop db1
dokku cassandra:start db1

# destruir (pede confirmação, a não ser que use -f)
dokku cassandra:destroy db1
```

## Comandos disponíveis

```
cassandra:admin-console <service>                    # abre um cqlsh contra o serviço (alias de connect)
cassandra:app-links <app>                             # lista os serviços cassandra linkados a um app
cassandra:clone <service> <new-service>                # cria <new-service> e copia os dados de <service>
cassandra:connect <service>                            # conecta via cqlsh
cassandra:create <service> [--create-flags...]         # cria um serviço
cassandra:destroy <service> [-f|--force]                # apaga serviço/dados/container
cassandra:enter <service> [cmd...]                      # abre um shell (ou roda um comando) no container
cassandra:exists <service>                              # verifica se o serviço existe
cassandra:export <service>                              # exporta um snapshot do diretório de dados (stdout)
cassandra:expose <service> <ports...>                   # expõe o serviço na interface pública
cassandra:import <service>                              # importa um snapshot para o diretório de dados (stdin)
cassandra:info <service> [--flag]                       # informações do serviço
cassandra:link <service> <app>                          # linka o serviço ao app
cassandra:linked <service> <app>                        # verifica se está linkado
cassandra:links <service>                                # lista apps linkados ao serviço
cassandra:list                                           # lista todos os serviços
cassandra:logs <service> [-t|--tail] [n]                 # logs do container
cassandra:pause <service>                                # pausa o container (mantém o container)
cassandra:promote <service> <app>                        # promove o serviço como CASSANDRA_URL principal
cassandra:restart <service>                              # reinicia o container
cassandra:set <service> <key> <value>                    # seta propriedades (initial-network, post-create-network, post-start-network)
cassandra:start <service>                                # inicia um serviço parado
cassandra:stop <service>                                 # para e remove o container (dados ficam no volume)
cassandra:unexpose <service>                             # remove exposição pública
cassandra:unlink <service> <app>                         # desfaz o link com o app
cassandra:upgrade <service> [--upgrade-flags...]         # troca a imagem/versão do serviço
```

## Flags de `create` / `upgrade` / `clone`

- `-i|--image IMAGE`: imagem docker a usar (padrão: `cassandra`)
- `-I|--image-version VERSION`: versão/tag da imagem (padrão: `5.0`)
- `-m|--memory MEMORY`: limite de memória do container, em MB
- `-c|--config-options "..."`: flags JVM extras (repassadas via `JVM_EXTRA_OPTS`)
- `-C|--custom-env "A=1;B=2"`: variáveis de ambiente extras, separadas por `;`
- `-d|--cluster-name NAME`: nome do cluster Cassandra (padrão: nome do serviço, sanitizado)
- `-N|--initial-network NETWORK`: rede docker inicial
- `-P|--post-create-network NETWORKS`: redes a conectar após criar o container
- `-S|--post-start-network NETWORKS`: redes a conectar após iniciar o container

Exemplo:

```bash
dokku cassandra:create db1 --memory 1024 --image-version 5.0
```

## Como funciona / limitações

- Usa a imagem oficial `cassandra` do Docker Hub diretamente (não builda imagem própria).
- Single-node por serviço: cada `cassandra:create` sobe um nó Cassandra isolado, próprio para desenvolvimento, staging ou cargas pequenas — não forma um cluster multi-nó automaticamente.
- **Sem autenticação por padrão**: a imagem oficial do Cassandra não autentica por padrão, e este plugin não configura `PasswordAuthenticator`. O isolamento é feito pela rede Docker (o serviço só é alcançável pelos containers linkados, a não ser que você use `cassandra:expose`). Se precisar de autenticação, configure manualmente dentro do container ou contribua um PR :)
- A URL de conexão gerada é do tipo `cassandra://dokku-cassandra-<service>:9042/<keyspace>` — o "keyspace" no final é só um valor sanitizado do nome do serviço; crie o keyspace de fato via `cassandra:connect` antes de usá-lo.
- `cassandra:export` / `cassandra:import` fazem um `nodetool flush` seguido de tar do diretório de dados — funcional para single-node, mas não é um backup consistente para clusters multi-nó.
- A inicialização de um container novo pode levar 1-3 minutos (JVM + bootstrap do Cassandra); `cassandra:create` e `cassandra:start` aguardam até `PLUGIN_STARTUP_TIMEOUT` (padrão 180 tentativas de 2s = ~6 min) o `cqlsh` responder antes de desistir.
- Triggers de app (`pre-delete`, `post-app-clone-setup`, `post-app-rename-setup`) não foram implementados nesta primeira versão — destruir/renomear apps não desfaz links automaticamente, então rode `cassandra:unlink` antes de apagar um app com serviço linkado.

## Estrutura do plugin

Segue o padrão clássico de plugins de datastore do Dokku (usado por postgres/mysql/mongo antes das reescritas em Go):

```
plugin.toml        # metadados do plugin
config              # variáveis de configuração (portas, imagem padrão, paths)
commands            # dispatcher de "dokku cassandra:<subcomando>"
functions           # lógica específica do Cassandra (create/start/export/url/...)
common-functions    # lógica genérica de serviço (link/unlink/info/list/...)
subcommands/        # um arquivo executável por subcomando
```
