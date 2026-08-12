# Especificação de Produto — Weather App

**Documento:** `specs/weather-app-spec.md`  
**Versão:** 1.0  
**Data:** 2026-08-12  
**Status:** Pronto para Plan Agent  
**Baseado em:** `specs/discovery.md`

---

## 1. Overview

### Visão Geral

O **Weather App** é uma aplicação web responsiva que permite usuários brasileiros consultar informações de clima atual e previsão de 5 dias para qualquer cidade do mundo. A aplicação é otimizada para dispositivos móveis, oferece interface intuitiva em português brasileiro, e requer zero autenticação ou cadastro.

### Objetivo Principal

Fornecer informações climáticas **rápidas, confiáveis e fáceis de acessar** para usuários que precisam planejar seu dia (roupas, atividades, transporte).

### Público-Alvo

- **Marina:** Commuter urbano (15-30 seg, mobile, necessita rapidez)
- **João:** Agricultor/jardineiro (5-10 min, tablet/mobile, necessita precisão e offline)
- **Sofia:** Mãe planejadora (2-3 min, mobile+desktop, necessita clareza 2-3 dias)

### Princípios de Design

1. **Velocidade:** < 2 segundos de carregamento
2. **Clareza:** Uma ação = uma informação
3. **Confiabilidade:** Funciona mesmo com internet lenta
4. **Acessibilidade:** WCAG 2.1 AA, suporte a leitores de tela
5. **Responsividade:** 320px a 4K, portrait e landscape

### Stack Técnico (Confirmado)

- **Frontend:** React 19.2.7 + TypeScript (strict mode)
- **Styling:** Tailwind CSS (tema dark glassmorphism)
- **Build:** Vite
- **Testes:** Vitest (unit) + Playwright (E2E)
- **Lint:** Biome
- **API:** Open-Meteo (gratuita, sem key)
- **Banco de dados:** Nenhum (MVP stateless)

### Escopo MVP

✅ **Incluído:**
- Busca de cidades
- Clima atual
- Previsão de 5 dias (diária)
- Toggle Celsius ↔ Fahrenheit
- Interface pt-BR
- Responsivo (320px+)
- Acessível (WCAG 2.1 AA)

❌ **Fora de Escopo (v2):**
- Autenticação
- Favoritos persistidos em servidor
- Histórico de busca
- Notificações
- Geolocalização automática
- Previsão horária
- Múltiplas cidades comparadas
- Analytics avançado
- i18n (apenas pt-BR)

---

## 2. Requisitos Funcionais

### RF1: Buscar Cidades

**Descrição:** O usuário deve poder buscar qualquer cidade do mundo digitando seu nome e receber sugestões em tempo real.

**Critérios Técnicos:**
- Campo de busca aceita entrada de texto (português, inglês, caracteres acentuados, 1-100 caracteres)
- Usa endpoint `/geocoding/search` da Open-Meteo
- Mostra até 10 sugestões (ordenadas por população municipal descrescente, depois relevância)
- Cada sugestão exibe: nome da cidade, estado/país, bandeira (emoji)
- **Debounce 300ms:** Requisição disparada 300ms APÓS última digitação (não intervalo)
- Tempo de resposta: < 1 segundo (rede + render)
- Suporta entrada parcial com normalização de acentos (ex: "sao paulo" encontra "São Paulo")
- Filtra por feature_code IN [PPL, PPLC, PPLA] (apenas cidades, não regiões)

**Casos de Sucesso:**
- Usuário digita "São Paulo" → vê "São Paulo, São Paulo, 🇧🇷"
- Usuário digita "Springf" → vê "Springfield" (USA, Canadá, UK com desambiguação)
- Usuário digita "Tokyo" → vê "Tokyo, Japan, 🇯🇵"

---

### RF2: Exibir Clima Atual

**Descrição:** Após selecionar uma cidade, exibir clima atual com temperatura, condição, ícone, umidade e vento.

**Critérios Técnicos:**
- Busca dados via `/forecast?latitude=...&longitude=...` endpoint (Open-Meteo)
- Exibe em tempo real (atualiza ao selecionar cidade ou ao clicar botão "Atualizar")
- **Campos exibidos:**
  - Temperatura: 1 casa decimal (ex: 28.5°C) em Celsius por padrão
  - Descrição: traduzida para pt-BR (ex: "Céu limpo", não "Clear sky")
  - Ícone: corresponde a weather_code (ver Apêndice A)
  - Umidade: porcentagem (ex: 65%)
  - Vento: velocidade em km/h (ex: 12 km/h)
  - Timestamp: "Atualizado em HH:MM" (hora local)
- Conversão C↔F via toggle (fórmula: F = C×9/5+32, 1 casa decimal, ±0.1°F tolerância)
- Carregamento: exibe skeleton com pulsação até dados chegarem

**Campos Obrigatórios:**
- `temperature_current` (°C)
- `weather_code` → mapeado para ícone + descrição
- `relative_humidity_2m` (%)
- `wind_speed_10m` (km/h)

**Estados:**
- Loading: skeleton com pulsação
- Success: dados exibidos
- Error: mensagem amigável + retry button

---

### RF3: Exibir Previsão de 5 Dias

**Descrição:** Mostrar previsão diária para próximos 5 dias (hoje + D+1 a D+4) com max/min, condição, ícone.

**Critérios Técnicos:**
- Cada dia exibe: data (ex: "12 de ago"), temp máxima, temp mínima, descrição do clima, ícone
- Dados vêm de `/forecast` endpoint (diária, não horária)
- Conversão automática C↔F se usuário ativou toggle
- Card por dia: layout compacto, scroll horizontal em mobile se necessário
- Cores/ícones indicam condição (cinza chuva, amarelo sol, etc)
- Expandível: ao clicar em um dia, pode mostrar detalhes adicionais (opcional em v2)

**Validação:**
- **"Hoje" = data civil local** baseada em timezone do navegador (00:00-23:59)
- D+1 a D+4 = próximos 4 dias consecutivos UTC (não relativos a hoje)
- **Sempre exibe exatamente 5 dias** — se Open-Meteo retorna < 5, é erro crítico (retry)
- Datas nunca "invertem" de timezone (respeita ISO 8601 com offset)

---

### RF4: Alternar Celsius ↔ Fahrenheit

**Descrição:** Usuário pode alternar entre Celsius e Fahrenheit; preferência é salva.

**Critérios Técnicos:**
- Toggle button na interface (ex: ícone termômetro C/F ou radiobutton)
- Ao alternar: atualiza TODAS as temperaturas na tela (atual + previsão)
- Salva preferência em localStorage (chave: `temperatureUnit`)
- Próxima visita carrega a unidade salva
- Sincroniza entre abas (se abrir app em 2 abas, toggle em uma = atualiza outra)
- Fórmula: F = (C × 9/5) + 32, arredondado para 1 casa decimal

**Validação:**
- Toggle sempre disponível (mesmo ao carregar)
- Padrão = Celsius (se localStorage vazio)
- Conversão matematicamente correta (±0.1°F tolerância)

---

### RF5: Suporte a Múltiplas Cidades (Não-MVP, mas pronto para v2)

**Descrição:** Usuário pode visualizar clima de múltiplas cidades. **[OUT OF SCOPE MVP, mas arquitetura deve permitir]**

**Nota:** Será implementado em v2 com favoritos. MVP = uma cidade por vez.

---

## 3. User Stories

### US1: Buscar Cidades — Achado Rápido

**Como Marina (commuter urbano), quero buscar uma cidade digitando seu nome para encontrar informações do clima em menos de 30 segundos.**

**Conexão com Requisito Funcional:** RF1 (Buscar Cidades)

**Critérios de Aceite:**
- [ ] Campo de busca aceita mínimo 1 caractere, máximo 100
- [ ] Debounce de 300ms evita requisições excessivas
- [ ] Retorna até 10 sugestões ordenadas por população
- [ ] Cada sugestão mostra: nome da cidade, estado/país, bandeira (emoji)
- [ ] Suporta acentos (ã, é, ç, ú) e caracteres especiais
- [ ] Tempo de resposta ≤ 1 segundo após debounce
- [ ] Mensagem "Nenhuma cidade encontrada" se resultado vazio
- [ ] Ao clicar sugestão, carrega clima dessa cidade

**Cenário de Sucesso:**
1. Abre app no smartphone
2. Digita "são" no campo de busca
3. Vê sugestões: "São Paulo, SP 🇧🇷" | "São Gonçalo, RJ 🇧🇷" | "São Leopoldo, RS 🇧🇷"
4. Seleciona "São Paulo, SP 🇧🇷"
5. App carrega clima atual de São Paulo em < 2 segundos

---

### US2: Visualizar Clima Atual — Informação Essencial

**Como Marina (commuter urbano), quero ver a temperatura, condição e risco de chuva para decidir que roupa usar antes de sair de casa.**

**Conexão com Requisito Funcional:** RF2 (Exibir Clima Atual)

**Critérios de Aceite:**
- [ ] Exibe temperatura em Celsius (padrão)
- [ ] Mostra descrição textual da condição (ex: "Céu limpo", "Nublado", "Chuva leve")
- [ ] Ícone visual representa a condição climática
- [ ] Exibe umidade relativa (%) e velocidade do vento (km/h)
- [ ] Timestamp indica "Atualizado em [horário]"
- [ ] Durante carregamento, mostra skeleton/spinner
- [ ] Em caso de erro, exibe mensagem amigável + botão retry
- [ ] Dados atualizam ao selecionar nova cidade
- [ ] Interface se adapta a 320px (mobile) até 4K (desktop)

**Cenário de Sucesso:**
1. Seleciona "São Paulo, SP"
2. Vê: "28°C | Céu limpo ☀️"
3. Vê adicionais: "Umidade 65% | Vento 12 km/h"
4. Vê timestamp: "Atualizado em 14:32"
5. Fecha app; próxima vez que abre, consegue ver este dado mesmo com Internet lenta

---

### US3: Consultar Previsão Estendida — Planejamento Agrícola

**Como João (agricultor), quero consultar a previsão de 5 dias com temperaturas máximas, mínimas, condição e ícones para planejar irrigação, plantio e colheita com precisão.**

**Conexão com Requisito Funcional:** RF3 (Exibir Previsão de 5 Dias)

**Critérios de Aceite:**
- [ ] Exibe exatamente 5 dias (hoje + próximos 4 dias)
- [ ] Cada dia mostra: data (ex: "12 ago"), temp máxima, temp mínima, condição, ícone
- [ ] Datas estão no formato DD mês (ex: "12 ago", "13 ago")
- [ ] Temperaturas refletem toggle C/F (padrão Celsius)
- [ ] Cards são organizados em grid responsivo
- [ ] Em mobile, permite scroll horizontal se necessário
- [ ] Cores/ícones indicam condição (cinza chuva, amarelo sol, azul nublado)
- [ ] Dados carregam junto ao clima atual (< 2 seg)

**Cenário de Sucesso:**
1. Seleciona "Campinas, SP"
2. Vê previsão 5 dias:
   - "12 ago: Máx 32°C | Mín 22°C | Céu limpo ☀️"
   - "13 ago: Máx 30°C | Mín 20°C | Chuva 🌧️"
   - "14 ago: Máx 28°C | Mín 18°C | Nublado ☁️"
   - "15 ago: Máx 31°C | Mín 21°C | Chuva leve 🌦️"
   - "16 ago: Máx 29°C | Mín 19°C | Céu limpo ☀️"
3. Decide: "Vou irrigar hoje (12), esperar chuva amanhã (13)"

---

### US4: Alternar Unidade de Temperatura — Preferência Persistida

**Como Sofia (mãe planejadora), quero alternar entre Celsius e Fahrenheit e ter essa preferência salva para minhas próximas visitas ao app.**

**Conexão com Requisito Funcional:** RF4 (Alternar Celsius ↔ Fahrenheit)

**Critérios de Aceite:**
- [ ] Toggle button/selector visível e acessível (sempre na interface)
- [ ] Ao clicar, todas as temperaturas (atual + previsão 5 dias) se atualizam
- [ ] Conversão correta: F = (C × 9/5) + 32, com ±0.1°F tolerância
- [ ] Preferência salva em localStorage (chave: `temperatureUnit`)
- [ ] Próxima visita carrega a unidade salva automaticamente
- [ ] Toggle sincroniza entre abas (se abrir app em 2 abas, muda em uma = atualiza outra)
- [ ] Padrão inicial = Celsius (se localStorage vazio ou primeira vez)
- [ ] Estado do toggle persiste após fechar/abrir o navegador

**Cenário de Sucesso:**
1. Abre app vê "28°C" (padrão Celsius)
2. Clica toggle "°F"
3. Vê "82°F" (conversão correta: 28 × 9/5 + 32 = 82.4)
4. Também vê previsão atualizada: "Máx 90°F | Mín 72°F"
5. Fecha aba e reabre
6. App ainda mostra "°F" (localStorage recuperou preferência)
7. Abre app em outra aba → também exibe "°F" (sincronização)

---

### US5: Funcionar Offline — Confiabilidade em Baixa Conectividade

**Como João (agricultor), quero que o app funcione mesmo sem Internet, mostrando dados em cache de até 24 horas atrás, para planejar meu dia mesmo com conexão 3G ou rural intermitente.**

**Conexão com Requisito Funcional:** RF2 + RF3 (Clima Atual + Previsão com cache)

**Critérios de Aceite:**
- [ ] Quando sem Internet, exibe dados cacheados (se disponível)
- [ ] Mostra label "Offline - dados de [tempo atrás]" (ex: "Offline - 4 horas atrás")
- [ ] Cache expira após 24 horas (TTL = 24h)
- [ ] Ao recuperar conexão, tenta atualizar dados automaticamente
- [ ] Se sem cache E sem internet, mostra msg: "Sem conexão e sem dados cacheados"
- [ ] Botão retry permite tentar reconectar manualmente
- [ ] Retry usa backoff exponencial (1s, 2s, 4s, 8s; máx 3 tentativas)
- [ ] localStorage é usado para cache (sem backend necessário)

**Cenário de Sucesso:**
1. Manhã: Abre app em área rural, vê "São Paulo: 28°C, Céu limpo"
2. Dados são cacheados em localStorage
3. Perde Internet (3G cai)
4. Reabre app: ainda vê dados, mas com label "Offline - 2 horas atrás"
5. Internet volta
6. App tenta atualizar automaticamente; sucesso em 1ª tentativa
7. Vê dados novos: "28°C → 26°C" (abaixou)

---

### US6: Aceitar Acentos e Caracteres Especiais — Busca Multilíngue

**Como Sofia (mãe planejadora), quero buscar cidades usando nomes com acentos (São Paulo, Brasília, Paraíba) sem precisar se preocupar com caracteres especiais, para encontrar minha região facilmente.**

**Conexão com Requisito Funcional:** RF1 (Busca de Cidades com suporte a acentos)

**Critérios de Aceite:**
- [ ] Campo de busca aceita: a-z, A-Z, 0-9, acentos (á, é, í, ó, ú, ã, õ, ç)
- [ ] Busca por "sao paulo" retorna "São Paulo" (tolera omissão de acentos)
- [ ] Busca por "são paulo" retorna "São Paulo" (com acentos funciona)
- [ ] Busca por "brasília" funciona corretamente
- [ ] Busca por "maceió" funciona corretamente
- [ ] Suporta também busca em inglês (ex: "Rio de Janeiro" ou "rio janeiro")
- [ ] Caracteres especiais (hífens, apóstrofos) são suportados
- [ ] Resultados não são afetados por variações de acentuação (busca normalizável)

**Cenário de Sucesso:**
1. Digita "são p" (com acento, incompleto)
2. Vê sugestão: "São Paulo, SP 🇧🇷" (match encontrado)
3. Digita "sao paulo" (sem acentos, completo)
4. Ainda vê sugestão: "São Paulo, SP 🇧🇷" (fuzzy match funciona)
5. Digita "brasília"
6. Vê sugestão: "Brasília, DF 🇧🇷"
7. Clica em qualquer uma, clima carrega corretamente

---

## 4. Acceptance Criteria (BDD — Given/When/Then)

Critérios formatados em **Gherkin/BDD** para automação em testes Vitest (unit) e Playwright (E2E).

---

### AC1: Busca de Cidades (RF1)

#### Cenário 1.1: Usuário digita cidade válida e recebe sugestões

```gherkin
Given o app está carregado e o campo de busca é visível
When o usuário digita "são paulo" no campo de busca
And aguarda 300ms (debounce)
Then aparecem até 10 sugestões de cidades
And cada sugestão exibe: nome, estado/país, bandeira (emoji)
And a primeira sugestão é "São Paulo, SP 🇧🇷" (por população)
```

#### Cenário 1.2: Debounce evita requisições excessivas

```gherkin
Given o campo de busca está vazio
When o usuário digita "são p" (4 caracteres em rápida sucessão)
Then apenas UMA requisição é enviada à API
And a requisição é disparada após 300ms de inatividade (última digitação)
```

#### Cenário 1.3: Cidade não encontrada

```gherkin
Given o app está carregado
When o usuário digita "xyzabc123" (cidade inexistente)
And aguarda 300ms
Then a API retorna lista vazia
And é exibida a mensagem: "Nenhuma cidade encontrada"
And nenhum card de sugestão aparece
```

#### Cenário 1.4: Suporta acentos e caracteres especiais

```gherkin
Given o campo de busca está vazio
When o usuário digita "brasília" (com acento)
Then aparecem sugestões com "Brasília, DF 🇧🇷"
When o usuário limpa e digita "brasilia" (sem acento)
Then aparecem as MESMAS sugestões (busca tolera omissão de acentos)
```

#### Cenário 1.5: Seleção de sugestão carrega clima

```gherkin
Given sugestões aparecem para "São Paulo"
When o usuário clica na sugestão "São Paulo, SP 🇧🇷"
Then a requisição `/forecast` é enviada para São Paulo
And o clima atual é exibido em < 2 segundos
And a previsão de 5 dias é exibida
```

#### Cenário 1.6: Validação de entrada (caracteres inválidos)

```gherkin
Given o campo de busca está vazio
When o usuário tenta digitar sequências inválidas: "@#$%", "<script>", "\x00"
Then caracteres inválidos são IGNORADOS (sanitizados)
And apenas caracteres válidos são processados (a-z, A-Z, 0-9, acentos, hífens)
```

---

### AC2: Exibir Clima Atual (RF2)

#### Cenário 2.1: Clima atual exibido com todos os campos

```gherkin
Given a cidade "São Paulo" foi selecionada
When os dados de `/forecast` são recebidos
Then a interface exibe:
  • Temperatura em Celsius: "28.5°C" (1 casa decimal)
  • Descrição traduzida: "Céu limpo" (não "Clear sky")
  • Ícone correspondente: ☀️ (sol/claro)
  • Umidade: "65%"
  • Vento: "12 km/h"
  • Timestamp: "Atualizado em 14:32" (hora local)
```

#### Cenário 2.2: Estado de carregamento (loading)

```gherkin
Given o usuário seleciona uma nova cidade
When a requisição `/forecast` é disparada
Then um skeleton/spinner é exibido no card de clima atual
And o texto "Carregando..." (ou equivalente visual) aparece
And após dados chegarem, o skeleton é substituído pelos dados
```

#### Cenário 2.3: Erro ao buscar clima

```gherkin
Given a cidade "São Paulo" foi selecionada
When a requisição `/forecast` falha (timeout, 500, etc)
Then uma mensagem amigável é exibida: "Erro ao carregar clima. Tente novamente."
And um botão "Atualizar" ou "Retry" está disponível
When o usuário clica "Atualizar"
Then a requisição é refeita
```

#### Cenário 2.4: Botão de atualização manual

```gherkin
Given o clima de "São Paulo" está sendo exibido
When o usuário clica no botão "Atualizar" (ou ícone 🔄)
Then a requisição `/forecast` é disparada novamente
And o timestamp muda para novo horário
And dados antigos são substituídos por dados frescos
```

#### Cenário 2.5: Conversão de unidades (padrão Celsius)

```gherkin
Given o clima atual é exibido: "28.5°C"
When o usuário não tocou o toggle C/F
Then a unidade permanece em Celsius
When o usuário clica toggle "°F"
Then a temperatura se atualiza para "83.3°F" (conversão correta)
And a descrição do ícone e umidade PERMANECEM (só temperatura muda)
```

---

### AC3: Previsão de 5 Dias (RF3)

#### Cenário 3.1: Exibe exatamente 5 dias com datas corretas

```gherkin
Given hoje é 12 de agosto de 2026
When a previsão é carregada
Then exatamente 5 cards aparecem:
  1. "12 ago" (hoje)
  2. "13 ago" (D+1)
  3. "14 ago" (D+2)
  4. "15 ago" (D+3)
  5. "16 ago" (D+4)
And nenhum card a mais, nenhum card a menos
```

#### Cenário 3.2: Cada dia mostra max, min, ícone e descrição

```gherkin
Given a previsão está carregada
When o usuário visualiza o card do dia "13 ago"
Then o card exibe:
  • Data: "13 ago"
  • Temperatura máxima: "30°C"
  • Temperatura mínima: "20°C"
  • Ícone: 🌧️ (chuva)
  • Descrição: "Chuva leve"
```

#### Cenário 3.3: Aplicar toggle C/F à previsão

```gherkin
Given a previsão mostra: "Máx 30°C | Mín 20°C"
When o usuário clica toggle "°F"
Then todos os 5 dias se atualizam:
  • Máx 86°F (30 × 9/5 + 32 = 86)
  • Mín 68°F (20 × 9/5 + 32 = 68)
And as datas e ícones NÃO mudam
```

#### Cenário 3.4: Layout responsivo em mobile

```gherkin
Given a viewport é 320px (mobile)
When a previsão de 5 dias é exibida
Then um dos seguintes é verdadeiro:
  a) Cards são exibidos em scroll horizontal (carousel)
  b) Cards são empilhados verticalmente (1 por linha)
And todos os 5 cards são acessíveis (sem overflow oculto)
```

#### Cenário 3.5: Cores/ícones indicam condição

```gherkin
Given a previsão está carregada
When o usuário visualiza cards com diferentes condições:
  • "Céu limpo" → ícone ☀️ (amarelo/ouro)
  • "Chuva" → ícone 🌧️ (azul/cinza)
  • "Nublado" → ícone ☁️ (cinza)
Then cada ícone e cor correspondem corretamente à condição
And não há confusão visual (chuva não é amarela)
```

---

### AC4: Alternar Celsius ↔ Fahrenheit (RF4)

#### Cenário 4.1: Toggle é sempre visível e acessível

```gherkin
Given o app está carregado (qualquer página/estado)
When o usuário visualiza a interface
Then o toggle C/F é visível (ex: botão ou radiobutton)
And o toggle está sempre acessível (não oculto, não scrollado para fora)
And o toggle tem aria-label descritivo (ex: "Alternar entre Celsius e Fahrenheit")
```

#### Cenário 4.2: Padrão é Celsius na primeira visita

```gherkin
Given é a primeira visita do usuário (localStorage vazio)
When o app é carregado
Then o toggle mostra "°C" como selecionado
And todas as temperaturas são exibidas em Celsius
And localStorage['temperatureUnit'] ainda não existe
```

#### Cenário 4.3: Clique no toggle altera todos os valores

```gherkin
Given clima atual mostra "28°C" e previsão mostra "Máx 30°C | Mín 20°C"
When o usuário clica toggle para "°F"
Then clima atual muda para "82.4°F" (arredondado a 1 casa decimal)
And previsão muda para "Máx 86°F | Mín 68°F"
And todas as temperaturas são atualizadas SIMULTANEAMENTE (sem delay)
```

#### Cenário 4.4: Preferência é salva em localStorage

```gherkin
Given o usuário alterna para "°F"
When a mudança é confirmada
Then localStorage['temperatureUnit'] = "fahrenheit" (salvo)
When o usuário fecha e reabre o navegador
Then o app carrega com "°F" como padrão
And localStorage['temperatureUnit'] = "fahrenheit"
```

#### Cenário 4.5: Sincroniza entre múltiplas abas

```gherkin
Given o app está aberto em Aba 1 com "°C" selecionado
When o usuário abre o app em Aba 2
Then Aba 2 também mostra "°C" (localStorage sincronizado)
When o usuário clica toggle "°F" na Aba 1
Then Aba 2 recebe evento de storage
And Aba 2 atualiza automaticamente para "°F" (sem refresh manual)
```

#### Cenário 4.6: Conversão é matematicamente correta

```gherkin
Given os seguintes valores em Celsius:
  • 0°C, 28°C, -40°C, 100°C
When conversão F = (C × 9/5) + 32 é aplicada
Then os resultados são:
  • 0°C → 32°F ✓
  • 28°C → 82.4°F ✓
  • -40°C → -40°F ✓
  • 100°C → 212°F ✓
And tolerância é ±0.1°F
```

---

### AC5: Performance (RNF-PERF)

#### Cenário 5.1: Carregamento inicial < 2 segundos

```gherkin
Given o app é acessado pela primeira vez (sem cache)
When a página carrega completamente
Then Largest Contentful Paint (LCP) ≤ 2.5 segundos
And First Input Delay (FID) ≤ 100ms
And Cumulative Layout Shift (CLS) ≤ 0.1
And medições são feitas em Lighthouse com throttle "Slow 3G"
```

#### Cenário 5.2: Busca de cidade responde em < 1 segundo

```gherkin
Given o usuário digita "são paulo" no campo de busca
When 300ms passam (debounce finalizado)
Then a resposta da API é recebida
And sugestões aparecem na interface
And tempo total ≤ 1 segundo (rede + render)
```

#### Cenário 5.3: UI responde a input em < 100ms

```gherkin
Given o usuário digita uma letra no campo de busca
When o evento "onChange" é disparado
Then o carácter aparece no input
And o campo não congela (resposta imediata, < 100ms)
```

#### Cenário 5.4: Bundle size < 150KB (gzipped)

```gherkin
Given o app é construído com `pnpm build`
When a build é finalizada
Then o arquivo de output (dist/index-*.js) comprimido é ≤ 150KB
And relatório de bundle pode ser gerado: `pnpm build --report`
```

---

### AC6: Acessibilidade — WCAG 2.1 AA (RNF-A11Y)

#### Cenário 6.1: Contraste mínimo 4.5:1

```gherkin
Given o app é visualizado em modo light e dark
When texto em qualquer cor é exibido (sobre fundo)
Then relação de contraste ≥ 4.5:1 (testado com axe DevTools ou similar)
And texto normal (tamanho >= 14px) atende WCAG AA
And texto grande (tamanho >= 18px) atende WCAG AA (3:1 suficiente)
```

#### Cenário 6.2: Navegação por teclado funciona

```gherkin
Given o app está carregado
When o usuário pressiona Tab
Then foco move entre elementos: campo busca → sugestões → toggle C/F → botão atualizar
And Enter seleciona elemento focado (sugestão ou botão)
And Escape fecha dropdown de sugestões
And Tab + Shift move foco para trás
And nenhum elemento fica "preso" (focusável)
```

#### Cenário 6.3: Rótulos ARIA em todos os controles interativos

```gherkin
Given a interface é analisada com inspetor de acessibilidade
When cada botão/input/dropdown é examinado
Then:
  • Input de busca tem aria-label: "Buscar cidades"
  • Toggle C/F tem aria-label: "Alternar entre Celsius e Fahrenheit"
  • Botão Atualizar tem aria-label: "Atualizar clima"
  • Dropdown de sugestões tem role="listbox" e itens role="option"
```

#### Cenário 6.4: Leitor de tela funciona (screen reader compatible)

```gherkin
Given o app é acessado com NVDA (Windows) ou VoiceOver (Mac)
When o leitor de tela navega pela interface
Then:
  • Lê campo de busca e seu rótulo
  • Lê sugestões com "São Paulo, SP, Brasil"
  • Lê temperatura e unidade: "28 graus Celsius"
  • Lê previsão diária com data, máx, mín, condição
And navegação é clara e sequencial (sem saltos)
```

#### Cenário 6.5: Zoom até 200% funciona

```gherkin
Given o navegador é configurado para 200% zoom
When o app é visualizado
Then:
  • Texto permanece legível (não truncado)
  • Botões permanecem clicáveis
  • Layout não quebra (não overflow horizontal)
  • Scroll permite acessar todo conteúdo
```

#### Cenário 6.6: Dark mode nativo é respeitado

```gherkin
Given o SO/navegador está em modo dark (prefers-color-scheme: dark)
When o app carrega
Then:
  • Tema dark glassmorphism é aplicado automaticamente
  • Cores estão otimizadas para dark mode
  • Usuário não precisa clicar em botão "Dark mode"
When o usuário muda SO para light
Then o app se adapta automaticamente (sem reload)
```

#### Cenário 6.7: Alto contraste do SO (Windows High Contrast)

```gherkin
Given o Windows é configurado para "High Contrast" mode
When o app é carregado
Then:
  • Cores de alto contraste do SO são respeitadas
  • App permanece funcional (contraste ≥ 7:1)
  • Borders/outlines são visíveis
```

#### Cenário 6.8: Redução de movimento é respeitada

```gherkin
Given o SO está configurado para prefers-reduced-motion: reduce
When o app carrega
Then:
  • Animações são DESABILITADAS (ou simplesmente removidas)
  • Transições CSS (ex: fade-in) são instantâneas ou removidas
  • Usuarios com sensibilidade a movimento não ficam indispostos
When prefers-reduced-motion: no-preference
Then animações retornam ao normal (ex: fade-in de skeleton)
```

---

### AC7: Responsividade (RNF-RESP)

#### Cenário 7.1: Mobile (320px) — Layout single-column

```gherkin
Given a viewport é 320px de largura (mobile pequeno)
When o app é carregado
Then:
  • Campo de busca é exibido full-width
  • Sugestões são exibidas em lista (full-width)
  • Clima atual ocupa full-width
  • Previsão usa scroll horizontal (5 cards em carousel)
  • Tudo é legível (sem truncamento excessivo)
  • Toggle C/F é acessível (thumb-friendly, ≥ 44x44px)
```

#### Cenário 7.2: Tablet (768px) — Layout otimizado

```gherkin
Given a viewport é 768px de largura (tablet portrait)
When o app é carregado
Then:
  • Layout pode usar 2 colunas (ex: busca esquerda, clima direita)
  • Cards de previsão podem aparecer em grid (ex: 3 colunas)
  • Espaçamento é adequado (não muito aperto)
```

#### Cenário 7.3: Desktop (1024px+) — Full-width

```gherkin
Given a viewport é ≥ 1024px (desktop)
When o app é carregado
Then:
  • Layout aproveita espaço (não é cramped em sidebar pequena)
  • Máxima largura é razoável (ex: 1200px max-width para legibilidade)
  • Grid de previsão pode ser 5 colunas (1 por dia)
```

#### Cenário 7.4: Portrait e Landscape funcionam

```gherkin
Given o device está em portrait (300px x 600px)
When o app é visualizado
Then layout é otimizado para portrait (tudo visível)
When o device é rotacionado para landscape (600px x 300px)
Then:
  • Layout se adapta automaticamente
  • Nenhum conteúdo desaparece
  • Legibilidade é mantida
And não há erros de console
```

#### Cenário 7.5: Imagens/ícones escalam corretamente

```gherkin
Given ícones de clima (SVG/PNG) estão em uso
When o viewport muda (320px → 1920px)
Then:
  • Ícones não ficam pixelados
  • Ícones mantêm proporção
  • Tamanho é legível em todas as resoluções (ex: min 20px, max 48px)
```

---

## 5. Requisitos Não-Funcionais

### Performance (RNF-PERF)

- **Tempo de Carregamento:** App carrega em < 2 segundos (medido em 3G, Lighthouse)
- **Busca de Cidade:** Resposta em < 1 segundo
- **Resposta UI:** Interface reage a input em < 100ms
- **Bundle Size:** < 150KB (gzipped)
- **Core Web Vitals:**
  - LCP (Largest Contentful Paint): < 2.5s
  - FID (First Input Delay): < 100ms
  - CLS (Cumulative Layout Shift): < 0.1

### Usabilidade & Acessibilidade (RNF-A11Y)

- **WCAG 2.1 AA:** 100% conformidade
- **Navegação por Teclado:** Tab, Enter, Esc; sem necessidade de mouse
- **Contraste:** Mínimo 4.5:1 para texto normal
- **Labels Semânticas:** Todos elementos interativos têm `aria-label` ou `<label>`
- **Leitores de Tela:** Funcional com NVDA, JAWS, VoiceOver, TalkBack

### Assistive Technologies (RNF-ASST)

- **Zoom:** Responsivo até 200%
- **Modo Alto Contraste:** Windows High Contrast Mode funcional
- **Redução de Movimento:** Respeita `prefers-reduced-motion` (sem animações)
- **Dark Mode:** Suporta `prefers-color-scheme: dark` nativo

### Responsividade (RNF-RESP)

- **Mobile:** 320px-768px (funcional, otimizado)
- **Tablet:** 768px-1024px (otimizado)
- **Desktop:** > 1024px (full-width)
- **Orientação:** Portrait e Landscape suportados

### Confiabilidade (RNF-REL)

- **Uptime:** 99% (dependente de Open-Meteo)
- **Tratamento de Erros:** Mensagens amigáveis (não técnicas)
- **Retry:** Exponential backoff com 3 tentativas (ver Apêndice E: 1s, 2s, 4s delays)
- **Storage:** localStorage com fallback automático
  - Prioridade: localStorage → sessionStorage → memória
  - Se localStorage indisponível: preferência perdida ao fechar aba
- **Cache:** Último clima consultado com 24h TTL
  - Formato: JSON serializado em `localStorage['weatherCache']`
  - Expiração: `savedAt + 86.400 segundos`
  - Exibido com label "Offline — dados de X horas atrás"

### Segurança (RNF-SEC)

- **HTTPS:** Todas comunicações criptografadas
- **Validação de Input:** Sanitizar entrada de usuário (XSS prevention)
- **Validação de API:** Schema validation para dados Open-Meteo
- **Sem PII:** Não coletar, logar ou armazenar dados pessoais
- **localStorage:** Apenas preferência C/F (não identificável)

### Compatibilidade de Navegadores (RNF-COMPAT)

- **Chrome/Chromium:** 90+
- **Firefox:** 88+
- **Safari:** 14+
- **Edge:** 90+
- **Mobile:** iOS Safari 14+, Chrome Android 90+
- **Fallback:** Navegadores antigos recebem mensagem informativa

### Conformidade Legal (RNF-LEGAL)

- **LGPD (Brasil):** localStorage é informado ao usuário
- **GDPR (EU):** Conformidade se acessado de EU
- **Política de Privacidade:** Documentada e acessível
- **Sem Tracking Externo:** Zero analytics sem consentimento

### Tecnologia (RNF-TECH)

- **Frontend:** React 19.2.7 + TypeScript (strict)
- **Build:** Vite
- **Styling:** Tailwind CSS
- **Testes:** Vitest (unit) + Playwright (E2E)
- **Lint:** Biome
- **API:** Open-Meteo (/geocoding/search, /forecast)

---

## 6. Edge Cases (Comportamentos Esperados)

---

### EC1: Cidade Não Encontrada (Geocoding sem Resultados)

**Descrição:** Usuário digita cidade inexistente ou nome muito genérico que a API não encontra.

```gherkin
Given o usuário digita "xyzabc" (cidade fictícia)
When aguarda 300ms (debounce) e a API `/geocoding/search` é chamada
Then a API retorna lista vazia
And a interface exibe mensagem: "Nenhuma cidade encontrada para 'xyzabc'"
And uma sugestão de ajuda: "Tente com outro nome ou verifique a ortografia"
And o campo de busca permanece focado (pronto para nova tentativa)
And não há spinner/loading visível (requisição finalizou)
```

**Comportamento Esperado:**
- Mensagem amigável em pt-BR
- Campo de busca não é "travado" (usuário pode continuar digitando)
- Histórico de busca não é atualizado (falha não conta como "busca válida")

---

### EC2: Input Vazio ou Apenas Espaços

**Descrição:** Usuário clica em buscar sem digitar nada, ou digita apenas espaços em branco.

```gherkin
Given o campo de busca está vazio ou contém apenas espaços " "
When o usuário pressiona Enter ou tira o foco do campo
Then NENHUMA requisição à API é enviada (validação no cliente)
And nenhum spinner/loading aparece
And a interface permanece no estado anterior (sem erro)
```

**Comportamento Esperado:**
- Input vazio = validação bloqueada localmente (não consome API)
- Espaços em branco são trimados (ex: " são paulo " = "são paulo")
- Usuário não vê mensagem de erro (apenas não há sugestões)

---

### EC3: Input Muito Longo (> 100 caracteres)

**Descrição:** Usuário digita mais de 100 caracteres (limite de segurança).

```gherkin
Given o usuário digita 150 caracteres no campo de busca
When atinge 100 caracteres
Then caracteres adicionais são IGNORADOS (input truncado em 100)
And apenas os primeiros 100 caracteres são processados
And API recebe apenas 100 caracteres (não 150)
```

**Comportamento Esperado:**
- Limite de 100 caracteres é uma proteção contra DoS
- Campo não permite digitar além (ou silenciosamente trunca)
- Sem mensagem de erro (discreto)

---

### EC4: Caracteres Especiais e Acentos

**Descrição:** Usuário digita cidades com acentos, hífens, apóstrofos, etc.

```gherkin
Given o usuário digita cidades com acentos e caracteres especiais:
  • "São Paulo" (ã)
  • "Zürich" (ü)
  • "Québec" (é)
  • "Port-au-Prince" (hífen)
  • "L'Isle-d'Abeau" (apóstrofo)
When a API é chamada
Then todos os casos funcionam corretamente
And cada sugestão é retornada com a grafia correta
And a busca não falha por caracteres especiais
```

**Comportamento Esperado:**
- Suporta Unicode (acentos latinos, caracteres especiais)
- Não há error de encoding
- Sugestões aparecem com a grafia original (ex: "São Paulo", não "Sao Paulo")

---

### EC5: Temperatura Extrema

**Descrição:** Open-Meteo retorna valores extremos (Antártida -50°C ou Deserto 60°C).

```gherkin
Given a Open-Meteo retorna dados para Antártida:
  • temperature_current: -50
  • weather_code: 71 (snow)
When os dados são exibidos
Then a temperatura é exibida como "-50°C" (sem truncamento)
And conversão para Fahrenheit: -50°C = -58°F (fórmula correta)
And o ícone mostra neve (apropriado para -50°C)
And layout não quebra (números negativos em unidades são tratados)
```

**Comportamento Esperado:**
- Sem limite artificial de temperatura (não bloqueia valores reais)
- Conversão C→F funciona corretamente para valores extremos
- Ícone e descrição refletem a realidade (frio extremo = neve/ícone apropriado)

---

### EC6: API Rate Limit (Muitas Requisições)

**Descrição:** Usuário digita muito rápido, acionando múltiplas requisições.

```gherkin
Given o usuário digita rápido: "s", "á", "o", " ", "p", "a", "u", "l", "o"
When cada tecla é pressionada em < 100ms (sem debounce)
Then apenas UMA requisição é enviada após 300ms de inatividade
And o debounce evita requisições em excesso
And nenhuma mensagem de "rate limit" aparece ao usuário
```

**Comportamento Esperado:**
- Debounce de 300ms protege contra rate limit automático
- Se Open-Meteo retornar erro 429 (rate limit), exibir: "Muitos acessos. Aguarde alguns segundos."
- Usuário pode tentar novamente após aguardar

---

### EC7: Falha de API — Erro HTTP 500

**Descrição:** Open-Meteo retorna erro servidor (500, 502, 503).

```gherkin
Given a Open-Meteo retorna HTTP 500 (Internal Server Error)
When o usuário tenta buscar "São Paulo"
Then a requisição falha
And é exibida mensagem: "Erro ao carregar dados. Tente novamente."
And um botão "Atualizar" é disponibilizado
When o usuário clica "Atualizar"
Then uma nova requisição é enviada
And se ainda falhar, mensagem persiste
```

**Comportamento Esperado:**
- Mensagem amigável (não mostra "HTTP 500" ou stack trace)
- Retry manual sempre disponível
- Se tiver dados em cache, mostrar: "Exibindo dados offline (podem estar desatualizados)"

---

### EC8: Timeout — API Não Responde (> 8 Segundos)

**Descrição:** Open-Meteo demora mais de 8 segundos para responder.

```gherkin
Given o usuário busca "São Paulo"
When a requisição é enviada e a API não responde em 8 segundos
Then a requisição é abortada (timeout acionado)
And é exibida mensagem: "Problemas ao conectar. Tentando novamente..."
And retry automático é disparado (até 3 tentativas com backoff exponencial):
  • 1ª tentativa: aguarda 1 segundo
  • 2ª tentativa: aguarda 2 segundos
  • 3ª tentativa: aguarda 4 segundos
And após 3 falhas: "Não foi possível carregar. Tente novamente mais tarde."
```

**Comportamento Esperado:**
- Timeout cliente é de ~8 segundos (não deixa usuário pendurado infinitamente)
- Retry automático com backoff exponencial (não bombardeia servidor)
- Mensagem indica que o app está "tentando novamente" (transparência)

---

### EC9: Sem Conexão de Internet (Offline)

**Descrição:** Usuário não tem conexão de Internet (wifi desligado, 3G caiu).

```gherkin
Given o usuário já buscou "São Paulo" antes (dados em cache)
When a conexão de Internet cai
And o usuário tenta buscar "Rio de Janeiro"
Then a requisição falha (erro de rede)
And é exibida mensagem: "Sem conexão de Internet"
And dados cacheados de "São Paulo" permanecem acessíveis:
  • Exibe clima anterior com label: "Offline — dados de 2 horas atrás"
  • Não é possível fazer nova busca (offline)
When a conexão é restaurada
Then o app tenta atualizar automaticamente
And a busca "Rio de Janeiro" é refeita
```

**Comportamento Esperado:**
- Cache local permite visualizar último clima buscado (mesmo offline)
- Label "Offline" deixa claro que dados podem estar desatualizados
- Novo search é bloqueado offline (não pode buscar nova cidade sem Internet)

---

### EC10: Resposta Parcial — Alguns Campos Faltam

**Descrição:** Open-Meteo retorna resposta, mas alguns campos estão ausentes ou null.

```gherkin
Given a Open-Meteo retorna dados incompletos:
  {
    "current": {
      "temperature": 28,
      "weather_code": 1,
      "humidity": null,        # FALTA
      "wind_speed": undefined  # FALTA
    }
  }
When a resposta é processada
Then:
  • Temperatura é exibida: "28°C" ✓
  • Umidade mostra fallback: "N/A" (ou omitida)
  • Vento mostra fallback: "N/A" (ou omitida)
And a interface não quebra (não é erro, é graceful degradation)
```

**Comportamento Esperado:**
- Validação de schema na resposta (verificar campos obrigatórios)
- Se campo obrigatório faltar, usar fallback (N/A, traço, ou omitir)
- Erro não é bloqueante (app continua funcionando com dados parciais)

---

### EC11: Previsão com Dados Incompletos

**Descrição:** Previsão de 5 dias retorna apenas 3 dias (Open-Meteo com problemas).

```gherkin
Given a Open-Meteo retorna apenas 3 dias de previsão (esperado 5)
When os dados são processados
Then:
  • Interface exibe apenas 3 cards (não força 5)
  • Ou exibe 3 cards + 2 cards com estado "N/A" (sem dados)
  • Label adiciona: "Previsão disponível para 3 dias"
And nenhum erro é mostrado ao usuário (graceful)
```

**Comportamento Esperado:**
- App se adapta a menos dias de previsão (não é erro crítico)
- Mensagem clara sobre limitação
- UI não quebra nem fica estranha com dados parciais

---

### EC12: Mudança de Timezone (Usuário Viajou)

**Descrição:** Usuário viajou; timezone local é diferente do que foi usado antes.

```gherkin
Given o usuário está em São Paulo (UTC-3)
When seleciona "São Paulo" e vê timestamp "Atualizado em 14:32"
And viaja para Tóquio (UTC+9)
When o app é aberto em Tóquio
Then:
  • Timestamp se ajusta ao timezone local: "Atualizado em 04:32" (próximo dia)
  • Datas de previsão respeitam novo timezone
  • Sem confusão de horários (ISO 8601 com timezone)
```

**Comportamento Esperado:**
- Timestamps sempre são baseados em ISO 8601 (com timezone explícito)
- Conversão local é feita no cliente (não no servidor)
- Datas de previsão nunca estão "invertidas" (respeitam cronologia)

---

### EC13: JSON Malformado da API

**Descrição:** Open-Meteo retorna JSON inválido ou corrompido.

```gherkin
Given a Open-Meteo retorna resposta malformada:
  "{ broken json [[ undefined"
When o app tenta fazer parse
Then a requisição falha (JSON.parse error)
And é exibida mensagem: "Erro ao processar dados. Tente novamente."
And nenhum erro de console crítico (tratado gracefully)
And cache anterior é usado (se disponível)
```

**Comportamento Esperado:**
- Try/catch captura erro de parse
- Mensagem amigável ao usuário
- Não quebra app (não há stack trace visível)

---

### EC14: Navegador Não Suporta localStorage

**Descrição:** localStorage está desabilitado (modo privado, configuração restritiva).

```gherkin
Given localStorage está desabilitado (modo privado, janela incógnito)
When o usuário alterna para "°F"
Then:
  • Toggle muda para "°F" normalmente (funciona)
  • localStorage.setItem() falha silenciosamente (try/catch)
  • Nenhuma mensagem de erro é mostrada
When o usuário fecha e reabre o app
Then:
  • Temperatura volta para Celsius (padrão, sem persistência)
  • Nenhum erro ou mensagem confusa
```

**Comportamento Esperado:**
- App funciona mesmo sem localStorage
- Preferência não é persistida (ok no MVP, sem banco de dados)
- Sem alertas ao usuário (transparente)

---

### EC15: Entrada Maliciosa — SQL Injection, XSS

**Descrição:** Usuário (ou ataque) digita código malicioso no campo de busca.

```gherkin
Given o usuário digita no campo de busca:
  • "<script>alert('xss')</script>"
  • "'; DROP TABLE cities; --"
  • "<img src=x onerror='alert(1)'>"
When a entrada é processada
Then:
  • Input é sanitizado/escapado (não interpretado como código)
  • Enviado para API como string literal: "%3Cscript%3E..." (URL encoded)
  • Nenhum script é executado
  • Sugestão pode aparecer (se Open-Meteo tiver uma cidade com esse nome)
```

**Comportamento Esperado:**
- Entrada de usuário é tratada como dados, nunca como código
- innerHTML é evitado; usar textContent ou template literals
- API recebe entrada sanitizada (URL encoded)

---

### EC16: Cidade com Múltiplas Resultados Idênticas

**Descrição:** Existem múltiplas cidades com o mesmo nome em países diferentes.

```gherkin
Given o usuário digita "Springfield"
When Open-Meteo retorna:
  • "Springfield, Illinois, USA"
  • "Springfield, Massachusetts, USA"
  • "Springfield, Missouri, USA"
When as sugestões são exibidas
Then cada uma mostra desambiguação clara:
  • "Springfield, IL 🇺🇸"
  • "Springfield, MA 🇺🇸"
  • "Springfield, MO 🇺🇸"
And ao clicar em "Springfield, IL", carrega clima correto (Illinois, não Missouri)
```

**Comportamento Esperado:**
- Sugestões incluem estado/país para desambiguação
- Ordenação por população/relevância (Springfield, IL é mais conhecida)
- Clima carrega para a cidade CORRETA (não há confusão)

---

### EC17: API Retorna Status 200 mas Dados Vazios

**Descrição:** Open-Meteo retorna HTTP 200, mas array de resultados está vazio.

```gherkin
Given a Open-Meteo retorna:
  {
    "results": []  # Vazio, mas sem erro HTTP
  }
When a resposta é processada
Then a interface exibe: "Nenhuma cidade encontrada"
And nenhuma sugestão aparece
And é idêntico ao comportamento de "cidade não encontrada"
```

**Comportamento Esperado:**
- Validação diferencia entre: erro HTTP vs. resultado vazio vs. resposta bem-sucedida
- Tratamento é consistente

---

### EC18: Debounce com Deleção Rápida

**Descrição:** Usuário digita rápido e depois apaga tudo muito rápido.

```gherkin
Given o usuário digita "são paulo" (8 caracteres em 500ms)
When então apaga tudo (delete 8 vezes em 200ms)
Then:
  • Requisição da primeira digitação pode ainda estar pendente
  • Se pendente, é cancelada (abortada) quando usuário apaga tudo
  • Campo fica vazio, sem sugestões
  • Nenhuma requisição "fantasma" chega à API
```

**Comportamento Esperado:**
- Debounce é cancelado se nova digitação vem antes de 300ms
- AbortController cancela requisições em voo (se input mudar)

---

## 7. Assumptions (Suposições)

## 7. Assumptions (Suposições)

### A1: Open-Meteo é Confiável

**Suposição:** Open-Meteo está disponível 99%+ do tempo  
**Validação:** Monitorar uptime; ter fallback se cair  
**Risco:** Se OpenMeteo sair do ar, app fica offline (mitigado com cache)

---

### A2: Usuários Preferem Digitar Nomes

**Suposição:** Busca por nome é suficiente; geolocalização é v2  
**Validação:** Analytics em v2 podem medir  
**Risco:** Alguns usuários querem GPS automático (v2 pode adicionar)

---

### A3: localStorage é Suficiente para MVP

**Suposição:** Salvar preferência C/F em localStorage é aceitável (sem backend)  
**Validação:** Usuários não precisam sincronizar entre devices no MVP  
**Risco:** v2 precisará de backend para sincronização

---

### A4: 5 Dias de Previsão é Adequado

**Suposição:** MVP com 5 dias; v2 pode expandir para 10+  
**Validação:** Atende personas Marina, Sofia e João (em parte)  
**Risco:** Usuários rurais podem pedir mais dias (v2)

---

### A5: Celsius é Padrão de Brasil

**Suposição:** Público-alvo primário usa Celsius; toggle para F é suficiente  
**Validação:** Brasil = Celsius; USA = Fahrenheit  
**Risco:** Nenhum (toggle sempre disponível)

---

### A6: React + TypeScript + Tailwind é Escolha Certa

**Suposição:** Stack confirmado é otimizado para MVP  
**Validação:** Já documentado em copilot-instructions.md  
**Risco:** Nenhum (fixo no projeto)

---

### A7: API Open-Meteo Retorna Dados Precisos

**Suposição:** Dados de Open-Meteo são confiáveis e recentes  
**Validação:** Comparação com INMET/observações  
**Risco:** Discrepâncias em casos específicos (v2 pode validar)

---

## 8. Risks (Riscos da Especificação)

| # | Risco | Probabilidade | Impacto | Mitigação |
|----|-------|---------------|---------|-----------|
| RS1 | Escopo 5 dias é insuficiente para agricultores | Média | Médio | Documentar como v2; coletar feedback em launch |
| RS2 | Performance ruim em 3G com bundle > 150KB | Média | Alto | Monitoring de bundle size em CI; code splitting agressivo |
| RS3 | Open-Meteo cai ou tem rate limit | Baixa | Alto | Cache 24h; retry exp. backoff; documentar SLA |
| RS4 | Usuários confundem C/F toggle | Baixa | Baixo | UI clara (ícone termômetro); documentação |
| RS5 | localStorage cheio ou desabilitado | Muito baixa | Baixo | Fallback silencioso; sem erro ao usuário |
| RS6 | Timezone confunde datas de previsão | Baixa | Médio | ISO 8601 timestamps; testes múltiplos timezones |

---

## 9. Out of Scope (MVP)

### Explicitamente NÃO Incluído no MVP

❌ **Autenticação & Contas**
- Sem login, cadastro, ou contas de usuário
- v2: autenticação para sincronização de favoritos

❌ **Favoritos Sincronizados**
- localStorage apenas (mesmo device)
- v2: backend + sincronização entre devices

❌ **Histórico de Busca**
- Não será armazenado
- v2: histórico com timestamp

❌ **Notificações & Alertas**
- Sem push notifications
- Sem alertas de tempestades
- v2: alerts personalizáveis

❌ **Geolocalização Automática**
- Sem GPS/geolocation
- Usuário digita cidade
- v2: sugerir cidade baseado em location

❌ **Previsão Horária**
- MVP = previsão diária
- v2: dados horários com gráficos

❌ **Múltiplas Cidades Comparadas**
- Uma cidade por vez
- v2: compare 2+ cidades lado-a-lado

❌ **Dados Adicionais Avançados**
- Sem índice UV, visibilidade, pressão (MVP)
- v2: dados estendidos

❌ **Personalizações Avançadas**
- Sem unidades customizáveis (vento sempre km/h)
- v2: preferência de unidades

❌ **Analytics & Tracking**
- Sem analytics de usuário
- v2: analytics agregado (sem PII)

❌ **i18n (Múltiplas Línguas)**
- MVP = pt-BR apenas
- v2: en-US, es-ES, etc.

---

## 10. Open Questions (Perguntas Resolvidas pela Discovery)

### Resolvidas ✅

✅ **Qual é a fonte de dados?**
- **Resposta:** Open-Meteo (sem API key, endpoints: /geocoding/search, /forecast)

✅ **"5 dias" significa o quê?**
- **Resposta:** Hoje + próximos 4 dias, exibição diária (não horária)

✅ **Qual é a unidade padrão?**
- **Resposta:** Celsius (com toggle para Fahrenheit sempre disponível)

✅ **Há autenticação ou banco de dados?**
- **Resposta:** Não. MVP stateless; localStorage apenas para preferência C/F

✅ **Qual é o idioma?**
- **Resposta:** Português Brasileiro (pt-BR)

---

### Pendentes para Plan Agent

⏳ **Qual é o volume esperado de usuários simultâneos?**
- **Resposta temporária:** Assumir 1K-10K usuários/dia para MVP
- **Validar em:** Plan Agent (infra)

⏳ **Qual é o SLA esperado?**
- **Resposta temporária:** 99% uptime (dependente de Open-Meteo)
- **Validar em:** Plan Agent (infra)

---

## 11. Critérios de Sucesso (Go/No-Go para Launch)

### Funcionais
- ✅ Busca de cidades funciona com 10+ cidades diferentes
- ✅ Clima atual exibe todos os 5 dados (temp, desc, ícone, umidade, vento)
- ✅ Previsão 5 dias exibe com datas corretas
- ✅ Toggle C/F funciona; preferência persiste
- ✅ Todos os critérios de aceite passam

### Performance
- ✅ Lighthouse score > 80 (3G throttle)
- ✅ Bundle size < 150KB (gzipped)
- ✅ Carregamento < 2 segundos

### Acessibilidade
- ✅ axe DevTools: 0 violações (nível de erro)
- ✅ Navegação por teclado: 100% funcional
- ✅ Teste com leitura de tela: compreensível

### Qualidade
- ✅ Cobertura de testes > 80%
- ✅ `pnpm lint` passa sem erros
- ✅ `pnpm build` produz assets
- ✅ `pnpm test` passa
- ✅ `pnpm test:e2e` fluxo crítico passa

### Documentação
- ✅ README com instruções de setup
- ✅ Política de Privacidade documentada
- ✅ Decisões arquitecturais documentadas

---

## APÊNDICE A: Mapeamento weather_code → UI (Open-Meteo)

| Código | Condição (EN) | Descrição pt-BR | Ícone | Cor (Tailwind) |
|--------|---------------|-----------------|-------|----------------|
| 0 | Clear sky | Céu limpo | ☀️ | yellow-400 |
| 1 | Mainly clear | Céu pouco nublado | ⛅ | yellow-300 |
| 2 | Partly cloudy | Parcialmente nublado | ⛅ | gray-300 |
| 3 | Overcast | Nublado | ☁️ | gray-500 |
| 45 | Foggy | Nevoeiro | 🌫️ | gray-600 |
| 48 | Depositing rime fog | Nevoeiro com geada | 🌫️ | gray-600 |
| 51-55 | Drizzle | Chuvisco | 🌦️ | blue-300 |
| 61-65 | Rain | Chuva | 🌧️ | blue-600 |
| 71-77 | Snow | Neve | ❄️ | blue-100 |
| 80-82 | Rain showers | Pancadas de chuva | ⛈️ | blue-700 |
| 85-86 | Snow showers | Pancadas de neve | ❄️ | blue-100 |
| 80-99 | Thunderstorm | Tempestade | ⚡ | purple-600 |

**Regra Fallback:** Se weather_code desconhecido → ☁️ + "Condição desconhecida"

---

## APÊNDICE B: Mensagens Padrão (pt-BR)

### Busca de Cidades

| Caso | Mensagem | Ação |
|------|----------|------|
| Cidade não encontrada | "Nenhuma cidade encontrada para '[termo]'\nTente outro nome ou verifique a ortografia" | Campo focado |
| Input vazio | (sem mensagem) | — |
| Input > 100 chars | (truncado automaticamente) | — |

### Clima Atual

| Caso | Mensagem | Duração |
|------|----------|---------|
| Carregando | (skeleton + spinner) | até dados |
| Sucesso | (dados exibidos) | — |
| Timeout (>8s) | "Problemas ao conectar. Tentando novamente..." | + spinner |
| Erro HTTP 5xx | "Erro ao carregar dados. Tente novamente." | + botão "Atualizar" |
| Rate limit (429) | "Muitos acessos. Aguarde alguns segundos." | + botão desabilitado |
| Offline (sem cache) | "Sem conexão de Internet" | — |
| 3 tentativas falhadas | "Não foi possível carregar. Tente novamente mais tarde." | + botão "Atualizar" |

### Cache/Offline

| Caso | Mensagem | Local |
|------|----------|-------|
| Dados < 24h | "Offline — dados de X horas atrás" | Acima do card |
| Dados ≥ 24h | Não exibe (expirado) | — |
| Sem dados + offline | "Sem conexão e sem dados salvos" | Center do card |

---

## APÊNDICE C: Model de Dados (TypeScript Types)

```typescript
// Open-Meteo Geocoding Response
interface GeocodeResult {
  id: number;
  name: string;
  latitude: number;
  longitude: number;
  elevation?: number;
  feature_code: 'PPL' | 'PPLC' | 'PPLA' | 'ADM1' | string;
  country_code: string;
  admin1?: string;
  admin1_id?: number;
  timezone: string;
  population?: number;
  country?: string;
}

// Open-Meteo Forecast Response
interface ForecastResponse {
  latitude: number;
  longitude: number;
  timezone: string;
  current: {
    time: string; // ISO 8601
    temperature: number;
    weather_code: number;
    relative_humidity: number;
    wind_speed_10m: number;
  };
  daily: {
    time: string[]; // ["2026-08-12", "2026-08-13", ...]
    weather_code: number[];
    temperature_2m_max: number[];
    temperature_2m_min: number[];
  };
}

// App Domain Models
interface CityData {
  id: number;
  name: string;
  admin1?: string;
  country: string;
  latitude: number;
  longitude: number;
  timezone: string;
  population?: number;
  countryCode: string;
}

interface CurrentWeather {
  temperature: number; // °C
  condition: string; // pt-BR
  weatherCode: number;
  humidity: number; // 0-100
  windSpeed: number; // km/h
  updatedAt: string; // ISO 8601
}

interface DayForecast {
  date: string; // YYYY-MM-DD
  tempMax: number; // °C
  tempMin: number; // °C
  condition: string; // pt-BR
  weatherCode: number;
}

interface AppState {
  selectedCity: CityData | null;
  currentWeather: CurrentWeather | null;
  forecast5days: DayForecast[]; // sempre length 5 ou []
  temperatureUnit: 'celsius' | 'fahrenheit';
  isLoading: boolean;
  error: string | null;
  isOffline: boolean;
}

interface CacheEntry {
  city: CityData;
  current: CurrentWeather;
  forecast: DayForecast[];
  savedAt: string; // ISO 8601
  expiresAt: string; // ISO 8601 (savedAt + 24h)
}
```

---

## APÊNDICE D: Layout Principal (Wireframe)

### Mobile (320px-768px)
```
┌──────────────────────────┐
│ Weather App      🌍 |°C  │ ← Header
├──────────────────────────┤
│ 🔍 Buscar cidades        │ ← Search + dropdown
│   ├─ São Paulo, SP 🇧🇷   │
│   ├─ São Gonçalo, RJ 🇧🇷 │
│   └─ São Leopoldo, RS 🇧🇷│
├──────────────────────────┤
│ 📍 São Paulo, SP 🇧🇷     │ ← Current weather card
│ ☀️ Céu limpo             │
│ 28.5°C                   │
│ Umidade: 65% Vento: 12km/h
│ 🔄 Atualizado em 14:32   │
├──────────────────────────┤
│ Previsão (scroll ←→):     │ ← Forecast carousel
│ ┌─────┬─────┬─────┐      │
│ │12ago│13ago│14ago│  ...│
│ │☀️ 32│🌧️ 30│☁️ 28│      │
│ │22   │20   │18   │      │
│ └─────┴─────┴─────┘      │
└──────────────────────────┘
```

### Desktop (1024px+)
```
┌─────────────────────────────────────┐
│ Weather App                  🌍 |°C │
├──────────────┬──────────────────────┤
│ 🔍 Buscar    │ 📍 São Paulo, SP 🇧🇷│
│              │ ☀️ Céu limpo         │
│ Sugestões... │ 28.5°C               │
│              │ Umidade: 65%         │
│              │ Vento: 12 km/h       │
│              │ 🔄 Atualizado 14:32 │
├──────────────────────────────────────┤
│ Previsão (5 cards full-width):       │
│ ┌─────┬─────┬─────┬─────┬─────┐    │
│ │12ago│13ago│14ago│15ago│16ago│    │
│ │☀️ 32│🌧️ 30│☁️ 28│🌦️ 31│☀️ 29│   │
│ │22   │20   │18   │21   │19   │    │
│ └─────┴─────┴─────┴─────┴─────┘    │
└─────────────────────────────────────┘
```

---

## APÊNDICE E: Retry Strategy (Exponential Backoff)

**Configuração:**
- Timeout por requisição: 8 segundos
- Máximo de tentativas: 3
- Delays: [1s, 2s, 4s]

**Timeline:**
```
t=0s:   1ª requisição enviada
t=8s:   Timeout, requisição abortada
t=9s:   Aguardando 1 segundo (1º retry)
t=10s:  2ª requisição enviada
t=18s:  Timeout, requisição abortada
t=20s:  Aguardando 2 segundos (2º retry)
t=22s:  3ª requisição enviada
t=30s:  Timeout, requisição abortada
t=30s:  Mensagem "Não foi possível carregar..."
```

**Pseudocódigo:**
```typescript
async function fetchWithRetry(lat, lng, attempt = 0) {
  try {
    const controller = new AbortController();
    const timeout = setTimeout(() => controller.abort(), 8000);
    
    const response = await fetch(
      `https://api.open-meteo.com/v1/forecast?latitude=${lat}&longitude=${lng}`,
      { signal: controller.signal }
    );
    clearTimeout(timeout);
    
    if (response.ok) return await response.json();
    if (response.status === 429) return null; // rate limit, sem retry
    throw new Error(`HTTP ${response.status}`);
  } catch (err) {
    if (attempt < 3) {
      const delay = [1000, 2000, 4000][attempt];
      await new Promise(r => setTimeout(r, delay));
      return fetchWithRetry(lat, lng, attempt + 1);
    }
    throw err;
  }
}
```

---

## APÊNDICE F: Design System (Dark Glassmorphism)

### Paleta de Cores

```css
:root {
  /* Background */
  --bg-primary: #0f1419;      /* rgb(15, 20, 25) */
  --bg-secondary: #1a1f2e;    /* rgb(26, 31, 46) */
  --bg-tertiary: #252b3b;     /* rgb(37, 43, 59) */
  
  /* Glass effect */
  --glass-bg: rgba(26, 31, 46, 0.8);
  --glass-blur: 16px;
  --glass-border: 1px solid rgba(255, 255, 255, 0.1);
  
  /* Text */
  --text-primary: #ffffff;           /* rgb(255, 255, 255) */
  --text-secondary: #b0b0b0;         /* rgb(176, 176, 176) */
  --text-tertiary: #808080;          /* rgb(128, 128, 128) */
  
  /* Accent (Tailwind) */
  --accent-primary: #3b82f6;         /* blue-500 */
  --accent-secondary: #fbbf24;       /* amber-400 */
}
```

### Components

```css
/* Glass Card */
.glass-card {
  background: var(--glass-bg);
  backdrop-filter: blur(var(--glass-blur));
  border: var(--glass-border);
  border-radius: 1rem;
  padding: 1rem;
  box-shadow: 0 10px 30px rgba(0, 0, 0, 0.5);
}

/* Button Primary */
.btn-primary {
  background: var(--accent-primary);
  color: white;
  border-radius: 0.5rem;
  padding: 0.5rem 1rem;
  min-width: 44px; /* touch-friendly */
  min-height: 44px;
}

/* Input Field */
input {
  background: var(--bg-secondary);
  color: var(--text-primary);
  border: 1px solid var(--text-secondary);
  border-radius: 0.5rem;
  padding: 0.75rem;
}
```

### Resposta a Preferências

```css
@media (prefers-color-scheme: light) {
  :root {
    --bg-primary: #f8f9fa;
    --text-primary: #0f1419;
  }
}

@media (prefers-reduced-motion: reduce) {
  * {
    animation: none !important;
    transition: none !important;
  }
}
```

---

**Versão:** 1.0 (Production-Ready)  
**Próximo:** Plan Agent → `plans/weather-app-plan.md`  
**Status:** ✅ Pronto para Plan Agent
