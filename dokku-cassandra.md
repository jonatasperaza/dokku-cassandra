# Documentação Técnica: Plugin Dokku Cassandra (`jonatasperaza/dokku-cassandra`)

Esta documentação analisa o repositório [jonatasperaza/dokku-cassandra](https://github.com/jonatasperaza/dokku-cassandra.git), detalhando sua arquitetura, funcionamento, comandos disponíveis e o guia prático para integrá-lo ao **ThingsBoard** no ambiente Dokku em modo híbrido (PostgreSQL + Cassandra).

---

## 1. Visão Geral e Contexto do Repositório

* **Repositório:** [https://github.com/jonatasperaza/dokku-cassandra.git](https://github.com/jonatasperaza/dokku-cassandra.git)
* **Autor:** Jonatas Peraza
* **Imagem Base Utilizada:** Imagem oficial do Docker Hub: **`cassandra:5.0.9`**
* **Objetivo:** O Dokku oficial fornece plugins de banco de dados gerenciados para PostgreSQL, MySQL, Redis, MongoDB e MariaDB, porém **não possui um plugin oficial para Apache Cassandra**. O repositório do Jonatas Peraza foi desenvolvido para preencher essa lacuna, permitindo provisionar, gerenciar, isolar e vincular instâncias do Apache Cassandra em servidores Dokku com a mesma experiência de CLI dos plugins oficiais.

---

## 2. O Que Foi Desenvolvido no Repositório (Análise Técnica)

O repositório segue rigorosamente as especificações de plugins para Dokku v0.19.x+:

### Estrutura de Arquivos

```
dokku-cassandra/
├── plugin.toml             # Metadados e versão da API do Dokku
├── config                  # Variáveis globais, portas e diretórios de dados
├── common-functions        # Biblioteca base de funções do ecossistema Dokku
├── functions               # Lógica de lifecycle do container Cassandra (create, connect, etc.)
├── commands                # Manifesto de comandos e documentação do CLI
├── post-app-clone-setup    # Hook disparado ao clonar apps no Dokku
├── post-app-rename-setup   # Hook disparado ao renomear apps no Dokku
├── pre-delete              # Hook de limpeza antes de excluir apps
├── pre-restore             # Hook executado na restauração do host Dokku
├── pre-start               # Hook executado antes do container subir
├── service-list            # Hook para listagem de serviços
├── docs/                   # Documentação adicional e notas de arquitetura
└── subcommands/            # 22 subcomandos executáveis via CLI do Dokku
```

### Principais Componentes Implementados

1. **Gestão de Rede e Portas (`config`):**
   * Configura o rastreamento da porta **9042** (porta do protocolo binário nativo CQL).
   * As portas `7000/7001` (comunicação inter-nó de cluster) e `7199` (JMX) não são expostas, permanecendo estritamente dentro da rede interna do Docker.
   * Alias padrão injetado no Dokku: `CASSANDRA`.

2. **Gerenciamento de Volumes e Persistência (`functions`):**
   * Os dados do Cassandra são armazenados persistentemente no host em:
     `/var/lib/dokku/services/cassandra/<nome-do-servico>/data`
   * Mapeado como volume para o caminho interno padrão do Cassandra: `/var/lib/cassandra`.

3. **Terminal Interativo CQL (`subcommands/connect` e `subcommands/admin-console`):**
   * Executa diretamente `docker exec -it <container> cqlsh`, dando acesso imediato ao shell CQL para criar keyspaces, tabelas e consultar dados.

4. **Vinculação com Aplicações (`subcommands/link` e `subcommands/unlink`):**
   * Conecta a rede Docker do app ThingsBoard à rede do container Cassandra.
   * Injeta automaticamente a variável de ambiente `CASSANDRA_URL` na aplicação Dokku vinculada.

---

## 3. Particularidades Técnicas e Decisões de Design do Plugin

Ao analisar o código do plugin, três detalhes técnicos fundamentais devem ser observados para a integração com o ThingsBoard:

> [!NOTE]
> ### 1. Sem Autenticação por Padrão
> A imagem oficial do Apache Cassandra não possui variáveis de ambiente padrão para habilitar senhas (diferente de `POSTGRES_PASSWORD` ou `MYSQL_PASSWORD`). Para evitar manter arquivos `cassandra.yaml` customizados e complexos, o plugin optou por rodar sem `PasswordAuthenticator`. A segurança e o isolamento são garantidos pelo isolamento de rede do Docker (o container só é acessível por apps linkados na mesma rede privada).

> [!WARNING]
> ### 2. O Keyspace NÃO é Criado Automaticamente
> Diferente do PostgreSQL (onde `dokku postgres:create` cria o banco automaticamente), o Cassandra não cria keyspaces na inicialização. É **obrigatório** acessar o `cqlsh` via `dokku cassandra:connect` e rodar o script de criação do keyspace `thingsboard` antes de iniciar a aplicação.

> [!CRITICAL]
> ### 3. Incompatibilidade do Formato de URL gerado pelo Dokku
> Ao executar `dokku cassandra:link <servico> <app>`, o Dokku gera uma URL no formato padrão:
> `CASSANDRA_URL="cassandra://dokku-cassandra-<servico>:9042/<servico>"`
> 
> O driver Java do ThingsBoard (**DataStax Java Driver v4**) utiliza a classe `CassandraDriverOptions`, que divide os pontos de contato por vírgula e espera estritamente o formato **`host:port`** (ex.: `dokku-cassandra-<servico>:9042`). Se a URL contiver o esquema `cassandra://`, o driver falhará ao resolver o hostname DNS.
> 
> **Solução:** Sobrescrever a variável no Dokku manualmente após o comando `link`.

---

## 4. Guia Passo a Passo: Integrando o ThingsBoard com o `dokku-cassandra`

### Passo 1: Instalar o Plugin no Servidor Dokku
```bash
sudo dokku plugin:install https://github.com/jonatasperaza/dokku-cassandra.git --name cassandra
```

### Passo 2: Criar o Serviço Cassandra
```bash
dokku cassandra:create tb-cassandra
```

### Passo 3: Criar o Keyspace e as Tabelas do ThingsBoard
Abra a console CQL interativa do serviço recém-criado:
```bash
dokku cassandra:connect tb-cassandra
```

Cole os comandos CQL oficiais do ThingsBoard:
```cql
-- 1. Criar Keyspace
CREATE KEYSPACE IF NOT EXISTS thingsboard
WITH replication = {
    'class': 'SimpleStrategy',
    'replication_factor': 1
};

-- 2. Criar Tabela de Telemetria Histórica
CREATE TABLE IF NOT EXISTS thingsboard.ts_kv_cf (
    entity_type text,
    entity_id timeuuid,
    key text,
    partition bigint,
    ts bigint,
    bool_v boolean,
    str_v text,
    long_v bigint,
    dbl_v double,
    json_v text,
    PRIMARY KEY (( entity_type, entity_id, key, partition ), ts)
);

-- 3. Criar Tabela de Partições
CREATE TABLE IF NOT EXISTS thingsboard.ts_kv_partitions_cf (
    entity_type text,
    entity_id timeuuid,
    key text,
    partition bigint,
    PRIMARY KEY (( entity_type, entity_id, key ), partition)
) WITH CLUSTERING ORDER BY ( partition ASC )
  AND compaction = { 'class' :  'LeveledCompactionStrategy'  };

-- 4. Criar Tabela de Último Valor (para suporte a latest no Cassandra se desejado)
CREATE TABLE IF NOT EXISTS thingsboard.ts_kv_latest_cf (
    entity_type text,
    entity_id timeuuid,
    key text,
    ts bigint,
    bool_v boolean,
    str_v text,
    long_v bigint,
    dbl_v double,
    json_v text,
    PRIMARY KEY (( entity_type, entity_id ), key)
) WITH compaction = { 'class' :  'LeveledCompactionStrategy'  };

exit
```

### Passo 4: Vincular o Serviço ao App ThingsBoard
Substitua `<nome-do-app-tb>` pelo nome da aplicação ThingsBoard no seu Dokku:
```bash
dokku cassandra:link tb-cassandra <nome-do-app-tb>
```

### Passo 5: Configurar as Variáveis de Ambiente Corretas no Dokku
Defina as variáveis para ativar o modo híbrido e corrigir o formato da URL:
```bash
dokku config:set <nome-do-app-tb> \
  DATABASE_TS_TYPE="cassandra" \
  CASSANDRA_URL="dokku-cassandra-tb-cassandra:9042" \
  CASSANDRA_KEYSPACE_NAME="thingsboard" \
  CASSANDRA_CLUSTER_NAME="Thingsboard Cluster" \
  DATABASE_TS_LATEST_TYPE="sql"
```

> **Por que `DATABASE_TS_LATEST_TYPE=sql`?**
> No modo híbrido oficial do ThingsBoard, o último valor de telemetria é mantido no PostgreSQL. Isso permite que a tela de listagem de dispositivos faça `JOIN` rápido na view `device_info_view`, exibindo o status Online/Offline e o último valor dos sensores de forma instantânea.

---

## 5. Comportamento dos Dados e Migração da Telemetria Antiga

### O que acontece com os dados que já estavam no PostgreSQL?
* **Entidades de negócio (Dispositivos, Clientes, Usuários, Dashboards, Regras):** Continuam 100% no PostgreSQL, totalmente funcionais e inalterados.
* **Última Telemetria Recebida (Latest):** Continua sendo lida e atualizada no PostgreSQL (`DATABASE_TS_LATEST_TYPE=sql`).
* **Histórico Passado de Telemetria (`ts_kv` do PostgreSQL):** O ThingsBoard **ignora** os registros antigos de telemetria que estavam no PostgreSQL. O ThingsBoard não realiza consultas federadas; todas as consultas de gráficos históricos passarão a buscar exclusivamente no Cassandra.

### Como migrar o histórico antigo do PostgreSQL para o Cassandra?
Para que os dados históricos antigos continuem visíveis nos gráficos do ThingsBoard após a ativação do Cassandra, utilize o script automatizado criado no projeto:
[`docker/migrate-postgres-to-cassandra.sh`](../docker/migrate-postgres-to-cassandra.sh)

**Executando a migração:**
```bash
PG_HOST="<ip-ou-host-postgres>" \
PG_PORT="5432" \
PG_USER="postgres" \
PG_DB="thingsboard" \
CASSANDRA_CONTAINER="dokku-cassandra-tb-cassandra" \
bash docker/migrate-postgres-to-cassandra.sh
```
O script fará o join com a tabela `key_dictionary`, calculará as partições de data (formato `MONTHS`) e importará tudo para as tabelas `ts_kv_cf` e `ts_kv_partitions_cf` do Cassandra via `cqlsh COPY`.

---

## 6. Referência Completa de Comandos do Plugin `dokku-cassandra`

| Comando | Descrição |
| :--- | :--- |
| `dokku cassandra:create <service>` | Cria e inicia o container com volume persistente (Cassandra 5.0.9). |
| `dokku cassandra:connect <service>` | Conecta ao console CQL (`cqlsh`) do container interativamente. |
| `dokku cassandra:admin-console <service>` | Alias de `connect`. |
| `dokku cassandra:link <service> <app>` | Conecta a rede do app ao container e injeta a variável `CASSANDRA_URL`. |
| `dokku cassandra:unlink <service> <app>` | Remove o container da rede do app e limpa as variáveis. |
| `dokku cassandra:promote <service> <app>` | Promove o serviço como `CASSANDRA_URL` principal da aplicação. |
| `dokku cassandra:info <service>` | Exibe informações de IP, portas, status e versão do serviço. |
| `dokku cassandra:logs <service> [-t]` | Exibe os logs do container do Cassandra. |
| `dokku cassandra:restart <service>` | Reinicia o container graciosamente. |
| `dokku cassandra:stop <service>` | Para o container do serviço. |
| `dokku cassandra:start <service>` | Inicia um container de serviço previamente parado. |
| `dokku cassandra:expose <service> <port>` | Expõe a porta do Cassandra para fora do host Dokku. |
| `dokku cassandra:unexpose <service>` | Remove o redirecionamento de porta pública. |
| `dokku cassandra:export <service>` | Gera um snapshot/dump do diretório de dados do serviço. |
| `dokku cassandra:import <service>` | Restaura um snapshot para o diretório de dados. |
| `dokku cassandra:destroy <service> [-f]` | Exclui definitivamente o serviço e seus volumes de dados. |
