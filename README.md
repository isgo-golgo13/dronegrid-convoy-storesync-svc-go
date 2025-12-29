# Cloud-Native Postgres Lottery Tx Application 

This Git repository provides provisioning and configuration for CloudNativePG Postgres in HA for Kubernetes with the add-on Rook CephFS Operator Filesystem Orchestrator. This Git repository will include HA failover workflows to force-fail the CNPG database.

## Prerequsites

- CNPG (Postgres) Kubernetes Operator installation
- Sealed Secrets Controller for Kubernetes (Auto-Secret Rotation Not-Required ONLY)



## Provisioning Sealed Secrets Controller

To add and install the Helm Chart.
```shell
helm repo add sealed-secrets https://bitnami-labs.github.io/sealed-secrets
helm repo update
```

To install using a `dry-run`.
```shell
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace sealed-secrets \
  --set fullnameOverride=sealed-secrets-controller --create-namespace --dry-run
```

To install without the dry-run.
```shell
helm install sealed-secrets sealed-secrets/sealed-secrets \
  --namespace sealed-secrets \
  --set fullnameOverride=sealed-secrets-controller --create-namespace
```


## Provisioning Cloud-Native Postgres

To add and install the Helm Chart.

```shell
helm repo add cloudnative-pg https://cloudnative-pg.io/charts/
helm repo update
```

To install using a `dry-run`
```shell
helm install enginevector-cloudnativepg cloudnative-pg/cloudnative-pg \
--namespace cnpg-database -f cloudnative-pg-chart/cnpg-override-values.yaml \
--version 0.22.0 --create-namespace --dry-run
```

To install without the dry-run.

```shell
helm install enginevector-cloudnativepg cloudnative-pg/cloudnative-pg \
--namespace cnpg-database -f cnpg-cluster-chart/cnpg-override-values.yaml \
--version 0.22.0 --create-namespace
``` 

NOTES on deletion for re-install.

Delete the following.

Delete the existing `MutatingWebhookConfiguration` 
```shell
kubectl delete mutatingwebhookconfiguration cnpg-mutating-webhook-configuration
```
and 

Delete the existing `ValidatingWebhookConfiguration` 
```shell
kubectl delete mutatingwebhookconfiguration cnpg-mutating-webhook-configuration
```


## Install CloudNativePG HA Cluster and ConfigMap SQL Schema

To do a pre-flight pre-render using `helm template` do the following.

```shell
helm template enginevector-cnpg-cluster . \
--namespace enginevector-cnpg-cluster \
--version 1.0.0 -f cnpg-cluster-chart/cnpg-cluster-override-values.yaml
```

To install the EQL CloudNative HA Cluster and EQL SQL Schema ConfigMap
To install without the dry-run.

```shell
helm install enginevector-cnpg-cluster . \
--namespace enginevector-cnpg-cluster \
--version 1.0.0 -f cnpg-cluster-chart/cnpg-cluster-override-values.yaml --create-namespace
``` 

To apply changes to the Helm Chart and do upgrade lifecycles.

```shell
helm upgrade enginevector-cnpg-cluster . \
--namespace enginevector-cnpg-cluster \
--version 1.0.0 -f cnpg-cluster-chart/cnpg-cluster-override-values.yaml
```

To get activity status of the deployed CloudNativePG cluster using the CNPG Kubernetes CLI plugin `cnpg`.

```shell
kubectl cnpg status  -n enginevector-cnpg-cluster
```

To uninstall the Helm chart.

```shell
helm uninstall enginevector-cnpg-cluster --namespace enginevector-cnpg-cluster
```


## Pre-Flight Check Connection to the Cluster EngineVector Lottery Game Database

From the one of the CNPG Cluster pods connect to the EQL database providing (non-production) username and password credentials.

Finding the Postgres `pg_hba.conf` on any of the Cluster pods after exec into the Pod to qualify connection config.

```shell
find / -name "pg_hba.conf" 2>/dev/null
```

Do cat of the pg_hba.conf file in the path returned as.
```shell
cat /var/lib/postgresql/data/pgdata/pg_hba.conf
```

The `pg_hba.conf` should look as follows.

```shell
#
# FIXED RULES
#

# Grant local access ('local' user map)
local all all peer map=local

# Require client certificate authentication for the streaming_replica user
hostssl postgres streaming_replica all cert
hostssl replication streaming_replica all cert
hostssl all cnpg_pooler_pgbouncer all cert

#
# USER-DEFINED RULES
#

host    enginevector    enginevector    0.0.0.0/0    md5
host    all             all             ::/0         md5

#
# DEFAULT RULES
#
host    all    all    all         scram-sha-256  # turn on as default for non-prod testing
```

In the chart values yaml file the section according to CloudNativePG docs to configure the `pg_hba.conf` is shown.

```yaml
postgresql:
    ...
    pg_hba:
    {{- range .Values.cnpg.postgresql.pg_hba }}
      - {{ . }}
    {{- end }}
```

The corresponding values yaml file is shown here.

```yaml
postgresql:
    ...
    pg_hba:
      - host    all    all    0.0.0.0/0    md5
      - host    all    all    ::/0         md5
      - host    all    all    all          md5
```

The preceding configuration is for non-prod and for production should use TLS certificates.

To test the connection to any of the CloudNativePG Cluster pods and issue a `\dt` to show all EQL datbase tables.

```shell
kubectl exec -it <pod-name> -n cluster-deploy-sql -- /bin/bash
PGPASSWORD=<apppassword> psql -U <appusername> -d eql
```

This (if working) should show a `psql` prompt to issue the `\dt` command.

or

```shell
kubectl exec -it <pod-name> -n cluster-deploy-sql -- /bin/bash
PGPASSWORD=<apppassword> psql -U <appusername> -d enginevector -c "SELECT 1;"
```



## Lottery Games Postgres DB Schema

The provided `enginevector-lottery-schema.sql` schema file is provided to create the CNPG Cluster database for validation of deployed Helm Chart resources IF and ONLY IF the official `eql-schema.sql` schema is down for changes and additional CNPG Cluster changes occurr separately. 

The following is the schema for this.

```sql
-- Ensure the pgcrypto extension is enabled for UUID generation
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Create `enginevector_games` table
CREATE TABLE enginevector_games (
    game_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    game_name VARCHAR(100) NOT NULL,
    start_time TIMESTAMP WITHOUT TIME ZONE NOT NULL,
    end_time TIMESTAMP WITHOUT TIME ZONE,
    status VARCHAR(50) NOT NULL
);

-- Index on `game_name` to speed up queries by game name
CREATE INDEX idx_game_name ON eql_games (game_name);

-- Index on `status` to speed up status-based queries
CREATE INDEX idx_game_status ON enginevector_games (status);

-- Create `enginevector_game_tickets` table
CREATE TABLE enginevector_game_tickets (
    ticket_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    game_id UUID NOT NULL,
    purchase_time TIMESTAMP WITHOUT TIME ZONE NOT NULL,
    player_id UUID NOT NULL,
    ticket_number VARCHAR(20) UNIQUE NOT NULL,
    status VARCHAR(50) NOT NULL,
    prize_amount NUMERIC,
    FOREIGN KEY (game_id) REFERENCES eql_games (game_id) ON DELETE CASCADE
);

-- Index on `game_id` for faster joins and lookups by game
CREATE INDEX idx_ticket_game_id ON enginevector_game_tickets (game_id);

-- Index on `player_id` for faster joins and lookups by player
CREATE INDEX idx_ticket_player_id ON enginevector_game_tickets (player_id);

-- Create `enginevector_game_players` table
CREATE TABLE enginevector_game_players (
    player_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    player_name VARCHAR(100) NOT NULL,
    email VARCHAR(100) UNIQUE,
    join_date TIMESTAMP WITHOUT TIME ZONE DEFAULT CURRENT_TIMESTAMP
);

-- Index on `player_name` to speed up searches by player name
CREATE INDEX idx_player_name ON enginevector_game_players (player_name);

-- Create `enginevector_game_player_rankings` table
CREATE TABLE enginevector_game_player_rankings (
    ranking_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    player_id UUID NOT NULL,
    game_id UUID NOT NULL,
    ranking INTEGER NOT NULL,
    points NUMERIC,
    FOREIGN KEY (player_id) REFERENCES enginevector_game_players (player_id) ON DELETE CASCADE,
    FOREIGN KEY (game_id) REFERENCES enginevector_games (game_id) ON DELETE CASCADE
);

-- Index on `player_id` to speed up joins and lookups by player in rankings
CREATE INDEX idx_ranking_player_id ON enginevector_game_player_rankings (player_id);

-- Index on `game_id` for faster joins with games in the rankings table
CREATE INDEX idx_ranking_game_id ON enginevector_game_player_rankings (game_id);



-- ALTER TABLE and GRANT PRIVLEGES statements for eql_games

-- Run this as the postgres user to transfer ownership of all tables to eql:
ALTER TABLE enginevector_games OWNER TO eenginevectorql;
ALTER TABLE enginevector_game_tickets OWNER TO enginevector;
ALTER TABLE enginevector_game_players OWNER TO enginevector;
ALTER TABLE enginevector_game_player_rankings OWNER TO enginevector;

-- As postgres, grant privileges:
GRANT ALL PRIVILEGES ON ALL TABLES IN SCHEMA public TO enginevector;
GRANT ALL PRIVILEGES ON ALL SEQUENCES IN SCHEMA public TO enginevector;
GRANT ALL PRIVILEGES ON DATABASE enginevector TO enginevector;



-- Insert a game into `enginevector_games`
INSERT INTO enginevector_games (game_name, start_time, end_time, status)
VALUES ('Poker Train', '2024-11-01 10:00:00', '2024-11-01 22:00:00', 'active');

-- Insert a player into `enginevector_game_players`
INSERT INTO enginevector_game_players (player_name, email, join_date)
VALUES ('Count Dracula', 'countdracula@transylvanians.com', '2024-11-01 09:30:00');

-- Insert a ticket for the game and player into `enginevector_game_tickets`
INSERT INTO enginevector_game_tickets (game_id, purchase_time, player_id, ticket_number, status, prize_amount)
VALUES (
    (SELECT game_id FROM enginevector_games WHERE game_name = 'Poker Train'),  -- Linking to the created game
    '2024-11-01 10:15:00',
    (SELECT player_id FROM enginevector_game_players WHERE player_name = 'Count Dracula'),  -- Linking to the created player
    'TICKET12345',
    'pending',
    100.50
);

-- Insert a ranking for the player in the game into `enginevector_game_player_rankings`
INSERT INTO enginevector_game_player_rankings (player_id, game_id, ranking, points)
VALUES (
    (SELECT player_id FROM enginevector_game_players WHERE player_name = 'Count Dracula'),
    (SELECT game_id FROM enginevector_games WHERE game_name = 'Poker Train'),
    1,
    1500
);
```


### Lottery Games DB Chart Installation


To do a pre-flight pre-render using `helm template` do the following.

```shell
helm template cnpg-proxy-cluster-writer-svc-config . \
--namespace enginevector-cnpg-cluster \
--version 1.0.0 
```

To install without the dry-run.

```shell
helm install cnpg-proxy-cluster-writer-svc-config . \
--namespace enginevector-cnpg-cluster \
--version 1.0.0
``` 

To apply changes to the Helm Chart and do upgrade lifecycles.

```shell
helm upgrade cnpg-proxy-cluster-writer-svc-config . \
--namespace enginevector-cnpg-cluster \
--version 1.0.0
```

To get activity status of the deployed CNPG cluster using the CNPG Kubernetes CLI plugin `cnpg`.

```shell
kubectl cnpg status  -n enginevector-cnpg-cluster
```

To uninstall the Helm chart.

```shell
helm uninstall cnpg-proxy-cluster-writer-svc-config --namespace enginevector-cnpg-cluster
```






## References

