# Especificação: Fluxo de Cálculo Sequencial

O motor deve seguir esta ordem exata para garantir a precisão dos cálculos. Cada passo depende dos resultados dos passos anteriores.

## Passo 0: Obtenção de Dados Externos

Antes de qualquer cálculo, obter e cachear os dados das NCMs via API do Portal Único.

**Ações:**
1. Para cada NCM única em `state.adicoes`:
   - Verificar se já existe em `state.ncmDataCache`.
   - Se não existir, fazer chamadas às APIs de Classificação Fiscal e Tratamento Administrativo.
   - Armazenar os resultados no cache.

**Dados obtidos por NCM:**
- Alíquotas de II, IPI, PIS, COFINS
- Lista de tratamentos administrativos (órgãos anuentes)
- Unidade estatística
- Descrição do produto

## Passo 1: Cálculos Primários

### 1.1. Valor Total das Mercadorias
```javascript
valorTotalMercadoriasUSD = SUM(adicao.qtd * adicao.valorUnitarioUSD)
```

### 1.2. Peso Total
```javascript
pesoTotalKg = SUM(adicao.qtd * adicao.pesoUnitarioKg)
```

### 1.3. Obtenção do Frete Internacional
O valor de `freteInternacionalUSD` pode ser obtido de duas formas:
- **Manual:** Inserido diretamente pelo usuário.
- **Estimativa:** Calculado via API do Comex Stat (ver `04-api-contracts.md`).

### 1.4. Cálculo do Seguro Internacional
O cálculo do seguro depende do Incoterm e da flag `seguroContratado`:

```javascript
if (!state.transporte.seguroContratado) {
  seguroInternacionalUSD = 0;
} else {
  // Base de cálculo varia conforme Incoterm
  if (['EXW', 'FCA', 'FOB'].includes(state.transporte.incoterm)) {
    baseSeguroUSD = valorTotalMercadoriasUSD + freteInternacionalUSD;
  } else if (['CFR', 'CIF'].includes(state.transporte.incoterm)) {
    baseSeguroUSD = valorTotalMercadoriasUSD;
  }
  
  seguroInternacionalUSD = baseSeguroUSD * taxaSeguro; // taxaSeguro vem de system-parameters.json
}
```

## Passo 2: Cálculo do Valor Aduaneiro (VA)

O Valor Aduaneiro é a base para o cálculo de todos os tributos. Sua composição varia conforme o Incoterm:

### 2.1. Fórmulas por Incoterm

**EXW, FCA, FOB (Frete e Seguro NÃO inclusos no preço):**
```javascript
VA_USD = valorTotalMercadoriasUSD + freteInternacionalUSD + seguroInternacionalUSD;
```

**CFR (Frete incluso, Seguro não incluso):**
```javascript
VA_USD = valorTotalMercadoriasUSD + seguroInternacionalUSD;
```

**CIF (Frete e Seguro inclusos):**
```javascript
VA_USD = valorTotalMercadoriasUSD;
```

### 2.2. Conversão para BRL
```javascript
VA_BRL = VA_USD * state.dadosGerais.taxaCambio;
```

## Passo 3: Geração e Cálculo das Despesas Operacionais

### 3.1. Carregamento dos Templates
1. Carregar os templates de `/config/expense-templates.json`.
2. Para cada template, avaliar a `condicaoAplicacao` com o `state` atual.
3. Se a condição for verdadeira, executar a `regraCalculo` para obter o `valorSugeridoBRL`.

### 3.2. População do Array de Despesas
```javascript
state.despesasOperacionais = [];

templates.forEach(template => {
  if (avaliarCondicao(template.condicaoAplicacao, state)) {
    const valorSugerido = executarRegra(template.regraCalculo, state);
    
    state.despesasOperacionais.push({
      id: template.id,
      descricao: template.descricao,
      valorSugeridoBRL: valorSugerido,
      valorFinalBRL: valorSugerido, // Inicialmente igual ao sugerido
      entraBaseICMS: template.entraBaseICMS
    });
  }
});
```

### 3.3. Cálculo do Total de Despesas para ICMS
```javascript
totalDespesasParaICMS_BRL = SUM(despesa.valorFinalBRL WHERE despesa.entraBaseICMS === true);
```

## Passo 4: Rateio por Sequência de Adição

### 4.1. Agrupamento por NCM
Agrupar as adições por código NCM para formar as "sequências de adição":

```javascript
sequenciasAdicao = {};

state.adicoes.forEach(adicao => {
  if (!sequenciasAdicao[adicao.ncm]) {
    sequenciasAdicao[adicao.ncm] = {
      ncm: adicao.ncm,
      valorSequenciaUSD: 0,
      itens: []
    };
  }
  
  const valorItem = adicao.qtd * adicao.valorUnitarioUSD;
  sequenciasAdicao[adicao.ncm].valorSequenciaUSD += valorItem;
  sequenciasAdicao[adicao.ncm].itens.push(adicao);
});
```

### 4.2. Cálculo dos Percentuais e Rateios
Para cada sequência:

```javascript
Object.values(sequenciasAdicao).forEach(sequencia => {
  // Percentual da sequência no total
  sequencia.percentualSequencia = sequencia.valorSequenciaUSD / valorTotalMercadoriasUSD;
  
  // Rateio do Valor Aduaneiro
  sequencia.valorAduaneiroSequenciaBRL = VA_BRL * sequencia.percentualSequencia;
  
  // Rateio das Despesas para ICMS
  sequencia.despesasSequenciaBRL = totalDespesasParaICMS_BRL * sequencia.percentualSequencia;
});
```

## Passo 5: Cálculo dos Tributos (por Sequência)

Para cada sequência de adição, calcular os tributos usando as alíquotas da NCM:

```javascript
Object.values(sequenciasAdicao).forEach(sequencia => {
  const ncmData = state.ncmDataCache[sequencia.ncm];
  const aliquotas = ncmData.aliquotas;
  
  // Imposto de Importação
  sequencia.tributos.ii = sequencia.valorAduaneiroSequenciaBRL * (aliquotas.ii / 100);
  
  // IPI (base: VA + II)
  const baseIPI = sequencia.valorAduaneiroSequenciaBRL + sequencia.tributos.ii;
  sequencia.tributos.ipi = baseIPI * (aliquotas.ipi / 100);
  
  // PIS (base: VA)
  sequencia.tributos.pis = sequencia.valorAduaneiroSequenciaBRL * (aliquotas.pis / 100);
  
  // COFINS (base: VA)
  sequencia.tributos.cofins = sequencia.valorAduaneiroSequenciaBRL * (aliquotas.cofins / 100);
  
  // ICMS (base: VA + II + IPI + PIS + COFINS + Despesas rateadas)
  const baseICMS = sequencia.valorAduaneiroSequenciaBRL + 
                   sequencia.tributos.ii + 
                   sequencia.tributos.ipi + 
                   sequencia.tributos.pis + 
                   sequencia.tributos.cofins + 
                   sequencia.despesasSequenciaBRL;
  
  const aliquotaICMSDecimal = state.dadosGerais.aliquotaIcms / 100;
  
  // Cálculo "por dentro" do ICMS
  sequencia.tributos.icms = (baseICMS * aliquotaICMSDecimal) / (1 - aliquotaICMSDecimal);
});
```

## Passo 6: Consolidação do Custo Final

### 6.1. Totalizações
```javascript
// Custo da mercadoria em BRL (sem tributos e despesas)
custoMercadoriaBRL = valorTotalMercadoriasUSD * state.dadosGerais.taxaCambio;

// Total de tributos de todas as sequências
totalTributosBRL = 0;
Object.values(sequenciasAdicao).forEach(sequencia => {
  totalTributosBRL += sequencia.tributos.ii + 
                      sequencia.tributos.ipi + 
                      sequencia.tributos.pis + 
                      sequencia.tributos.cofins + 
                      sequencia.tributos.icms;
});

// Total de despesas operacionais (todas, incluindo as que não entram na base de ICMS)
totalDespesasOperacionaisBRL = SUM(despesa.valorFinalBRL) // de todas as despesas
```

### 6.2. Custo Total Final
```javascript
custoTotalProjeto = custoMercadoriaBRL + totalTributosBRL + totalDespesasOperacionaisBRL;
```

## Passo 7: Cálculo do Custo Unitário (Opcional)

Para apresentação ao usuário, calcular o custo unitário considerando os regimes tributários:

### 7.1. Regime Lucro Real (com créditos)
```javascript
// Créditos de PIS, COFINS e IPI
creditoPIS = totalTributosBRL_PIS;
creditoCOFINS = totalTributosBRL_COFINS;
creditoIPI = totalTributosBRL_IPI;

custoUnitarioLucroReal = (custoTotalProjeto - creditoPIS - creditoCOFINS - creditoIPI) / quantidadeTotalItens;
```

### 7.2. Regime Presumido/Simples (sem créditos)
```javascript
custoUnitarioPresumido = custoTotalProjeto / quantidadeTotalItens;
```

## Observações Importantes

1. **Ordem Crítica:** Os passos devem ser executados na ordem especificada, pois cada um depende dos resultados dos anteriores.

2. **Reatividade:** Qualquer mudança nos dados de entrada deve disparar o recálculo a partir do passo apropriado.

3. **Precisão:** Usar aritmética de ponto flutuante com precisão adequada (recomendado: 2 casas decimais para valores monetários).

4. **Validação:** Validar os dados de entrada antes de iniciar os cálculos para evitar erros de execução.

5. **Cache:** Manter o cache de NCM atualizado para evitar chamadas desnecessárias às APIs externas.

