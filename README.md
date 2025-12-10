# PROMPT COMPLETO PARA CLAUDE CODE - ANÁLISE E CORREÇÃO DO SCRAPER

## CONTEXTO E OBJETIVO PRINCIPAL
Você está trabalhando em um projeto de scraper que baixa laudos médicos em PDF de um portal web (WorkLab). O scraper atual está apresentando problemas críticos de funcionamento que precisam ser resolvidos de forma sistemática e completa.

## PROBLEMA ATUAL
O scraper está falhando em:
1. Aplicar corretamente os filtros de data e status
2. Obter a lista correta de nomes/PDFs após navegar pela paginação
3. Baixar TODOS os PDFs disponíveis sem repetições ou omissões
4. Lidar adequadamente com o JavaScript dinâmico da página

## INSTRUÇÕES CRÍTICAS DE ABORDAGEM

### 🔴 REGRAS OBRIGATÓRIAS (NÃO NEGOCIÁVEIS)

1. **NÃO PARE NO PRIMEIRO PROBLEMA**: 
   - Identifique TODOS os problemas antes de começar a resolver
   - Liste cada problema encontrado com sua causa raiz
   - Crie um mapa completo de todos os pontos de falha

2. **NÃO PARE NA PRIMEIRA SOLUÇÃO**:
   - Para cada problema, liste NO MÍNIMO 3 abordagens diferentes
   - Teste CADA solução de forma isolada
   - Documente os resultados de cada teste
   - Selecione a melhor solução baseada em evidências concretas

3. **ANÁLISE COMPLETA DO PROJETO**:
   - Examine TODOS os arquivos do projeto sem exceção:
     * Todos os arquivos .js
     * Todos os arquivos .json (package.json, config, etc)
     * Todos os arquivos .md (README, documentação)
     * Histórico do Git (commits recentes, branches)
     * Arquivos de configuração (.env, .gitignore, etc)
     * Dependências e versões no package.json
   - Mapeie as relações entre os arquivos
   - Identifique padrões e anti-padrões no código existente

## ANÁLISE TÉCNICA OBRIGATÓRIA

### 1. ESTRUTURA DA PÁGINA (baseada no HTML fornecido)

**Características críticas da página:**
- Framework: Vue.js (VueTables component)
- Rendering: Híbrido (server-side + client-side)
- Paginação: Controlada por JavaScript
- Conteúdo dinâmico: Carregado após interações

**Elementos-chave identificados no DOM:**
```javascript
// FILTROS
const SELECTORS = {
  dataInicial: '#app > div > main > div.container > div:nth-child(2) > div > div:nth-child(1) > input',
  dataFinal: '#app > div > main > div.container > div:nth-child(2) > div > div:nth-child(2) > input',
  status: '#app > div > main > div.container > div:nth-child(2) > div > div:nth-child(3) > select',
  botaoFiltrar: '#app > div > main > div.container > div:nth-child(2) > div > div:nth-child(5) > a',
  
  // TABELA E PAGINAÇÃO
  tabela: 'table.VueTables__table tbody',
  linhasTabela: 'table.VueTables__table tbody tr',
  paginacao: '.VuePagination__pagination',
  paginaAtiva: '.VuePagination__pagination-item.active',
  proximaPagina: '.VuePagination__pagination-item-next-page:not(.disabled) a',
  
  // DOWNLOADS
  linkDownloadPDF: 'a[href*="download=1"]',
  nomePaciente: 'td:nth-child(3)', // 3ª coluna da tabela
  codigoPaciente: 'td:nth-child(2)', // 2ª coluna da tabela
};
```

### 2. FLUXO ESPERADO DO SCRAPER

Documente o fluxo atual e compare com o fluxo ideal:
```
FLUXO IDEAL:
1. Fazer login → 2. Aplicar filtros (data + status) → 3. Aguardar carregamento 
→ 4. Extrair lista de PDFs da página atual → 5. Baixar PDFs 
→ 6. Verificar se há próxima página → 7. Se sim, clicar e repetir 4-6 
→ 8. Validar: todos baixados, sem duplicatas

PONTOS DE FALHA POSSÍVEIS:
- [ ] Login não está completando
- [ ] Filtros não estão sendo aplicados
- [ ] Não está aguardando o JavaScript carregar os dados
- [ ] Seletores estão incorretos/mudaram
- [ ] Paginação não está funcionando
- [ ] Lista de PDFs está sendo duplicada entre páginas
- [ ] Downloads estão falhando silenciosamente
```

## TAREFAS ESPECÍFICAS

### ETAPA 1: DIAGNÓSTICO COMPLETO (obrigatório antes de qualquer correção)

1. **Analise o código atual do scraper**:
```bash
   # Liste todos os arquivos relacionados
   find . -type f \( -name "*.js" -o -name "*.json" -o -name "*.md" \) -not -path "*/node_modules/*"
   
   # Examine cada arquivo identificado
```

2. **Identifique problemas no código**:
   - Uso incorreto de seletores
   - Falta de waits para elementos dinâmicos
   - Lógica de paginação incorreta
   - Controle de estado inadequado (lista de PDFs já baixados)
   - Tratamento de erros insuficiente

3. **Teste isolado de cada componente**:
   - Login funciona? (teste isolado)
   - Filtros aplicam? (teste isolado)
   - Dados carregam após filtro? (teste isolado)
   - Paginação funciona? (teste isolado)
   - Download funciona? (teste isolado)

### ETAPA 2: CORREÇÃO DOS FILTROS

**Implementação dos filtros com múltiplas estratégias:**
```javascript
// ESTRATÉGIA 1: Fill + Click simples
await page.fill(SELECTORS.dataInicial, dataInicial);
await page.fill(SELECTORS.dataFinal, dataFinal);
await page.selectOption(SELECTORS.status, 'resultado_disponivel');
await page.click(SELECTORS.botaoFiltrar);

// ESTRATÉGIA 2: Evaluate (bypass Playwright)
await page.evaluate((data) => {
  document.querySelector(data.seletorInicial).value = data.dataInicial;
  document.querySelector(data.seletorFinal).value = data.dataFinal;
  document.querySelector(data.seletorStatus).value = 'resultado_disponivel';
}, { seletorInicial, seletorFinal, seletorStatus, dataInicial, dataFinal });

// ESTRATÉGIA 3: Dispatch Events (simular usuário real)
await page.dispatchEvent(SELECTORS.dataInicial, 'input');
await page.dispatchEvent(SELECTORS.dataInicial, 'change');
```

**TESTE CADA ESTRATÉGIA** e documente qual funciona melhor.

### ETAPA 3: CORREÇÃO DA PAGINAÇÃO E COLETA

**Problemas comuns a verificar:**

1. **Wait inadequado após clicar na paginação**:
```javascript
// ERRADO (possível problema atual)
await page.click('.next-page');
// Imediatamente tenta extrair dados - FALHA!

// CORRETO - múltiplas estratégias de wait:
await page.click('.next-page');

// Estratégia 1: Wait for network idle
await page.waitForLoadState('networkidle');

// Estratégia 2: Wait for URL change
const urlAntes = page.url();
await page.click('.next-page');
await page.waitForFunction((urlAnterior) => window.location.href !== urlAnterior, urlAntes);

// Estratégia 3: Wait for element staleness
const elementoAntigo = await page.locator('tbody tr').first();
await page.click('.next-page');
await elementoAntigo.waitFor({ state: 'detached' });

// Estratégia 4: Wait for specific content change
const textoAntes = await page.locator('.VuePagination__count').textContent();
await page.click('.next-page');
await page.waitForFunction((textoAnterior) => {
  const textoAtual = document.querySelector('.VuePagination__count').textContent;
  return textoAtual !== textoAnterior;
}, textoAntes);
```

2. **Controle de duplicatas**:
```javascript
// Implementar SET para rastrear PDFs já processados
const pdfsProcessados = new Set();

function extrairPDFs(page) {
  const pdfs = await page.$$eval(SELECTORS.linhasTabela, rows => {
    return rows.map(row => ({
      codigo: row.querySelector('td:nth-child(2)').textContent.trim(),
      nome: row.querySelector('td:nth-child(3)').textContent.trim(),
      linkDownload: row.querySelector('a[href*="download=1"]')?.href,
      identificadorUnico: row.querySelector('td:nth-child(2)').textContent.trim() // Usar código como ID único
    }));
  });
  
  // Filtrar apenas novos
  const pdfsNovos = pdfs.filter(pdf => !pdfsProcessados.has(pdf.identificadorUnico));
  
  // Adicionar ao set
  pdfsNovos.forEach(pdf => pdfsProcessados.add(pdf.identificadorUnico));
  
  return pdfsNovos;
}
```

### ETAPA 4: VALIDAÇÃO E LOGGING

Implemente logging detalhado em CADA etapa:
```javascript
const logger = {
  info: (msg, data) => console.log(`[INFO] ${new Date().toISOString()} - ${msg}`, data || ''),
  error: (msg, error) => console.error(`[ERROR] ${new Date().toISOString()} - ${msg}`, error),
  debug: (msg, data) => console.log(`[DEBUG] ${new Date().toISOString()} - ${msg}`, JSON.stringify(data, null, 2))
};

// Exemplo de uso:
logger.info('Iniciando aplicação de filtros');
logger.debug('Valores dos filtros', { dataInicial, dataFinal, status });
logger.info(`Página ${numeroPagina}: encontrados ${pdfs.length} PDFs`);
logger.info(`Total de PDFs únicos coletados: ${pdfsProcessados.size}`);
```

## CHECKLIST DE VALIDAÇÃO FINAL

Antes de considerar o trabalho concluído, valide:

- [ ] Filtros aplicam corretamente e dados carregam
- [ ] Todas as páginas são navegadas sem pular nenhuma
- [ ] Lista de PDFs não contém duplicatas
- [ ] Todos os PDFs disponíveis foram baixados
- [ ] Código tem tratamento de erros robusto
- [ ] Logs permitem debug fácil de problemas
- [ ] Código segue boas práticas do Playwright
- [ ] Documentação atualizada com mudanças

## RECURSOS TÉCNICOS

### Consulte a documentação oficial:

1. **Playwright:**
   - Waits: https://playwright.dev/docs/actionability
   - Selectors: https://playwright.dev/docs/selectors
   - Network: https://playwright.dev/docs/network

2. **Chromium DevTools:**
   - Use `page.pause()` para debug interativo
   - Use `page.screenshot()` em pontos críticos

### Comandos úteis para análise:
```bash
# Ver estrutura do projeto
tree -L 3 -I 'node_modules'

# Ver commits recentes
git log --oneline -10

# Ver dependências
cat package.json | grep -A 20 "dependencies"

# Buscar padrões no código
grep -r "page.click" --include="*.js"
grep -r "waitFor" --include="*.js"
```

## ENTREGA ESPERADA

1. **Relatório de diagnóstico** listando TODOS os problemas encontrados
2. **Análise de soluções** com no mínimo 3 abordagens para cada problema
3. **Código corrigido** com comentários explicando mudanças críticas
4. **Logs de teste** mostrando o scraper funcionando corretamente
5. **Documentação atualizada** com o novo fluxo e seletores

## LEMBRE-SE

- 🔴 **PRIORIDADE MÁXIMA**: Fazer o scraper trazer TODOS os PDFs sem repetir
- 🔴 **NÃO PARE** no primeiro problema ou primeira solução
- 🔴 **TESTE TUDO** antes de considerar concluído
- 🔴 **USE OS FILTROS**: Data inicial, data final, Status = "Disponíveis"

Comece pela ETAPA 1 (Diagnóstico Completo) e não avance sem completá-la totalmente.
