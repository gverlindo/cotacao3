# Projeto de Engenharia: Motor de Cálculo do Módulo de Cotação Verlaz

## 1. Objetivo
Construir o "cérebro" (lógica de negócios e motor de cálculo) para o Módulo de Cotação Verlaz. Este projeto é "headless" (sem interface de usuário) e foca em criar uma base de código robusta, testável e precisa para todos os cálculos de uma cotação de importação.

## 2. Escopo
- **DENTRO DO ESCOPO:**
  - Estruturação dos dados da cotação.
  - Implementação de todo o fluxo de cálculo de tributos e custos.
  - Integração com as APIs do Portal Único Siscomex e Comex Stat.
  - Lógica para carregar e aplicar despesas operacionais a partir de um arquivo de configuração.
  - Implementação de regras de negócio condicionais (Modal vs. Incoterm).

- **FORA DO ESCOPO:**
  - Implementação da interface de usuário (UI/Front-End).
  - Autenticação de usuários.
  - Persistência dos dados em banco de dados (neste momento, o foco é no cálculo em memória).

## 3. Guia de Leitura para Desenvolvedores
Para uma compreensão completa, siga a ordem dos documentos na pasta `/specs`.
1.  **`01-data-structures.md`**: Entenda a "forma" de todos os dados.
2.  **`02-calculation-flow.md`**: Veja a sequência lógica dos cálculos.
3.  **`03-business-rules.md`**: Compreenda como os cálculos mudam com base nas escolhas do usuário.
4.  **`04-api-contracts.md`**: Veja como o sistema interage com o mundo exterior.

Os arquivos na pasta `/config` devem ser carregados pelo sistema no início da execução. A pasta `/tests` contém cenários de entrada e saída para validar a corretude da sua implementação.

## 4. Estrutura do Projeto

```
/projeto-cotacao-verlaz
|
|-- README.md                  <-- Este arquivo
|
|-- /specs                     <-- Especificações técnicas detalhadas
|   |-- 01-data-structures.md  <-- Dicionário completo de dados
|   |-- 02-calculation-flow.md <-- A ordem exata e as fórmulas de cálculo
|   |-- 03-business-rules.md   <-- Lógica condicional (Modal x Incoterm)
|   |-- 04-api-contracts.md    <-- Contratos com as APIs externas
|
|-- /config                    <-- Configurações externas ao código
|   |-- expense-templates.json <-- Tabela de despesas traduzida para JSON
|   |-- system-parameters.json <-- Parâmetros de negócio (taxas, fatores)
|
|-- /tests                     <-- Cenários de teste para validação
    |-- scenario-fob-fcl.json  <-- Caso de teste 1: Marítimo FCL, FOB
    |-- scenario-cfr-lcl.json  <-- Caso de teste 2: Marítimo LCL, CFR
    |-- scenario-exw-aereo.json <-- Caso de teste 3: Aéreo, EXW
```

## 5. Tecnologias Recomendadas
- **Linguagem:** JavaScript/TypeScript ou Python
- **Framework:** Node.js/Express ou Flask/FastAPI
- **Testes:** Jest/Mocha ou pytest
- **Validação:** Joi/Zod ou Pydantic

## 6. Próximos Passos
1. Leia todos os documentos na pasta `/specs` na ordem numérica
2. Implemente as estruturas de dados conforme especificado
3. Desenvolva o motor de cálculo seguindo o fluxo sequencial
4. Integre com as APIs externas conforme os contratos
5. Execute os cenários de teste para validar a implementação
6. Documente qualquer desvio ou adaptação necessária

## 7. Contato
Para dúvidas sobre as regras de negócio ou especificações técnicas, consulte a documentação detalhada nos arquivos de especificação ou entre em contato com a equipe de produto.

