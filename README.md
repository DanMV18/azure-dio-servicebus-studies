# Sistema de Emissão Distribuída de Boletos Bancários

## Infraestrutura de Alta Performance com .NET 8 e Azure Service Bus
📑 1. Introdução e Contexto Estratégico
Este repositório apresenta uma solução de missão crítica projetada para resolver um dos desafios mais sensíveis do setor de Fintechs e sistemas de ERP: a geração massiva e resiliente de documentos de cobrança (Boletos Bancários).

Em ambientes financeiros de alto volume, a emissão de boletos não é apenas uma tarefa de formatação de dados; é um processo computacionalmente intensivo que envolve cálculos precisos de dígitos verificadores (Linha Digitável e Código de Barras), renderização de buffers complexos de PDF e integrações síncronas ou assíncronas com APIs de registro bancário.

1.1. O Problema da Carga Volátil
Tradicionalmente, sistemas monolíticos sofrem degradação de performance durante janelas de faturamento mensal, onde milhares de requisições de emissão ocorrem simultaneamente. Esse cenário resulta em:

Latência Excessiva: O usuário aguarda segundos ou minutos pela geração do PDF.

Timeouts de Conexão: Quedas de serviço devido ao esgotamento de threads no servidor de aplicação.

Inconsistência de Dados: Falhas no meio do processo que deixam o sistema sem saber se o boleto foi ou não registrado no banco.

1.2. A Proposta de Solução: Arquitetura Orientada a Eventos
Este projeto utiliza o Azure Service Bus como o núcleo de orquestração para implementar o padrão Queue-Based Load Leveling (Nivelamento de Carga Baseado em Fila). Através do desacoplamento total entre o front-end (Producer) e o motor de geração (Consumer), garantimos:

Elasticidade: O sistema absorve picos de carga instantâneos, enfileirando as solicitações para processamento conforme a disponibilidade de recursos.

Resiliência e Tolerância a Falhas: Através de políticas de Retry e Dead Letter Queues (DLQ), asseguramos que nenhuma cobrança seja perdida por instabilidades momentâneas em serviços de terceiros.

Experiência do Usuário (UX): A interface responde de forma instantânea ao cliente, confirmando o recebimento da solicitação, enquanto o processamento ocorre em segundo plano.

1.3. Pilares Tecnológicos
A stack foi selecionada visando a modernidade e o suporte de longo prazo (LTS):

.NET 8: Aproveitando as melhorias de performance do runtime e as novas APIs de serialização JSON.

Azure Service Bus (Standard/Premium): Utilizado para garantir a entrega de mensagens no estilo at-least-once e suporte a transações atômicas.

---

## 🏗️ 2. Arquitetura do Sistema

A solução foi desenhada seguindo os princípios de **Sistemas Distribuídos** e **Clean Architecture**. O fluxo de dados segue o padrão de mensageria assíncrona para garantir o desacoplamento total entre a origem do pedido e a execução da lógica de negócio.

### 2.1. Componentes Principais

| Componente | Função | Tecnologia |
| :--- | :--- | :--- |
| **API Gateway / Entrypoint** | Recebe as requisições REST, valida o payload e publica no barramento. | ASP.NET Core Web API |
| **Message Broker** | Orquestra a fila de mensagens, gerencia concorrência e retentativas. | Azure Service Bus (Standard/Premium) |
| **Boleto Processor (Worker)** | Consome a fila, executa a lógica financeira e gera o artefato final. | .NET Worker Service |
| **Persistence Layer** | Armazena o estado da transação e metadados do boleto. | Azure SQL / Entity Framework Core |
| **Blob Storage** | Armazena o arquivo PDF final gerado para download posterior. | Azure Blob Storage |

### 2.2. Diagrama de Fluxo Lógico

1.  **Ingestão:** O usuário/sistema externo envia um JSON para a API.
2.  **Validação:** A API valida a integridade dos dados (CPF/CNPJ, valores, datas).
3.  **Enfileiramento:** Uma mensagem é postada na fila `financeiro-geracao-boleto`.
4.  **Processamento:** O Worker Service "escuta" a fila e bloqueia a mensagem (*Peek-Lock*).
5.  **Geração:** O Worker utiliza a engine `BoletoNetCore` para renderizar o boleto.
6.  **Finalização:** O PDF é persistido, e a mensagem é completada na fila.

---

## 🛠️ 3. Pré-requisitos e Dependências

Para executar, compilar e implantar este projeto, você precisará de:

### 3.1. Ambiente de Desenvolvimento
* **SDK .NET 8.0** (ou superior).
* **IDE:** Visual Studio 2022, VS Code ou JetBrains Rider.
* **Docker** (opcional, para rodar dependências locais como SQL Server).

### 3.2. Infraestrutura Azure
* **Namespace do Service Bus:** Com uma fila configurada.
* **Storage Account:** Para armazenamento dos arquivos PDF.
* **Application Insights:** Para monitoramento de telemetria e logs.

---

## 🚀 4. Configuração da Infraestrutura (Step-by-Step)

### Configurando o Azure Service Bus
1.  Acesse o [Portal do Azure](https://portal.azure.com).
2.  Crie um novo **Service Bus Namespace**.
3.  Dentro do Namespace, crie uma **Queue** chamada `cmd-gerar-boleto-v1`.
4.  **Configurações Importantes da Fila:**
    * **Max Delivery Count:** Defina como `5` (número de tentativas antes de ir para a DLQ).
    * **Lock Duration:** Defina como `1 minuto` (tempo de exclusividade do worker sobre a mensagem).
    * **Enable Dead Lettering on Message Expiration:** Ativado.

---

## 💻 5. Implementação Técnica Detalhada

### 5.1. O Contrato de Dados (Message Schema)
Utilizamos uma abordagem de contratos compartilhados para garantir que o Produtor e o Consumidor falem a mesma língua.

```csharp
namespace Financeiro.Domain.Messages
{
    public record GerarBoletoCommand
    {
        public Guid TransactionId { get; init; }
        public string CedenteCpfCnpj { get; init; }
        public string SacadoNome { get; init; }
        public decimal Valor { get; init; }
        public DateTime DataVencimento { get; init; }
        public int BancoCodigo { get; init; } // Ex: 001 (BB), 237 (Bradesco)
        public string NossoNumero { get; init; }
    }
}
```

### 5.2. O Worker Service (Consumer)
O Worker implementa o padrão `BackgroundService`. Abaixo, um exemplo da lógica de consumo resiliente:

```csharp
protected override async Task ExecuteAsync(CancellationToken stoppingToken)
{
    _processor.ProcessMessageAsync += async args =>
    {
        var body = args.Message.Body.ToString();
        var command = JsonConvert.DeserializeObject<GerarBoletoCommand>(body);

        try 
        {
            // 1. Lógica de Geração do Boleto
            var pdfBuffer = await _boletoService.GerarPdfAsync(command);

            // 2. Upload para o Blob Storage
            await _storageService.UploadAsync($"{command.TransactionId}.pdf", pdfBuffer);

            // 3. Marcar mensagem como concluída
            await args.CompleteMessageAsync(args.Message);
        }
        catch (Exception ex)
        {
            _logger.LogError(ex, "Erro ao processar boleto {Id}", command.TransactionId);
            // A mensagem voltará para a fila para nova tentativa conforme política de Retry
            await args.AbandonMessageAsync(args.Message);
        }
    };

    await _processor.StartProcessingAsync(stoppingToken);
}
```

---

## 🛡️ 6. Resiliência e Tratamento de Erros

O sistema foi projetado para ser "Fail-Safe":

1.  **Retentativas (Retry Policy):** Se o banco de dados estiver temporariamente fora do ar, o Service Bus reintroduz a mensagem na fila após alguns segundos.
2.  **Dead Letter Queue (DLQ):** Se após 5 tentativas o boleto não puder ser gerado (ex: dados inválidos), a mensagem é movida para a DLQ. Isso evita o travamento da fila principal.
3.  **Idempotência:** O sistema verifica no banco de dados se o `TransactionId` já possui um boleto gerado antes de iniciar o processo, evitando cobranças duplicadas em caso de reprocessamento.

---

## 📊 7. Observabilidade e Monitoramento

Para operações financeiras, a visibilidade é crucial. O projeto utiliza:

* **Serilog:** Com *Sinks* para o Azure Application Insights, permitindo rastreamento ponta a ponta através do `CorrelationId`.
* **Health Checks:** Endpoints `/health` na API e no Worker para monitorar a conectividade com o Service Bus e SQL.
* **Métricas de Fila:** Monitoramento do "Message Count" e "Active Messages" para gatilhos de escalonamento automático (Auto-scaling).

---

## ⚙️ 8. Configuração de Variáveis de Ambiente

No seu `appsettings.json`, configure as seguintes chaves:

```json
{
  "ConnectionStrings": {
    "ServiceBus": "Endpoint=sb://SUO-NAMESPACE.servicebus.windows.net/;...",
    "StorageAccount": "DefaultEndpointsProtocol=https;AccountName=...",
    "SqlDatabase": "Server=tcp:seu-servidor.database.windows.net..."
  },
  "BoletoSettings": {
    "QueueName": "cmd-gerar-boleto-v1",
    "Environment": "Production"
  }
}
```

---

## 🧪 9. Testes e Validação

### Testes de Unidade
Focados na lógica financeira de cálculo de juros, multas e dígito verificador do código de barras.
`dotnet test src/Financeiro.Tests.Unit`

### Testes de Integração
Validam a publicação e o consumo de mensagens utilizando um emulador de Service Bus ou ambiente de Sandbox.

---

## 📜 10. Licença e Contribuição

1.  Faça um **Fork** do projeto.
2.  Crie uma **Feature Branch** (`git checkout -b feature/novo-banco`).
3.  Envie um **Pull Request** detalhando as alterações.

Este projeto está licenciado sob a **MIT License**.

---

## 🛠️ GitHub Actions Workflow (`.github/workflows/azure-deploy.yml`)

Este arquivo deve ser colocado na pasta `.github/workflows/` na raiz do seu repositório.

```yaml
name: CI/CD - Sistema de Boletos Azure

on:
  push:
    branches: [ "main" ]
  pull_request:
    branches: [ "main" ]

env:
  DOTNET_VERSION: '8.0.x'
  API_PACKAGE_PATH: './publish-api'
  WORKER_PACKAGE_PATH: './publish-worker'
  # Substitua pelos nomes reais dos seus recursos no Azure
  AZURE_API_APP_NAME: 'webapp-boleto-api-prod'
  AZURE_WORKER_APP_NAME: 'webapp-boleto-worker-prod'

jobs:
  build:
    name: Build & Test
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: ${{ env.DOTNET_VERSION }}

      - name: Restore dependencies
        run: dotnet restore src/BoletoGenerator.sln

      - name: Build
        run: dotnet build src/BoletoGenerator.sln --configuration Release --no-restore

      - name: Test
        run: dotnet test src/BoletoGenerator.sln --configuration Release --no-build --verbosity normal

      - name: Publish API
        run: dotnet publish src/BoletoGenerator.API/BoletoGenerator.API.csproj -c Release -o ${{ env.API_PACKAGE_PATH }}

      - name: Publish Worker
        run: dotnet publish src/BoletoGenerator.Worker/BoletoGenerator.Worker.csproj -c Release -o ${{ env.WORKER_PACKAGE_PATH }}

      - name: Upload API Artifact
        uses: actions/upload-artifact@v4
        with:
          name: api-app
          path: ${{ env.API_PACKAGE_PATH }}

      - name: Upload Worker Artifact
        uses: actions/upload-artifact@v4
        with:
          name: worker-app
          path: ${{ env.WORKER_PACKAGE_PATH }}

  deploy:
    name: Deploy to Azure
    needs: build
    runs-on: ubuntu-latest
    environment: production

    steps:
      - name: Download API Artifact
        uses: actions/download-artifact@v4
        with:
          name: api-app
          path: ./api

      - name: Download Worker Artifact
        uses: actions/download-artifact@v4
        with:
          name: worker-app
          path: ./worker

      # Login no Azure usando Service Principal (Recomendado)
      # ou via Publish Profile (Simplificado)
      
      - name: 'Deploy API to Azure Web App'
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_API_APP_NAME }}
          publish-profile: ${{ secrets.AZURE_API_PUBLISH_PROFILE }}
          package: ./api

      - name: 'Deploy Worker to Azure Web App'
        uses: azure/webapps-deploy@v3
        with:
          app-name: ${{ env.AZURE_WORKER_APP_NAME }}
          publish-profile: ${{ secrets.AZURE_WORKER_PUBLISH_PROFILE }}
          package: ./worker
```

---

## 📖 Explicação dos Estágios

### 1. Gatilhos (Triggers)
O pipeline é acionado automaticamente em cada **push** ou **pull request** direcionado à branch `main`. Isso garante que o código em produção seja sempre validado antes do deploy.

### 2. Job de Build & Test
* **Restore & Build:** Garante que o código compila corretamente em ambiente isolado.
* **Test:** Executa a suite de testes (xUnit/NUnit). Se um teste falhar, o pipeline é interrompido imediatamente, impedindo que um bug chegue ao servidor.
* **Artifacts:** O comando `dotnet publish` gera os binários otimizados. Eles são "zipados" e armazenados temporariamente como artefatos do GitHub.

### 3. Job de Deploy
* **Separation of Concerns:** O deploy só inicia se o build terminar com sucesso (`needs: build`).
* **Artifact Download:** Recupera os binários gerados no estágio anterior.
* **Azure Web Apps Deploy:** Utiliza a action oficial da Microsoft para injetar o código no serviço de destino.

---

## 🔐 Configuração de Segurança (Secrets)

Para que o deploy funcione, você precisa configurar os **Secrets** no seu repositório GitHub (**Settings > Secrets and variables > Actions**):

1.  `AZURE_API_PUBLISH_PROFILE`: O conteúdo do arquivo de perfil de publicação (.publishsettings) baixado do seu App Service da API no portal do Azure.
2.  `AZURE_WORKER_PUBLISH_PROFILE`: O mesmo, mas referente ao recurso do Worker Service.

---
