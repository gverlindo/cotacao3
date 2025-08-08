# Especificação: Contratos de API

Este documento detalha como o sistema irá interagir com as APIs externas do Portal Único Siscomex e do Comex Stat para obter dados em tempo real.

## 1. APIs do Portal Único Siscomex

### 1.1. API de Classificação Fiscal (Tratamento Tributário)

**Objetivo:** Obter as alíquotas de tributos federais para uma NCM específica.

- **URL Base:** `https://portalunico.siscomex.gov.br`
- **Endpoint:** `GET /classif/api/publico/nomenclatura/{codigoNcm}/tributos`
- **Método:** GET
- **Autenticação:** Não requerida (endpoint público)

**Parâmetros:**
- `{codigoNcm}`: Código NCM de 8 dígitos (ex: "85171231")

**Exemplo de Chamada:**
```
GET https://portalunico.siscomex.gov.br/classif/api/publico/nomenclatura/85171231/tributos
```

**Resposta Esperada (JSON):**
```json
[
  {
    "tipo": "II",
    "aliquota": 16.00,
    "unidadeMedida": "%",
    "legislacao": "TEC"
  },
  {
    "tipo": "IPI",
    "aliquota": 15.00,
    "unidadeMedida": "%",
    "legislacao": "TIPI"
  },
  {
    "tipo": "PIS_PASEP_IMPORTACAO",
    "aliquota": 2.10,
    "unidadeMedida": "%",
    "legislacao": "Lei 10.865/2004"
  },
  {
    "tipo": "COFINS_IMPORTACAO",
    "aliquota": 9.65,
    "unidadeMedida": "%",
    "legislacao": "Lei 10.865/2004"
  }
]
```

**Mapeamento para o Sistema:**
```javascript
function mapearAliquotas(responseAPI) {
  const aliquotas = {};
  
  responseAPI.forEach(tributo => {
    switch (tributo.tipo) {
      case 'II':
        aliquotas.ii = tributo.aliquota;
        break;
      case 'IPI':
        aliquotas.ipi = tributo.aliquota;
        break;
      case 'PIS_PASEP_IMPORTACAO':
        aliquotas.pis = tributo.aliquota;
        break;
      case 'COFINS_IMPORTACAO':
        aliquotas.cofins = tributo.aliquota;
        break;
    }
  });
  
  return aliquotas;
}
```

### 1.2. API de Tratamento Administrativo

**Objetivo:** Obter informações sobre licenças, anuências e tratamentos administrativos necessários para uma NCM.

- **URL Base:** `https://portalunico.siscomex.gov.br`
- **Endpoint:** `GET /talpco/api/p/tratamento-administrativo/consultar-por-ncm`
- **Método:** GET
- **Autenticação:** Não requerida (endpoint público)

**Parâmetros (Query String):**
- `ncm`: Código NCM de 8 dígitos

**Exemplo de Chamada:**
```
GET https://portalunico.siscomex.gov.br/talpco/api/p/tratamento-administrativo/consultar-por-ncm?ncm=85171231
```

**Resposta Esperada (JSON):**
```json
{
  "total": 2,
  "items": [
    {
      "orgaoAnuente": {
        "codigo": "ANATEL",
        "sigla": "ANATEL",
        "nome": "Agência Nacional de Telecomunicações"
      },
      "tipoProcesso": "LICENCA_IMPORTACAO",
      "informacoesAdicionais": "Exigível para homologação de equipamentos de telecomunicações conforme Resolução nº 242/2000.",
      "situacao": "ATIVO"
    },
    {
      "orgaoAnuente": {
        "codigo": "INMETRO",
        "sigla": "INMETRO", 
        "nome": "Instituto Nacional de Metrologia, Qualidade e Tecnologia"
      },
      "tipoProcesso": "CERTIFICACAO_COMPULSORIA",
      "informacoesAdicionais": "Certificação compulsória para equipamentos eletrônicos.",
      "situacao": "ATIVO"
    }
  ]
}
```

**Mapeamento para o Sistema:**
```javascript
function mapearTratamentosAdministrativos(responseAPI) {
  return responseAPI.items
    .filter(item => item.situacao === 'ATIVO')
    .map(item => ({
      orgao: item.orgaoAnuente.sigla,
      tipo: item.tipoProcesso,
      informacoes: item.informacoesAdicionais
    }));
}
```

### 1.3. API de Simulação de DUIMP (Validação Completa)

**Objetivo:** Validar os cálculos da cotação usando o mesmo motor da Receita Federal.

- **URL Base:** `https://portalunico.siscomex.gov.br`
- **Endpoint:** `POST /duimp/api/v1/duimp/simular`
- **Método:** POST
- **Autenticação:** Requerida (certificado digital)
- **Content-Type:** `application/json`

**Uso Recomendado:** Como uma ação final do usuário ("Validar Cotação com a Receita Federal").

**Payload Exemplo (Simplificado):**
```json
{
  "declaracao": {
    "importador": {
      "cpfCnpj": "12345678000195",
      "nome": "Empresa Importadora Ltda"
    },
    "dadosGerais": {
      "paisOrigem": "CHN",
      "ufDesembaraco": "SC",
      "modalidadeDespacho": "NORMAL"
    },
    "adicoes": [
      {
        "numeroAdicao": 1,
        "ncm": "85171231",
        "unidadeMedida": "UN",
        "quantidade": 1000,
        "valorUnitario": 150.00,
        "pesoLiquido": 200.00
      }
    ],
    "frete": {
      "valor": 7000.00,
      "moeda": "USD"
    },
    "seguro": {
      "valor": 465.00,
      "moeda": "USD"
    }
  }
}
```

**Resposta Esperada:**
A resposta será complexa e incluirá:
- Tributos calculados por adição
- Validações de tratamentos administrativos
- Possíveis erros ou inconsistências
- Valor total da operação

## 2. API do Comex Stat (Estimativa de Frete)

### 2.1. Obtenção de Dados Estatísticos

**Objetivo:** Obter dados históricos de importação para calcular estimativas de frete.

- **URL Base:** `https://api-comexstat.mdic.gov.br`
- **Endpoint:** `GET /general`
- **Método:** GET
- **Autenticação:** Não requerida

**Estratégia de Consulta:**
1. Filtrar por NCM e país de origem
2. Usar período recente (ex: últimos 12 meses)
3. Obter agregados de valor FOB e peso líquido
4. Calcular custo médio por kg

**Parâmetros de Consulta:**
```javascript
const params = {
  filter: JSON.stringify({
    co_ncm: ncm,
    co_pais: codigoPais,
    co_ano: [2023, 2024] // Últimos anos
  }),
  aggregate: JSON.stringify([
    { $group: {
      _id: null,
      totalVlFob: { $sum: "$vl_fob" },
      totalKgLiquido: { $sum: "$kg_liquido" }
    }}
  ])
};
```

**Exemplo de Chamada:**
```
GET https://api-comexstat.mdic.gov.br/general?filter={"co_ncm":"85171231","co_pais":"160"}&aggregate=[{"$group":{"_id":null,"totalVlFob":{"$sum":"$vl_fob"},"totalKgLiquido":{"$sum":"$kg_liquido"}}}]
```

**Resposta Esperada:**
```json
[
  {
    "_id": null,
    "totalVlFob": 15750000.50,
    "totalKgLiquido": 12500.75
  }
]
```

### 2.2. Cálculo da Estimativa de Frete

**Processamento dos Dados:**
```javascript
function calcularEstimativaFrete(responseComexStat, modal, parametros) {
  const dados = responseComexStat[0];
  
  if (!dados || dados.totalKgLiquido === 0) {
    throw new Error('Dados insuficientes para estimativa');
  }
  
  // Custo médio por kg da mercadoria
  const custoKgFOB = dados.totalVlFob / dados.totalKgLiquido;
  
  let freteEstimadoUSD = 0;
  
  if (modal === 'Maritimo FCL') {
    // Frete baseado no peso estimado de um contêiner
    const pesoContainerKg = parametros.fatorPesoContainer; // 22.000 kg
    const valorFOBContainer = custoKgFOB * pesoContainerKg;
    freteEstimadoUSD = valorFOBContainer * parametros.fatorFreteMaritimo; // 15%
    
  } else if (modal === 'Maritimo LCL') {
    // Para LCL, usar uma abordagem similar mas ajustada
    const valorFOBPorM3 = custoKgFOB * 500; // Estimativa: 500kg por m³
    freteEstimadoUSD = valorFOBPorM3 * parametros.fatorFreteMaritimo;
    
  } else if (modal === 'Aereo') {
    // Frete aéreo baseado no peso real
    const pesoTotalKg = calcularPesoTotal(); // Do state atual
    freteEstimadoUSD = (custoKgFOB * pesoTotalKg) * parametros.fatorFreteAereo; // 25%
  }
  
  return {
    freteEstimadoUSD: Math.round(freteEstimadoUSD * 100) / 100,
    custoKgFOB: Math.round(custoKgFOB * 100) / 100,
    fonte: 'ComexStat',
    periodo: 'Últimos 12 meses'
  };
}
```

## 3. Tratamento de Erros e Fallbacks

### 3.1. Estratégias de Fallback

**Para APIs do Portal Único:**
```javascript
async function obterDadosNCM(ncm) {
  try {
    // Tentar obter dados das APIs
    const [aliquotas, tratamentos] = await Promise.all([
      obterAliquotas(ncm),
      obterTratamentosAdministrativos(ncm)
    ]);
    
    return { aliquotas, tratamentos };
    
  } catch (error) {
    console.warn(`Erro ao obter dados da NCM ${ncm}:`, error);
    
    // Usar dados padrão
    return {
      aliquotas: {
        ii: 10.0,  // Alíquota padrão
        ipi: 5.0,
        pis: 1.65,
        cofins: 7.6
      },
      tratamentos: [],
      fonte: 'Dados padrão (API indisponível)'
    };
  }
}
```

**Para API do Comex Stat:**
```javascript
async function obterEstimativaFrete(ncm, paisOrigem, modal) {
  try {
    const dados = await consultarComexStat(ncm, paisOrigem);
    return calcularEstimativaFrete(dados, modal);
    
  } catch (error) {
    console.warn('Erro ao obter estimativa de frete:', error);
    
    // Retornar indicação de que estimativa não está disponível
    return {
      erro: 'Estimativa indisponível',
      mensagem: 'Insira o valor do frete manualmente',
      fonte: 'Manual'
    };
  }
}
```

### 3.2. Timeout e Retry

**Configuração de Timeout:**
```javascript
const API_CONFIG = {
  timeout: 10000, // 10 segundos
  retries: 3,
  retryDelay: 1000 // 1 segundo entre tentativas
};
```

**Implementação de Retry:**
```javascript
async function chamarAPIComRetry(url, config = {}) {
  let ultimoErro;
  
  for (let tentativa = 1; tentativa <= API_CONFIG.retries; tentativa++) {
    try {
      const response = await fetch(url, {
        ...config,
        timeout: API_CONFIG.timeout
      });
      
      if (response.ok) {
        return await response.json();
      }
      
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
      
    } catch (error) {
      ultimoErro = error;
      
      if (tentativa < API_CONFIG.retries) {
        await new Promise(resolve => 
          setTimeout(resolve, API_CONFIG.retryDelay * tentativa)
        );
      }
    }
  }
  
  throw ultimoErro;
}
```

## 4. Cache e Performance

### 4.1. Cache de NCM
```javascript
class NCMCache {
  constructor() {
    this.cache = new Map();
    this.ttl = 24 * 60 * 60 * 1000; // 24 horas
  }
  
  async obterDados(ncm) {
    const cacheKey = ncm;
    const cached = this.cache.get(cacheKey);
    
    if (cached && (Date.now() - cached.timestamp) < this.ttl) {
      return cached.dados;
    }
    
    const dados = await obterDadosNCM(ncm);
    this.cache.set(cacheKey, {
      dados,
      timestamp: Date.now()
    });
    
    return dados;
  }
}
```

### 4.2. Debounce para Estimativas
```javascript
function debounce(func, delay) {
  let timeoutId;
  return function (...args) {
    clearTimeout(timeoutId);
    timeoutId = setTimeout(() => func.apply(this, args), delay);
  };
}

// Usar debounce para evitar muitas chamadas durante a digitação
const obterEstimativaDebounced = debounce(obterEstimativaFrete, 500);
```

## 5. Monitoramento e Logs

### 5.1. Log de Chamadas de API
```javascript
function logChamadaAPI(endpoint, parametros, resultado, tempoResposta) {
  console.log({
    timestamp: new Date().toISOString(),
    endpoint,
    parametros,
    sucesso: !resultado.erro,
    tempoResposta,
    fonte: resultado.fonte || 'API'
  });
}
```

### 5.2. Métricas de Performance
- Tempo de resposta das APIs
- Taxa de sucesso/erro
- Uso do cache
- Frequência de fallbacks

