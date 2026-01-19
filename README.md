# Script de Ajuste de Inconsistências

## Descrição

Script Python para corrigir inconsistências de dados entre os bancos `accounts`, `gestao` e `contrato`, utilizando `accounts.users` como fonte da verdade.

## Funcionalidades

- ✅ Sincroniza dados de CPF, nome, email e telefone
- ✅ Atualiza `gestao.tb_usuario` com dados corretos
- ✅ Atualiza `contrato.usuario` com dados corretos
- ✅ Desvincula segurados com CPF divergente
- ✅ Gera relatório detalhado de execução
- ✅ Confirmação antes de executar UPDATEs
- ✅ Suporte a limite de registros para testes

## Pré-requisitos

1. **Python 3.8+** instalado
2. **Dependências:**
   ```bash
   pip install psycopg2-binary python-dotenv openpyxl
   ```
3. **Relatório de análise:** Execute primeiro o script `analise-inconsistencia/main.py` para gerar o relatório de emails duplicados
4. **Túnel SSH configurado:** Acesso aos bancos de dados via SSH

## Configuração

### 1. Criar arquivo `.env.staging`

Crie o arquivo `.env.staging` na raiz do projeto com as seguintes variáveis:

```bash
# Nome do Cliente
NOME_CLIENTE=STAGING

# Host único para todos os bancos
DB_HOST=localhost

# Banco GESTAO (gestao-usuarios-api)
DB_GESTAO_NAME=gestao_usuarios
DB_GESTAO_USER=postgres
DB_GESTAO_PASS=senha123

# Banco CONTRATO (contrato-api)
DB_CONTRATO_NAME=contrato
DB_CONTRATO_USER=postgres
DB_CONTRATO_PASS=senha123

# Banco PESSOA (pessoa-api)
DB_PESSOA_NAME=pessoa
DB_PESSOA_USER=postgres
DB_PESSOA_PASS=senha123

# Banco ACCOUNTS (via dblink)
URL_ACCOUNTS=servidor.exemplo.com
DB_ACCOUNTS_NAME_USER=accounts
DB_ACCOUNTS_PASS=senha123

# Configuração SSH (Túnel)
SSH_HOST=servidor.exemplo.com
SSH_USER=usuario
SSH_PORT=22
SSH_PKEY_PATH=/caminho/para/chave.pem  # Recomendado
# SSH_PASSWORD=senha_ssh  # Alternativa (não recomendado)
SSH_REMOTE_DB_HOST=localhost
SSH_REMOTE_DB_PORT=5432
SSH_LOCAL_PORT=5435

# Limite de registros para testes (0 = todos)
LIMITE_REGISTROS=10
```

### 2. Ajustar limite de registros

- **Para validação inicial:** `LIMITE_REGISTROS=1` (ativa **MODO DEBUG** com análise detalhada)
- **Para testes:** `LIMITE_REGISTROS=10` (processa apenas 10 registros)
- **Para produção:** `LIMITE_REGISTROS=0` (processa todos os registros)

> 💡 **MODO DEBUG:** Quando `LIMITE_REGISTROS=1`, o script entra em modo interativo detalhado, mostrando todos os dados, divergências campo a campo, e validando o resultado após o UPDATE. Perfeito para validar o script antes de executar em massa!

## Uso

### 1. Executar o script

```bash
cd ajuste-inconsistencia
python main.py
```

### 2. Selecionar o cliente

O script apresentará um menu com os clientes configurados (baseado nos arquivos `.env.*`):

```
 🔧  SELEÇÃO DE CLIENTE - AJUSTE DE INCONSISTÊNCIAS
============================================================
  1. STAGING
  0. Sair
============================================================

➤ Selecione o cliente (número): 1
```

### 3. Confirmação de execução

Após análise, o script exibirá um resumo das alterações:

```
RESUMO DAS ALTERAÇÕES A SEREM EXECUTADAS
============================================================
Registros processados: 10
  - Updates em gestao.tb_usuario: 5
  - Updates em contrato.usuario: 3
  - Desvinculações em segurado: 2
  - Registros ignorados: 0
  - Erros: 0
============================================================

⚠️  ATENÇÃO: As alterações serão executadas DIRETAMENTE no banco de dados!

Confirmar execução dos UPDATEs? (S/N):
```

### 4. Relatório de execução

Após a execução, um arquivo Excel será gerado:

```
ajuste_executado_<cliente>.xlsx
```

Com as seguintes abas:
- **0-Resumo:** Estatísticas gerais
- **1-Updates Gestão:** Registros atualizados em `gestao.tb_usuario`
- **2-Updates Contrato:** Registros atualizados em `contrato.usuario`
- **3-Desvinculações:** Segurados desvinculados
- **4-Ignorados:** Registros que foram ignorados (sem CPF, etc)
- **5-Erros:** Erros encontrados durante a execução

## Fluxo de Processamento

```
1. Carregar relatório de emails duplicados
   ↓
2. Para cada UUID:
   ↓
3. Buscar CPF em accounts (fonte da verdade)
   ↓
4. Verificar se CPF existe em segurado
   ↓
5. Se existe: Comparar com gestao.tb_usuario
   ↓
6. Se divergente: Preparar UPDATE
   ↓
7. Comparar com contrato.usuario
   ↓
8. Se divergente: Preparar UPDATE
   ↓
9. Buscar segurados com CPF divergente
   ↓
10. Preparar desvinculação (usuario_id = NULL)
   ↓
11. Confirmar com usuário
   ↓
12. Executar todos os UPDATEs
   ↓
13. Gerar relatório de execução
```

## Critérios de Validação

### Registros são **PROCESSADOS** se:
- ✅ UUID existe em `accounts.users`
- ✅ CPF existe em `accounts.users` (não vazio)
- ✅ CPF existe em `contrato.segurado`

### Registros são **IGNORADOS** se:
- ❌ UUID não encontrado em `accounts`
- ❌ CPF vazio em `accounts`
- ❌ CPF não encontrado em `segurado`

### Campos Sincronizados:

**gestao.tb_usuario:**
- `cpf_cnpj`
- `name`
- `email`
- `phone`

**contrato.usuario:**
- `cpf_cnpj`
- `nome`
- `email`

**contrato.segurado:**
- `usuario_id` (setado para NULL se CPF divergente)

## Segurança

- ⚠️ **ATENÇÃO:** Este script executa UPDATEs DIRETOS no banco de dados
- 🔒 Sempre teste primeiro com `LIMITE_REGISTROS` configurado
- 💾 Não há backup automático (por enquanto)
- ✅ Confirmação obrigatória antes de executar
- 📊 Relatório detalhado de todas as alterações

## Troubleshooting

### Erro: "Relatório não encontrado"
- Execute primeiro o script `analise-inconsistencia/main.py`
- Verifique se o arquivo `relatorio_<cliente>.xlsx` existe

### Erro: "SSH_HOST e SSH_USER são obrigatórios"
- Configure as variáveis SSH no arquivo `.env.staging`

### Erro: "Timeout ao aguardar túnel SSH"
- Verifique credenciais SSH
- Confirme que o servidor SSH está acessível
- Verifique se a porta local não está em uso

### Nenhuma alteração necessária
- ✅ Todos os dados já estão consistentes!
- Nenhum UPDATE será executado

## Logs e Monitoramento

Durante a execução, o script exibe:
- ✅ Status de cada registro processado
- 📊 Contadores em tempo real
- ⚠️ Warnings para registros ignorados
- ❌ Erros detalhados quando ocorrem

## Suporte

Para dúvidas ou problemas:
1. Verifique os logs no console
2. Consulte o relatório de execução gerado
3. Revise as configurações do `.env.staging`

## Status

Ainda em desenvolvimento
Na espera de um abiente de stging adequado para teste em lote