###🎯 Objetivo

O principal objetivo deste workflow é automatizar a extração de dados-chave de contratos em formato PDF relacionados a um cliente específico (identificado pelo codigoCliente) e armazenar esses dados em um DataTable para acesso rápido e geração 
de relatórios formatados em HTML.

###⚙️ Fluxo de Processamento (Workflow)

O processo é dividido em duas principais rotas, começando com a entrada via Webhook e normalização de parâmetros.

1. Entrada e Validação

    Exportar dados (Webhook): Ponto de entrada que aguarda requisições HTTP para iniciar o fluxo.

    Code in JavaScript2 (Normalização): Normaliza os parâmetros de entrada (trackingId, codigoCliente, limit, offset, format) e realiza uma verificação de segurança opcional via X-Auth (se SEGREDO_STATUS estiver configurado no ambiente).

    Get row(s) (Verificação Cache): Consulta o DataTable zoho_plu_extract_dev para verificar se dados contratuais (isContract: true) já foram extraídos e armazenados para o codigoCliente fornecido.

2. Ramificação Condicional (If)

    If: Decide a rota de execução com base no resultado da verificação do DataTable:

        Rota 1: Cache (YES): Se houver dados pré-existentes, pula a etapa de extração de documentos e segue para a formatação e exibição do resultado.

        Rota 2: Extração (NO): Se não houver dados, inicia o processo de busca e extração de contratos.

3. Extração e Análise (Rota 2)

    Consulta no banco zohodadosdb (SQL): Busca na base de dados (SQL Server) os anexos (dbo.anexos) vinculados aos negócios (dbo.negocios) do codigoCliente.

    Formata os campos (Set): Prepara as variáveis para o download do S3 (configura bucket, key, region).

    Chamada no s3 para baixar os arquivos (AWS S3): Baixa o anexo do bucket group-zoho-anexos usando as credenciais AWS.

    Validação para saber se é um PDF (If): Filtra os arquivos, processando apenas aqueles cuja extensão termina em .pdf.

    Extração de PDF: Utiliza o nó extractFromFile para extrair o texto completo do PDF.

    Agente Inteligente responsável por extrair os campos (LLM Chain):

        Modelo: openai/gpt-4o-mini (via OpenRouter).

        Função: Analisa o texto extraído, determina se é um CONTRATO (isContract: true/false), e extrai os campos estruturados conforme um schema JSON rígido.

        Schema e Campos-Chave: fileName, isContract (booleano), razaoSocial, produtos, avisoPrevio, fidelizacao, multasRescisorias, renovacaoAutomatica ("sim" ou "nao"), e dataVencimentoLTDU.

4. Armazenamento e Finalização

    Code in JavaScript3 (Preservar Chave): Garante que a chave S3 (s3Key) original seja preservada junto com os dados extraídos pela IA.

    Insert row (DataTable): Insere os dados extraídos e estruturados (incluindo isContract e chaves) no DataTable zoho_plu_extract_dev para cache.

    Formata os campos1 (Code): Realiza a padronização e normalização final dos dados (ex: avisoPrevio para "60 dias", fidelizacao para "-", renovacaoAutomatica para "sim"/"nao").

    Filter: Filtra os resultados, exibindo apenas os itens onde isContract é verdadeiro (contratos).

    HTML formatado (Code): Constrói a resposta final em formato HTML a partir dos resultados filtrados, utilizando estilização para apresentação em tabela.
   
    Respond to Webhook1: Envia a resposta HTML formatada de volta ao solicitante com o status 200 OK e Content-Type: text/html.


### 🛠️ Credenciais Necessárias

Este workflow requer as seguintes credenciais configuradas no n8n:

| Credencial | Tipo | Uso |
| :--- | :--- | :--- |
| **OpenRouter account 3** | OpenRouter API Key | Para comunicação com o LLM (`openai/gpt-4o-mini`) para extração de dados. |
| **Microsoft SQL account 2** | Microsoft SQL | Para consultar a base de dados de anexos (`zohodadosdb`). |
| **AWS account 3** | AWS (S3) | Para baixar os arquivos anexados do bucket `group-zoho-anexos`. |
| **Header Auth account 2** | HTTP Header Auth | Opcional, para segurança de acesso ao Webhook via cabeçalho `X-Auth`. |
