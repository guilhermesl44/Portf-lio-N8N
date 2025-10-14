# ⚠️ Sistema de Cobrança Automática de Boletos Atrasados

> **Categoria:** Financeiro | **Complexidade:** ⭐⭐⭐⭐ | **Status:** ✅ Em Produção

## 📝 Visão Geral

Criei este workflow para fazer o acompanhamento ativo de boletos em atraso. Ele roda diariamente, identifica todos os boletos com status "ATRASADO", busca o PDF atualizado e envia automaticamente um email de lembrete personalizado para cada cliente com pendência.

O diferencial é que o email não é genérico - ele busca informações do responsável financeiro na planilha de clientes e envia uma mensagem profissional e empática, com o boleto atualizado em anexo.

## 🎯 Por Que Criei Esta Automação

**O problema:**
- Boletos vencem e ficam em aberto sem acompanhamento
- Enviar lembretes manualmente consome muito tempo
- Difícil identificar quais boletos estão atrasados
- Buscar e enviar 2ª via manualmente é trabalhoso
- Perda de receita por falta de follow-up

**A solução:**
Um sistema que:
- Identifica automaticamente boletos atrasados
- Busca informações do responsável financeiro
- Gera 2ª via do boleto atualizada
- Envia email profissional e personalizado
- Roda diariamente sem intervenção manual

## 🏗️ Como Funciona

### Fluxo do Processo

```
1. Schedule Trigger (08:10 AM diariamente)
   ↓
2. Define período de busca (90 dias atrás até 30 dias à frente)
   ↓
3. Autentica no Banco Inter
   ↓
4. Busca lista de cobranças (paginada)
   ↓
5. Para cada cobrança:
   ├─ Filtra apenas status "ATRASADO"
   ├─ Busca detalhes completos
   └─ Se atrasado, continua processamento
   ↓
6. Loop por cobrança atrasada:
   ├─ Formata dados (datas, valores)
   ├─ Busca responsável financeiro no Google Sheets
   ├─ Busca PDF atualizado do boleto
   ├─ Converte para arquivo anexável
   ├─ Envia email personalizado com PDF
   └─ Aguarda e processa próximo
   ↓
7. Verifica se há mais páginas
   ├─ Se sim: aguarda 1 minuto e busca próxima
   └─ Se não: finaliza
```

### Diagrama Visual

![Fluxo de Cobrança](./Imagens/fluxo-cobranca-atrasados.png)

*Nota: Por questões de confidencialidade, o arquivo JSON não está disponível publicamente.*

## ⚙️ Componentes e Integrações

### Tecnologias Utilizadas:
- **N8N:** Plataforma de automação
- **Banco Inter API:** Listagem, detalhes e PDF de cobranças
- **Google Sheets:** Dados de clientes e responsáveis
- **Gmail:** Envio de emails com anexos
- **Schedule:** Execução diária automática

### Nodes do Workflow:
- **Schedule Trigger:** Dispara às 08:10
- **Set Credenciais:** Configurações
- **Range vencimento:** Define período
- **Request Token:** Autenticação OAuth2
- **Edit Fields1:** Controle de paginação
- **Request Listar Cobrancas:** Busca lista paginada
- **Split Out:** Separa cobranças
- **IF1:** Filtra apenas atrasados
- **Request Recuperar Cobranca:** Detalhes completos
- **Loop Over Items:** Processa um por vez
- **Edit Fields3:** Formata datas
- **Edit Fields2:** Organiza dados
- **Get row(s) in sheet:** Busca responsável financeiro
- **Request Recuperar Cobranca1:** Busca PDF
- **Convert to File:** Prepara anexo
- **Send a message:** Envia email
- **Edit Fields:** Controle de páginas
- **IF:** Verifica última página
- **Wait:** Delay entre páginas

## 🔧 Funcionalidades Principais

### 1. Filtro Inteligente de Atrasados

Diferente da sincronização que pega tudo, este workflow filtra apenas boletos com problema:

**Condição de filtro:**
```javascript
IF (situacao === "ATRASADO") {
  // Processa a cobrança
} ELSE {
  // Ignora
}
```

**Por que filtrar?**
- Não envia email para quem está em dia
- Foca apenas em pendências
- Economiza processamento e créditos de email

### 2. Busca de Responsável Financeiro

Antes de enviar email, busca quem é o responsável no Google Sheets:

**Query no Google Sheets:**
```javascript
documentId: "planilha_de_clientes"
sheetName: "Clientes"
filtro: nome === nome_da_cobranca
```

**Retorna:**
```json
{
  "Responsável financeiro": "Maria Silva",
  // outros dados do cliente
}
```

**Uso no email:**
```html
Bom dia, {{ $('Get row(s) in sheet').item.json['Responsável financeiro'] }}
```

**Por que buscar?**
- Personalização do email
- Direciona para pessoa certa
- Aumenta chance de resposta

### 3. Geração de 2ª Via Atualizada
