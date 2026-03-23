# Realtime Event Engine

Engine de processamento de eventos em tempo real, construído em Go, para cenários de logística, telemetria IoT e monitoramento operacional com alta vazão, processamento assíncrono e observabilidade completa.

## Visão Geral

Este projeto tem como objetivo receber milhares de eventos por segundo, validar o payload, enriquecer os dados com informações em cache, aplicar controle de pressão no pipeline e persistir os eventos de forma eficiente para consumo analítico e operacional.

O sistema é pensado para ambientes onde há grande volume de mensagens, picos de carga imprevisíveis e necessidade de resposta resiliente. Em vez de um CRUD tradicional, a proposta é uma arquitetura orientada a eventos com desacoplamento entre ingestão, processamento e armazenamento.

## Problema que o Projeto Resolve

Em sistemas de logística e IoT, é comum lidar com:

- eventos gerados por milhares de dispositivos, veículos ou sensores;
- bursts de tráfego em horários específicos;
- necessidade de validação rápida sem bloquear a ingestão;
- enriquecimento com dados contextuais, como rota, cliente, dispositivo ou configuração;
- persistência otimizada para consulta posterior;
- rastreamento e monitoramento da saúde do pipeline em tempo real.

O `realtime-event-engine` é projetado para resolver esse fluxo fim a fim.

## Objetivos do Projeto

- Receber eventos em alta taxa com baixa latência.
- Desacoplar ingestão e processamento via RabbitMQ.
- Validar e normalizar eventos com workers escritos em Go.
- Enriquecer eventos consultando Redis como camada de cache.
- Persistir dados de forma eficiente, com escrita em lote quando necessário.
- Aplicar backpressure para proteger o sistema sob carga.
- Expor métricas para Prometheus e dashboards no Grafana.
- Permitir evolução futura para particionamento, retries, DLQ e processamento paralelo por chave.

## Casos de Uso de Referência

### Logística

- rastreamento de posição de veículos;
- atualização de status de entrega;
- monitoramento de temperatura ou abertura de compartimentos;
- detecção de atraso ou desvio de rota.

### IoT e Telemetria

- sensores industriais enviando leituras periódicas;
- gateways transmitindo eventos de conectividade;
- dispositivos reportando estado, bateria e falhas;
- geração de alertas a partir de thresholds ou anomalias.

## Requisitos Funcionais

- Receber eventos por HTTP e, futuramente, também por gRPC.
- Publicar eventos recebidos em fila para processamento assíncrono.
- Validar estrutura, campos obrigatórios, tipos e regras de negócio básicas.
- Enriquecer eventos com metadados vindos de Redis.
- Persistir eventos válidos em armazenamento principal.
- Direcionar eventos inválidos ou não processáveis para retry ou DLQ.
- Expor status operacional e métricas técnicas.

## Requisitos Não Funcionais

- Alta vazão.
- Escalabilidade horizontal.
- Tolerância a picos de carga.
- Idempotência sempre que aplicável.
- Observabilidade desde o início.
- Estrutura preparada para execução em containers.
- Código organizado para manutenção e evolução de longo prazo.

## Stack Tecnológica

### Linguagem principal

- Go

### Componentes de infraestrutura

- RabbitMQ para mensageria e desacoplamento do pipeline
- Redis para cache de enriquecimento e dados de apoio de baixa latência
- Banco de dados relacional ou analítico para persistência final
- Prometheus para coleta de métricas
- Grafana para visualização e dashboards
- Docker e Docker Compose para ambiente local

### Bibliotecas e abordagens sugeridas em Go

- `Fiber` para API HTTP
- `amqp091-go` para integração com RabbitMQ
- `go-redis` para Redis
- `pgx` se a persistência inicial for PostgreSQL
- `promhttp` e ecossistema Prometheus para métricas
- worker pools com goroutines, channels e context cancellation

## Arquitetura Proposta

O projeto será dividido em componentes bem definidos para separar responsabilidades e facilitar escala independente.

### Componentes principais

#### 1. Ingestion API

Responsável por receber eventos de produtores externos.

Responsabilidades:

- autenticar ou identificar origem do evento;
- validar formato mínimo do payload;
- gerar metadados iniciais, como `received_at` e `request_id`;
- publicar no RabbitMQ;
- responder rapidamente ao cliente sem aguardar o processamento completo.

#### 2. Broker de Mensagens

RabbitMQ será o backbone assíncrono do pipeline.

Responsabilidades:

- desacoplar recepção e processamento;
- absorver bursts temporários;
- distribuir eventos entre consumidores;
- permitir retry, dead-letter queues e políticas por fila.

#### 3. Processor Workers

Workers em Go consumirão mensagens da fila e executarão o pipeline de negócio.

Responsabilidades:

- deserialização;
- validação de regras;
- enriquecimento com Redis;
- transformação para modelo interno;
- encaminhamento para persistência;
- envio para fila de erro quando necessário.

#### 4. Enrichment Cache

Redis será usado para armazenar dados consultados com alta frequência.

Exemplos:

- cadastro resumido de dispositivo;
- configuração de sensor;
- vínculo entre veículo e rota;
- parâmetros de cliente ou tenant;
- dados operacionais necessários para enriquecimento.

#### 5. Storage Writer

Camada responsável por persistir eventos de maneira eficiente.

Responsabilidades:

- escrita individual ou em lote;
- controle de flush por tempo e tamanho;
- isolamento entre processamento e storage;
- tratamento de falhas de escrita e reprocessamento.

#### 6. Observability Stack

Prometheus e Grafana serão integrados desde as primeiras entregas.

Responsabilidades:

- métricas de throughput, erro e latência;
- filas, backlog e taxa de consumo;
- hit ratio de cache;
- health checks e indicadores de saturação;
- dashboards operacionais.

## Fluxo de Processamento

1. O produtor envia um evento para a API.
2. A API valida o envelope mínimo e publica a mensagem no RabbitMQ.
3. Um ou mais workers consomem a fila.
4. O evento é validado em profundidade.
5. O worker consulta Redis para enriquecer o payload.
6. O evento enriquecido é encaminhado para persistência.
7. Em caso de erro transitório, ocorre retry.
8. Em caso de erro definitivo, o evento segue para DLQ.
9. Métricas e logs são emitidos ao longo de todo o fluxo.

## Desenho de Alto Nível

```text
Producers
   |
   v
[Ingestion API - Go]
   |
   v
[RabbitMQ Exchange/Queue]
   |
   v
[Processing Workers - Go] ---> [Redis]
   |
   v
[Writer / Batch Persist]
   |
   v
[Primary Storage]

Metrics -> Prometheus -> Grafana
Errors  -> Retry Queue / DLQ
```

## Estratégia de Backpressure

Backpressure é requisito central deste projeto. O sistema não deve aceitar carga infinita sem controle, nem propagar saturação silenciosa até colapsar.

### Estratégias previstas

- filas internas limitadas em memória nos workers;
- worker pool com concorrência configurável;
- `prefetch` controlado no RabbitMQ para evitar consumo acima da capacidade real;
- rejeição controlada ou degradação quando a ingestão ultrapassar limites;
- batch writer com limites de tamanho e tempo;
- timeouts e circuit breaking nas chamadas externas;
- rate limiting por origem, tenant ou dispositivo;
- métricas de saturação para ajuste operacional.

### Objetivo operacional

Quando houver pico de carga, o sistema deve desacelerar de forma previsível:

- priorizando estabilidade;
- preservando mensagens já aceitas;
- expondo backlog claramente;
- evitando explosão de memória e cascata de falhas.

## Estratégia de Particionamento

O projeto deve nascer preparado para paralelismo orientado por chave.

### Chaves de particionamento candidatas

- `tenant_id`
- `device_id`
- `vehicle_id`
- janela temporal, como dia ou hora

### Motivações

- preservar ordem relativa por entidade quando necessário;
- distribuir melhor o processamento entre workers;
- facilitar escalabilidade horizontal;
- apoiar escrita e consulta eficiente no storage;
- reduzir contenção em cenários de alto volume.

### Aplicações práticas

- filas ou rotas específicas por domínio;
- sharding lógico por tenant;
- particionamento no banco por tempo e entidade;
- workers especializados por tipo de evento.

## Modelo Conceitual do Evento

Exemplo de envelope base:

```json
{
  "event_id": "evt_01HQZP6R8J3G7A1F0K2M4N5P6Q",
  "event_type": "vehicle.position.updated",
  "occurred_at": "2026-03-22T14:30:00Z",
  "producer": "tracking-gateway-01",
  "tenant_id": "tenant-acme",
  "device_id": "truck-2048",
  "payload": {
    "lat": -23.5505,
    "lng": -46.6333,
    "speed": 68.4,
    "temperature": 4.8
  }
}
```

### Campos esperados

- `event_id`: identificador único para rastreabilidade e idempotência;
- `event_type`: tipo do evento;
- `occurred_at`: horário de ocorrência no produtor;
- `producer`: origem lógica do evento;
- `tenant_id`: escopo organizacional;
- `device_id`: entidade principal do evento;
- `payload`: conteúdo específico do domínio.

### Campos enriquecidos internamente

- `received_at`
- `processed_at`
- `route_id`
- `customer_id`
- `device_profile`
- `validation_status`
- `processing_attempt`

## Estratégia de Persistência

O storage final poderá variar conforme o foco da solução, mas a documentação assume uma fase inicial com PostgreSQL pela simplicidade operacional e boa integração com Go.

### Diretrizes

- escrita orientada a lote quando possível;
- índices mínimos e bem planejados;
- particionamento por tempo em fases posteriores;
- separação entre tabelas operacionais e analíticas, se necessário;
- idempotência de gravação com base em `event_id`.

### Evoluções possíveis

- PostgreSQL particionado;
- TimescaleDB para séries temporais;
- ClickHouse para analytics de alta escala;
- data lake ou fila secundária para consumo analítico.

## Confiabilidade e Tratamento de Falhas

### Cenários previstos

- evento inválido por schema ou regra de negócio;
- falha temporária no Redis;
- indisponibilidade de banco;
- erro de serialização;
- consumo acima da capacidade configurada.

### Abordagens

- retries com política limitada;
- DLQ para inspeção e reprocessamento;
- logs estruturados com correlação por `event_id` e `request_id`;
- health checks por dependência;
- métricas para falhas por etapa do pipeline;
- graceful shutdown para evitar perda de mensagens em deploy ou restart.

## Observabilidade

Observabilidade não será tratada como etapa final. Ela faz parte do desenho do sistema.

### Métricas principais

- eventos recebidos por segundo;
- eventos processados por segundo;
- latência média e percentis do pipeline;
- taxa de erro por etapa;
- tamanho do backlog nas filas;
- tempo de permanência na fila;
- taxa de retry e volume em DLQ;
- cache hit/miss no Redis;
- tempo de escrita no banco.

### Dashboards esperados no Grafana

- visão executiva de throughput e erro;
- visão operacional de filas e consumidores;
- visão de infraestrutura por serviço;
- visão de qualidade do dado e rejeições;
- visão de performance do storage.

## Estrutura Sugerida do Repositório

Como o projeto será implementado em Go, esta é uma estrutura base recomendada:

```text
.
├── cmd/
│   ├── api/
│   ├── processor/
│   └── writer/
├── internal/
│   ├── app/
│   ├── config/
│   ├── domain/
│   ├── ingestion/
│   ├── messaging/
│   ├── processing/
│   ├── enrichment/
│   ├── persistence/
│   ├── observability/
│   └── platform/
├── pkg/
├── deployments/
│   ├── docker/
│   ├── compose/
│   └── grafana/
├── scripts/
├── test/
└── README.md
```

## Serviços Planejados

### `cmd/api`

Serviço de entrada para receber eventos e publicá-los no RabbitMQ.

### `cmd/processor`

Serviço que consome mensagens, valida e enriquece eventos.

### `cmd/writer`

Serviço responsável por persistência especializada, especialmente se for necessário desacoplar escrita e processamento.

## Configuração Prevista por Ambiente

Principais variáveis esperadas:

- `APP_ENV`
- `HTTP_PORT`
- `RABBITMQ_URL`
- `RABBITMQ_EXCHANGE`
- `RABBITMQ_QUEUE`
- `RABBITMQ_PREFETCH`
- `REDIS_ADDR`
- `REDIS_PASSWORD`
- `DATABASE_URL`
- `WORKER_CONCURRENCY`
- `BATCH_SIZE`
- `BATCH_FLUSH_INTERVAL`
- `RATE_LIMIT_RPS`
- `PROMETHEUS_PORT`

## Critérios de Sucesso

O projeto será considerado bem sucedido quando atingir:

- pipeline estável sob carga contínua;
- ingestão desacoplada do processamento;
- observabilidade suficiente para operação;
- recuperação controlada de falhas;
- base sólida para escalar horizontalmente.

## Fora do Escopo Inicial

Para manter foco na primeira fase, alguns itens não entram no MVP:

- machine learning para detecção de anomalia;
- processamento complexo de stream analytics;
- multi-region ativo-ativo;
- interface web própria de operação;
- orquestração em Kubernetes nas primeiras entregas.

## Estratégia de Entrega

O projeto será construído por incrementos, priorizando uma base operacional antes de otimizações avançadas.

## Roadmap

### Entrega 1. Fundação do Projeto

- inicializar módulo Go;
- definir estrutura de pastas;
- criar configuração centralizada;
- preparar Docker Compose com RabbitMQ, Redis, Prometheus e Grafana;
- definir padrão de logs, métricas e health checks;
- documentar arquitetura e decisões iniciais.

### Entrega 2. Ingestão de Eventos

- implementar API HTTP para recebimento de eventos;
- validar envelope mínimo;
- publicar mensagens no RabbitMQ;
- instrumentar métricas de entrada, erro e latência;
- adicionar testes básicos de contrato.

### Entrega 3. Processamento Assíncrono

- implementar consumers em Go;
- aplicar validação de negócio;
- estruturar worker pool com concorrência configurável;
- configurar ACK, retry e DLQ;
- registrar métricas de consumo e falha.

### Entrega 4. Enriquecimento com Redis

- integrar Redis ao pipeline;
- definir estratégia de cache lookup;
- tratar cache miss e fallback;
- medir latência, hit rate e impacto no throughput.

### Entrega 5. Persistência Eficiente

- implementar camada de escrita;
- suportar batch insert;
- definir idempotência por `event_id`;
- preparar schema inicial do banco;
- medir latência e taxa de gravação.

### Entrega 6. Backpressure e Resiliência

- limitar buffers internos;
- ajustar `prefetch` e concorrência;
- aplicar rate limiting e políticas de overload;
- validar comportamento sob pico de carga;
- documentar limites operacionais.

### Entrega 7. Observabilidade Completa

- expandir métricas Prometheus;
- criar dashboards Grafana;
- definir alertas principais;
- acompanhar backlog, retries, DLQ e saturação;
- consolidar health endpoints e readiness checks.

### Entrega 8. Escala e Particionamento

- definir estratégia de particionamento por chave;
- avaliar filas por domínio ou tenant;
- preparar storage para particionamento temporal;
- executar testes de carga com throughput crescente;
- revisar gargalos de CPU, memória, broker e banco.

### Entrega 9. Hardening para Produção

- revisar segurança e autenticação;
- adicionar políticas de retenção e reprocessamento;
- formalizar runbooks operacionais;
- endurecer deploy, rollback e graceful shutdown;
- preparar baseline para ambiente produtivo.
