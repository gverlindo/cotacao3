# Especificação: Estruturas de Dados

## 1. Objeto Principal: `State` da Cotação
Este é o objeto central que armazena todos os dados de uma cotação em andamento.

```json
{
  "dadosGerais": { ... },
  "adicoes": [ ... ],
  "transporte": { ... },
  "servicos": { ... },
  "despesasOperacionais": [ ... ],
  "ncmDataCache": { ... }
}
```

### 1.1. `dadosGerais`
Dados básicos da operação de importação.

- `paisOrigem`: (String) Código do país de origem. Ex: "CHN" para China.
- `ufDesembaraco`: (String) Sigla do estado de desembaraço. Ex: "SC" para Santa Catarina.
- `taxaCambio`: (Number) Valor da PTAX (taxa de câmbio USD/BRL). Ex: 5.45.
- `aliquotaIcms`: (Number) Alíquota padrão de ICMS do estado. Ex: 17 (para 17%).

**Exemplo:**
```json
{
  "paisOrigem": "CHN",
  "ufDesembaraco": "SC", 
  "taxaCambio": 5.45,
  "aliquotaIcms": 17
}
```

### 1.2. `adicoes` (Array de Objetos)
Lista de produtos/itens da cotação. Cada objeto representa uma adição.

- `id`: (Number) Identificador único do item na cotação.
- `ncm`: (String) Código NCM do produto (8 dígitos).
- `unidadeDeMedida`: (String) Unidade de medida comercializada (ex: "UN", "KG", "Pares"). Deve ser validada contra a unidade estatística da NCM.
- `qtd`: (Number) Quantidade de itens, na `unidadeDeMedida` especificada.
- `valorUnitarioUSD`: (Number) Valor do produto em USD, por unidade.
- `pesoUnitarioKg`: (Number) Peso do produto em Kg, por unidade.

**Exemplo:**
```json
[
  {
    "id": 1,
    "ncm": "85171231",
    "unidadeDeMedida": "UN",
    "qtd": 1000,
    "valorUnitarioUSD": 150.00,
    "pesoUnitarioKg": 0.2
  },
  {
    "id": 2,
    "ncm": "90065300",
    "unidadeDeMedida": "UN", 
    "qtd": 50,
    "valorUnitarioUSD": 400.00,
    "pesoUnitarioKg": 0.8
  }
]
```

### 1.3. `transporte`
Dados relacionados ao modal de transporte e condições comerciais.

- `modal`: (String) Modal de transporte. Valores válidos: "Maritimo FCL", "Maritimo LCL", "Aereo".
- `incoterm`: (String) Termo comercial internacional. Valores válidos: "EXW", "FOB", "CFR", "CIF", "FCA".
- `freteInternacionalUSD`: (Number) Valor do frete internacional em USD, obtido manualmente ou via estimativa.
- `seguroInternacionalUSD`: (Number) Valor do seguro internacional em USD, **sempre calculado pelo sistema**.
- `seguroContratado`: (Boolean) Flag para indicar se o usuário optou por contratar o seguro.
- `numeroContainers`: (Number, Opcional) Obrigatório se `modal` for "Maritimo FCL".
- `metrosCubicos`: (Number, Opcional) Obrigatório se `modal` for "Maritimo LCL".

**Exemplo (Marítimo FCL):**
```json
{
  "modal": "Maritimo FCL",
  "incoterm": "FOB",
  "freteInternacionalUSD": 7000.00,
  "seguroInternacionalUSD": 465.00,
  "seguroContratado": true,
  "numeroContainers": 2,
  "metrosCubicos": null
}
```

**Exemplo (Aéreo):**
```json
{
  "modal": "Aereo",
  "incoterm": "EXW", 
  "freteInternacionalUSD": 5000.00,
  "seguroInternacionalUSD": 315.00,
  "seguroContratado": true,
  "numeroContainers": null,
  "metrosCubicos": null
}
```

### 1.4. `servicos`
Dados relacionados aos serviços adicionais da operação.

- `numeroDesembaracos`: (Number) Número de desembaraços aduaneiros. Default: 1.
- `servicosTecnicosUSD`: (Number) Valor para serviços técnicos em USD.
- `servicosComerciaisUSD`: (Number) Valor para custos comerciais extras em USD.
- `comissaoAgentePercentual`: (Number) Percentual da comissão do agente de compras.

**Exemplo:**
```json
{
  "numeroDesembaracos": 1,
  "servicosTecnicosUSD": 500.00,
  "servicosComerciaisUSD": 200.00,
  "comissaoAgentePercentual": 3.5
}
```

### 1.5. `despesasOperacionais` (Array de Objetos)
Array populado dinamicamente a partir dos templates em `/config/expense-templates.json`.

- `id`: (String) ID do template de despesa.
- `descricao`: (String) Descrição da despesa.
- `valorSugeridoBRL`: (Number) Valor calculado pela regra padrão do template.
- `valorFinalBRL`: (Number) Valor editável pelo usuário (inicia com o valor sugerido).
- `entraBaseICMS`: (Boolean) Flag que indica se o custo compõe a base de cálculo do ICMS.

**Exemplo:**
```json
[
  {
    "id": "desp_afrmm",
    "descricao": "AFRMM - Marinha Mercante",
    "valorSugeridoBRL": 3052.00,
    "valorFinalBRL": 3052.00,
    "entraBaseICMS": true
  },
  {
    "id": "desp_armazenagem_fcl",
    "descricao": "Armazenamento Portuário",
    "valorSugeridoBRL": 3600.00,
    "valorFinalBRL": 3200.00,
    "entraBaseICMS": true
  }
]
```

### 1.6. `ncmDataCache` (Objeto)
Cache para armazenar dados da NCM obtidos via API para evitar chamadas repetidas.

- **Chave**: Código NCM (String).
- **Valor**: (Objeto) Dados completos da NCM.

**Estrutura do valor:**
```json
{
  "aliquotas": {
    "ii": 16.00,
    "ipi": 15.00,
    "pis": 2.10,
    "cofins": 9.65
  },
  "tratamentosAdministrativos": [
    {
      "orgao": "ANATEL",
      "tipo": "LICENCA_IMPORTACAO",
      "informacoes": "Exigível para homologação de equipamentos de telecomunicações."
    }
  ],
  "unidadeEstatistica": "UN",
  "descricao": "Telefones celulares"
}
```

**Exemplo completo:**
```json
{
  "85171231": {
    "aliquotas": {
      "ii": 16.00,
      "ipi": 15.00,
      "pis": 2.10,
      "cofins": 9.65
    },
    "tratamentosAdministrativos": [
      {
        "orgao": "ANATEL",
        "tipo": "LICENCA_IMPORTACAO",
        "informacoes": "Exigível para homologação de equipamentos de telecomunicações."
      }
    ],
    "unidadeEstatistica": "UN",
    "descricao": "Telefones celulares"
  }
}
```

## 2. Objetos de Resultado (Calculados)

### 2.1. `ResultadoCalculos`
Objeto que contém todos os resultados dos cálculos da cotação.

```json
{
  "valorTotalMercadoriasUSD": 170000.00,
  "pesoTotalKg": 240.00,
  "valorAduaneiroUSD": 177465.00,
  "valorAduaneiroBRL": 968184.25,
  "sequenciasAdicao": [ ... ],
  "totalTributosBRL": 425678.90,
  "totalDespesasOperacionaisBRL": 15420.00,
  "custoTotalProjeto": 1409283.15
}
```

### 2.2. `SequenciaAdicao`
Objeto que representa o agrupamento de adições por NCM para rateio.

```json
{
  "ncm": "85171231",
  "valorSequenciaUSD": 150000.00,
  "percentualSequencia": 0.8824,
  "valorAduaneiroSequenciaBRL": 854050.00,
  "tributos": {
    "ii": 136648.00,
    "ipi": 148804.70,
    "pis": 17935.05,
    "cofins": 82440.83,
    "icms": 251890.45
  }
}
```

## 3. Validações e Regras de Integridade

### 3.1. Validações de Modal vs. Incoterm
- **Aéreo:** Permite apenas `EXW`, `FCA`.
- **Marítimo (FCL/LCL):** Permite `EXW`, `FOB`, `CFR`, `CIF`.

### 3.2. Campos Obrigatórios Condicionais
- Se `modal` = "Maritimo FCL": `numeroContainers` é obrigatório.
- Se `modal` = "Maritimo LCL": `metrosCubicos` é obrigatório.
- Se `modal` = "Aereo": nenhum campo adicional obrigatório.

### 3.3. Validações de Dados
- `ncm`: Deve ter exatamente 8 dígitos numéricos.
- `unidadeDeMedida`: Deve ser validada contra a unidade estatística da NCM.
- `taxaCambio`: Deve ser maior que 0.
- `aliquotaIcms`: Deve estar entre 0 e 100.
- `qtd`, `valorUnitarioUSD`, `pesoUnitarioKg`: Devem ser maiores que 0.

