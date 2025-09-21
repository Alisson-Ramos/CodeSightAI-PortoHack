# Relatório Preditivo de Operações Portuárias - CodeSightIA

Este é um workflow avançado do n8n projetado para automatizar a coleta, análise e notificação do status operacional de navios em um terminal portuário. O sistema agrega dados de múltiplas fontes, utiliza um agente de Inteligência Artificial para aplicar uma lógica de negócios complexa e gera um relatório preditivo detalhado que é enviado por e-mail e disponibilizado via API.

## Visão Geral

O objetivo principal deste protótipo é criar um "Relatório Preditivo de Situação Operacional" que centraliza informações de diversos stakeholders (Armador, Agência Marítima, Receita Federal, Anvisa, Capitania dos Portos) para fornecer uma visão unificada e inteligente das operações. Ele identifica proativamente gargalos, calcula atrasos preditivos e notifica as partes interessadas sobre pontos críticos.

## Como Funciona

O fluxo de trabalho é dividido em cinco etapas principais:

1.  **Trigger (Gatilho):** A execução pode ser iniciada de duas maneiras:
    * **`BUSCA PARA NOTIFICAÇÃO` (Schedule Trigger):** O workflow é executado automaticamente em intervalos regulares (a cada minuto, no exemplo) para monitoramento contínuo.
    * **`TRIGGER API` (Webhook):** O workflow pode ser acionado sob demanda por meio de uma chamada de API externa.

2.  **Coleta e Consolidação de Dados:**
    * O nó **`Armador`** busca a lista inicial de navios e suas operações.
    * O nó **`Split resultado`** divide a lista para processar cada navio individualmente.
    * Uma série de nós **`HTTPRequest`** (**`AgenciaMaritima`**, **`ReceitaFederal`**, **`Anvisa`**, **`CaptaniaDosPortos`**) consulta APIs específicas para obter o status detalhado de cada navio em diferentes órgãos reguladores.
    * O nó **`Combina os dados do navio nas APIs`** (Merge) une todas as informações coletadas em um único registro consolidado para cada navio usando uma consulta SQL.

3.  **Análise com Inteligência Artificial:**
    * O nó **`Logica do negócio e retorno para a IA`** prepara um prompt detalhado para a IA. Este prompt contém a persona, o objetivo, as regras de negócio e os dados consolidados do navio.
    * O nó **`AI Agent`**, utilizando o modelo **`OpenAI Chat Model`** (gpt-4.1-mini), processa o prompt. A IA analisa os dados com base nas regras de negócio para identificar pontos críticos, calcular atrasos preditivos e gerar um sumário executivo.
    * O resultado é um objeto JSON estruturado contendo o relatório completo.

4.  **Geração e Formatação do Relatório:**
    * O nó **`Conversão de dados`** limpa e converte a saída de texto da IA em um objeto JSON válido.
    * O nó **`Conversão para o HTML`** transforma dinamicamente o relatório JSON em um e-mail HTML bem formatado, com seções claras para o sumário, pontos críticos e próximas operações.

5.  **Distribuição e Saída:**
    * **`Envio do Email`:** O relatório em HTML é enviado via Gmail para os destinatários pré-definidos. O assunto do e-mail é dinâmico, baseado na notificação gerada pela IA (ex: "Alerta: X navios com pendências críticas.").
    * **`API RESPOSTA`:** O relatório completo em formato JSON é retornado como resposta para o webhook, caso o fluxo tenha sido iniciado por ele.

## Lógica de Negócios Aplicada pela IA

O núcleo do workflow reside no prompt enviado ao agente de IA. As regras de processamento são as seguintes:

* **Visão Geral:** Contabiliza o número total de operações e agrupa os navios por seu `statusOperacao`.
* **Identificação de Pontos Críticos:**
    * Utiliza uma lista de status negativos (ex: `pendente`, `atrasado`, `bloqueado`, `reprovado`).
    * Verifica os status de documentação, verificação fiscal, sanitário e autorização de múltiplos órgãos.
    * Se um status negativo é encontrado, o navio é marcado como um "ponto crítico", e os detalhes da pendência são registrados.
* **Cálculo do ETA Preditivo:**
    * Para cada ponto crítico, um atraso em horas é calculado com base na pendência:
        * **+4 horas** se a pendência for com a `Receita-Federal`.
        * **+2 horas** se a pendência for com a `Anvisa`.
    * A nova previsão de chegada (`previsaoChegada`) é calculada somando o atraso à data original.
* **Sumário e Notificação:**
    * Um **`sumarioExecutivo`** em linguagem natural é gerado com base nos dados calculados.
    * Uma **`push_notification`** curta e direta é criada para ser usada como alerta ou assunto de e-mail.

## Estrutura do Relatório de Saída (JSON)

A IA é instruída a retornar um objeto JSON com a seguinte estrutura:

```json
{
  "relatorioInfo": {
    "titulo": "Relatório Preditivo de Situação Operacional",
    "dataGeracao": "string",
    "versao": "1.2-unificado"
  },
  "visaoGeral": {
    "totalOperacoesTerminal": 0,
    "status": {}
  },
  "pontosCriticos": [
    {
      "identificadorNavio": "string",
      "dataPrevisaoChegadaOriginal": "string (formato ISO) ou null",
      "atrasoPreditivoHoras": "integer ou null",
      "previsaoChegada": "string (formato ISO) ou null",
      "situacao": "string ou null",
      "pendencias": [
        {
          "orgao": "string",
          "status": "string",
          "detalhe": "string",
          "impactoHoras": "integer ou null"
        }
      ]
    }
  ],
  "proximasOperacoes": {
    "chegadasConfirmadas": []
  },
  "sumarioExecutivo": "string",
  "push_notification": "string"
}
