# Especificação: Regras de Negócio Condicionais

Este documento detalha todas as regras condicionais que governam os cálculos da cotação. Estas regras determinam como os valores são calculados com base nas escolhas do usuário.

## 1. Regras de Validação: Modal vs. Incoterm

### 1.1. Combinações Válidas
O sistema deve validar e permitir apenas as seguintes combinações:

| Modal | Incoterms Válidos | Campos Obrigatórios Adicionais |
|-------|-------------------|--------------------------------|
| **Marítimo FCL** | `EXW`, `FOB`, `CFR`, `CIF` | `numeroContainers` |
| **Marítimo LCL** | `EXW`, `FOB`, `CFR`, `CIF` | `metrosCubicos` |
| **Aéreo** | `EXW`, `FCA` | Nenhum |

### 1.2. Implementação da Validação
```javascript
function validarModalIncoterm(modal, incoterm) {
  const combinacoesValidas = {
    'Maritimo FCL': ['EXW', 'FOB', 'CFR', 'CIF'],
    'Maritimo LCL': ['EXW', 'FOB', 'CFR', 'CIF'],
    'Aereo': ['EXW', 'FCA']
  };
  
  return combinacoesValidas[modal]?.includes(incoterm) || false;
}
```

## 2. Regras de Cálculo do Seguro Internacional

### 2.1. Lógica Principal
O cálculo do seguro depende de duas condições: se o usuário optou por contratar seguro e qual o Incoterm negociado.

```javascript
function calcularSeguroInternacional(state) {
  // Se usuário optou por não contratar seguro
  if (!state.transporte.seguroContratado) {
    return 0;
  }
  
  let baseSeguroUSD = 0;
  
  // Base de cálculo varia conforme Incoterm
  if (['EXW', 'FCA', 'FOB'].includes(state.transporte.incoterm)) {
    // Frete e seguro NÃO estão inclusos no preço
    baseSeguroUSD = state.valorTotalMercadoriasUSD + state.transporte.freteInternacionalUSD;
  } else if (['CFR', 'CIF'].includes(state.transporte.incoterm)) {
    // Frete já está incluso no preço (CFR) ou frete e seguro estão inclusos (CIF)
    baseSeguroUSD = state.valorTotalMercadoriasUSD;
  }
  
  const taxaSeguro = carregarParametro('taxaSeguro'); // Default: 0.003 (0.30%)
  return baseSeguroUSD * taxaSeguro;
}
```

### 2.2. Casos Especiais

**Incoterm CIF:**
- Tecnicamente, o seguro já está incluso no valor da mercadoria.
- O sistema pode calcular um valor de seguro para referência, mas na prática, para o cálculo do Valor Aduaneiro, este valor será ignorado.

## 3. Regras de Cálculo do Valor Aduaneiro

### 3.1. Fórmulas por Incoterm
O Valor Aduaneiro é a base para todos os tributos e varia conforme o Incoterm:

```javascript
function calcularValorAduaneiro(state) {
  const vm = state.valorTotalMercadoriasUSD;
  const frete = state.transporte.freteInternacionalUSD;
  const seguro = state.transporte.seguroInternacionalUSD;
  
  let vaUSD = 0;
  
  switch (state.transporte.incoterm) {
    case 'EXW':
    case 'FCA':
    case 'FOB':
      // Frete e seguro NÃO inclusos - devem ser adicionados
      vaUSD = vm + frete + seguro;
      break;
      
    case 'CFR':
      // Frete incluso, seguro não incluso
      vaUSD = vm + seguro;
      break;
      
    case 'CIF':
      // Frete e seguro inclusos - nada a adicionar
      vaUSD = vm;
      break;
      
    default:
      throw new Error(`Incoterm inválido: ${state.transporte.incoterm}`);
  }
  
  return vaUSD * state.dadosGerais.taxaCambio;
}
```

### 3.2. Tabela de Referência Rápida

| Incoterm | Frete no VA? | Seguro no VA? | Fórmula |
|----------|--------------|---------------|---------|
| **EXW** | ✅ Sim | ✅ Sim | VM + Frete + Seguro |
| **FCA** | ✅ Sim | ✅ Sim | VM + Frete + Seguro |
| **FOB** | ✅ Sim | ✅ Sim | VM + Frete + Seguro |
| **CFR** | ❌ Não (já incluso) | ✅ Sim | VM + Seguro |
| **CIF** | ❌ Não (já incluso) | ❌ Não (já incluso) | VM |

## 4. Regras de Aplicação de Despesas Operacionais

### 4.1. Lógica de Filtro por Modal
As despesas operacionais são aplicadas condicionalmente com base no modal de transporte:

```javascript
function aplicarDespesasPorModal(modal) {
  const despesasPorModal = {
    'Maritimo FCL': [
      'desp_taxa_siscomex',
      'desp_afrmm',
      'desp_armazenagem_fcl',
      'desp_coleta_fcl',
      'desp_desembaraco_aduaneiro',
      'desp_liberacao_bl'
    ],
    'Maritimo LCL': [
      'desp_taxa_siscomex',
      'desp_afrmm',
      'desp_custos_lcl',
      'desp_desembaraco_aduaneiro'
    ],
    'Aereo': [
      'desp_taxa_siscomex',
      'desp_coleta_aereo',
      'desp_custos_air',
      'desp_desembaraco_aduaneiro'
    ]
  };
  
  return despesasPorModal[modal] || [];
}
```

### 4.2. Despesas Comuns a Todos os Modais
Algumas despesas se aplicam independentemente do modal:
- `desp_taxa_siscomex`: Taxa Siscomex (sempre aplicável)
- `desp_desembaraco_aduaneiro`: Desembaraço Aduaneiro (sempre aplicável)
- `desp_comissao_agente`: Comissão do Agente (se percentual > 0)
- `desp_servicos_tecnicos`: Serviços Técnicos (se valor > 0)
- `desp_custos_comerciais`: Custos Comerciais (se valor > 0)

### 4.3. Despesas Específicas por Modal

**Marítimo FCL:**
- `desp_afrmm`: AFRMM (8% sobre o frete)
- `desp_armazenagem_fcl`: R$ 1.800 por contêiner
- `desp_coleta_fcl`: R$ 4.200 por contêiner
- `desp_liberacao_bl`: R$ 2.100 por desembaraço

**Marítimo LCL:**
- `desp_afrmm`: AFRMM (8% sobre o frete)
- `desp_custos_lcl`: R$ 5.600 por metro cúbico

**Aéreo:**
- `desp_coleta_aereo`: R$ 2,00 por kg (coleta/movimentação)
- `desp_custos_air`: R$ 20,00 por kg (custos aéreos)

## 5. Regras de Composição da Base de ICMS

### 5.1. Despesas que Entram na Base de ICMS
Nem todas as despesas compõem a base de cálculo do ICMS:

**ENTRAM na base de ICMS:**
- Valor Aduaneiro (sempre)
- II, IPI, PIS, COFINS (sempre)
- AFRMM
- Taxa Siscomex
- Armazenagem
- Coleta/Movimentação
- Custos LCL/Aéreos
- Desembaraço Aduaneiro
- Liberação de BL

**NÃO ENTRAM na base de ICMS:**
- Comissão do Agente
- Serviços Técnicos
- Custos Comerciais Extras

### 5.2. Implementação da Regra
```javascript
function calcularBaseICMS(sequencia, despesasRateadas) {
  return sequencia.valorAduaneiroSequenciaBRL +
         sequencia.tributos.ii +
         sequencia.tributos.ipi +
         sequencia.tributos.pis +
         sequencia.tributos.cofins +
         despesasRateadas; // Apenas despesas com entraBaseICMS = true
}
```

## 6. Regras de Estimativa de Frete

### 6.1. Frete Marítimo (FCL)
Baseado no peso estimado de um contêiner:

```javascript
function estimarFreteMaritimo(custoKgFOB) {
  const pesoContainerKg = carregarParametro('fatorPesoContainer'); // 22.000 kg
  const fatorFrete = carregarParametro('fatorFreteMaritimo'); // 0.15 (15%)
  
  const valorFOBContainer = custoKgFOB * pesoContainerKg;
  return valorFOBContainer * fatorFrete;
}
```

### 6.2. Frete Aéreo
Baseado no peso real da carga:

```javascript
function estimarFreteAereo(custoKgFOB, pesoTotalKg) {
  const fatorFrete = carregarParametro('fatorFreteAereo'); // 0.25 (25%)
  
  return (custoKgFOB * pesoTotalKg) * fatorFrete;
}
```

## 7. Regras de Rateio Proporcional

### 7.1. Critério de Rateio
Todos os custos (Valor Aduaneiro, Despesas para ICMS) são rateados proporcionalmente entre as sequências de adição com base no valor FOB:

```javascript
function calcularPercentualSequencia(valorSequenciaUSD, valorTotalMercadoriasUSD) {
  return valorSequenciaUSD / valorTotalMercadoriasUSD;
}
```

### 7.2. Aplicação do Rateio
```javascript
function ratearCustos(sequencia, percentual, valorTotalRatear) {
  return valorTotalRatear * percentual;
}
```

## 8. Regras de Validação de Dados

### 8.1. Validações Obrigatórias
```javascript
function validarDadosEntrada(state) {
  const erros = [];
  
  // Validar modal vs incoterm
  if (!validarModalIncoterm(state.transporte.modal, state.transporte.incoterm)) {
    erros.push('Combinação de Modal e Incoterm inválida');
  }
  
  // Validar campos condicionais
  if (state.transporte.modal === 'Maritimo FCL' && !state.transporte.numeroContainers) {
    erros.push('Número de contêineres é obrigatório para FCL');
  }
  
  if (state.transporte.modal === 'Maritimo LCL' && !state.transporte.metrosCubicos) {
    erros.push('Metros cúbicos é obrigatório para LCL');
  }
  
  // Validar NCMs
  state.adicoes.forEach(adicao => {
    if (!/^\d{8}$/.test(adicao.ncm)) {
      erros.push(`NCM inválida: ${adicao.ncm}`);
    }
  });
  
  return erros;
}
```

## 9. Regras de Arredondamento

### 9.1. Precisão Monetária
Todos os valores monetários devem ser arredondados para 2 casas decimais:

```javascript
function arredondarMonetario(valor) {
  return Math.round(valor * 100) / 100;
}
```

### 9.2. Precisão de Percentuais
Percentuais de rateio devem manter 4 casas decimais para precisão:

```javascript
function arredondarPercentual(valor) {
  return Math.round(valor * 10000) / 10000;
}
```

## 10. Regras de Tratamento de Erros

### 10.1. Dados de NCM Indisponíveis
Se a API do Portal Único não retornar dados para uma NCM:
- Usar alíquotas padrão (II: 10%, IPI: 5%, PIS: 1.65%, COFINS: 7.6%)
- Registrar um aviso para o usuário
- Continuar o processamento

### 10.2. APIs Externas Indisponíveis
Se as APIs externas estiverem indisponíveis:
- Para NCM: usar dados em cache ou alíquotas padrão
- Para frete: permitir apenas inserção manual
- Informar o usuário sobre a limitação

