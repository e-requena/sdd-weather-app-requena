# Discovery — Weather App

**Data:** 2026-08-12  
**Status:** Análise Inicial  
**Preparado para:** Fase de Especificação (Spec Agent)

---

## 1. Contexto

A empresa solicitou uma aplicação web de previsão do tempo que permita aos usuários:
- Consultar informações climáticas de qualquer cidade
- Visualizar condições atuais e previsão estendida
- Adaptar a exibição de temperatura conforme preferência
- Acessar via dispositivos móveis com experiência otimizada

**Escopo primário:** MVP (Minimum Viable Product) com funcionalidades essenciais.  
**Público-alvo:** Usuários finais que necessitam de informações rápidas e confiáveis sobre o clima.

---

## 2. Requisitos Funcionais

### Nível Crítico (Must-Have)

1. **Buscar Cidades**
   - O usuário deve poder digitar o nome de uma cidade
   - A aplicação deve retornar uma lista de cidades correspondentes
   - Suportar autocompletar ou dropdown com sugestões
   - Suportar buscas em português, inglês e caracteres acentuados

2. **Exibir Clima Atual**
   - Após selecionar uma cidade, exibir:
     - Temperatura atual
     - Descrição do clima (ex: "Céu limpo", "Nublado")
     - Ícone visual representando a condição
     - Umidade relativa do ar
     - Velocidade do vento
   - Atualizar dados em tempo real (ou refresh manual)

3. **Exibir Previsão de 5 Dias**
   - Mostrar previsão diária com:
     - Data
     - Temperatura máxima e mínima
     - Condição climática
     - Ícone representativo
   - Expandível/colapsável para não sobrecarregar a interface

4. **Alternar Entre Celsius e Fahrenheit**
   - Botão/toggle na interface para alternar unidade
   - Conversão automática de todos os valores exibidos
   - Persistir preferência do usuário (opcional, vide perguntas em aberto)

### Nível Desejável (Should-Have)

5. **Cidades Favoritas** (decisão pendente)
   - Permitir salvar cidades consultadas frequentemente
   - Atalho para acesso rápido

6. **Histórico de Busca** (decisão pendente)
   - Listar últimas cidades pesquisadas

7. **Notificações de Alerta** (decisão pendente)
   - Avisos de condições severas (tempestades, frio extremo, etc.)

---

## 3. Requisitos Não-Funcionais

### Performance
- **Tempo de carregamento:** < 2 segundos para busca de cidade
- **Tempo de carregamento:** < 2 segundos para dados climáticos
- **Responsividade:** Interface reage a entrada do usuário em < 100ms

### Usabilidade & Acessibilidade
- **WCAG 2.1 AA:** Conformidade com diretrizes de acessibilidade
- **Navegação por teclado:** 100% funcional (Tab, Enter, Esc)
- **Contraste:** Mínimo 4.5:1 para texto em relação ao fundo
- **Labels semânticas:** Todos os elementos interativos têm rótulos ARIA
- **Leitores de tela:** Aplicação funcional com NVDA, JAWS, VoiceOver

### Assistive Technologies
- **Leitores de tela:** Compatível com NVDA (Windows), JAWS, VoiceOver (Mac/iOS), TalkBack (Android)
- **Modo alto contraste:** Funcional em Windows High Contrast Mode
- **Zoom:** Totalmente responsivo até 200% de zoom
- **Redução de movimento:** Respeitar `prefers-reduced-motion` (sem animações se desabilitado)
- **Tamanho de fonte:** Ajustável pelo usuário (até 18px)
- **Dark mode nativo:** Respeitar `prefers-color-scheme` (dark/light)

### Responsividade
- **Mobile:** Funcional em telas de 320px até 768px
- **Tablet:** Otimizado para 768px a 1024px
- **Desktop:** Layout full-width para telas > 1024px
- **Orientação:** Suportar portrait e landscape

### Confiabilidade
- **Taxa de uptime:** Mínimo 99% (dependente da disponibilidade da API Open-Meteo)
- **Tratamento de erros:** Mensagens amigáveis em caso de falha
- **Retry automático:** Tentar reconectar em caso de timeout

### Segurança
- **HTTPS:** Todas as comunicações criptografadas
- **Validação:** Todos os inputs validados no cliente e (se houver backend) no servidor
- **XSS Prevention:** Sanitizar dados da API antes de exibir
- **Rate limiting:** Respeitar limites da API Open-Meteo

### Compatibilidade de Navegadores
- **Chrome/Chromium:** Versão 90+
- **Firefox:** Versão 88+
- **Safari:** Versão 14+
- **Edge:** Versão 90+
- **Mobile:** iOS Safari 14+, Chrome Android 90+
- **Fallback:** Navegadores antigos devem receber mensagem informativa (sem crash)
- **Testing:** Validar em desktop (Windows, macOS, Linux) e mobile (iOS, Android)

### Conformidade Legal & Privacidade
- **LGPD (Brasil):** Consentimento explícito para geolocalização (se aplicável)
- **GDPR (Europa):** Conformidade se acessado de EU
- **Política de Privacidade:** Documentar dados coletados (buscas, preferências)
- **localStorage:** Informar ao usuário que dados são armazenados localmente
- **Sem rastreamento externo:** Nenhum tracking de terceiros sem consentimento
- **Cookie banner:** Se aplicável, implementar de forma LGPD/GDPR compliant
- **Dados técnicos:** Não logar informações sensíveis (senhas, dados pessoais)

### Monitoramento, Logging & Observabilidade
- **Error Tracking:** Capturar erros com stack trace e contexto (ex: Sentry)
- **Performance Monitoring:** Medir Core Web Vitals (LCP, FID, CLS, TTFB)
- **API Latency:** Registrar tempo de resposta de Open-Meteo
- **User Sessions:** Rastrear fluxos de busca (agregado, sem PII)
- **Alertas:** Notificar equipe se error rate > 5% ou uptime < 99%
- **Retenção de logs:** Mínimo 30 dias de retenção
- **Conformidade:** Não logar senhas, IPs completos, ou dados sensíveis
- **Source maps:** Disponíveis em staging/production para debugging

### Tecnologia
- **Stack confirmado:** React + TypeScript + Tailwind CSS
- **Compilação:** Vite
- **Testes:** Vitest (unit) + Playwright (E2E)
- **Lint:** Biome
- **Banco de dados:** Não aplicável (MVP stateless)

---

## 4. Riscos Identificados

| # | Risco | Probabilidade | Impacto | Mitigação |
|----|-------|---------------|---------|-----------|
| R1 | API Open-Meteo indisponível ou lenta | Média | Alto | Cache local, mensagem de erro clara, retry automático |
| R2 | Nomes de cidades ambíguos (ex: "Springfield") | Média | Médio | Mostrar país/estado, permitir refinamento |
| R3 | Dados de API inconsistentes (ex: falta de ícones) | Baixa | Médio | Validação de schema, fallback para valores padrão |
| R4 | Performance ruim em conexões 3G | Média | Alto | Lazy loading, compressão, otimizar bundle React |
| R5 | Conversão C/F com arredondamento incorreto | Baixa | Baixo | Testes unitários rigorosos, validação manual |
| R6 | Escopo creep (favoritos, histórico, notificações) | Alta | Alto | Documentar explicitamente o Out of Scope |

---

## Decisões Estratégicas (Decision Gates)

**Data:** 2026-08-12  
**Status:** Aprovado  
**Impacto:** Desbloqueia Spec Agent; reduz 5+ riscos críticos

---

### D1: Fonte de Dados — Open-Meteo (sem API Key)

**Decisão:** Usar **Open-Meteo** como única fonte de dados climáticos (sem API key)

**Justificativa:**
- ✅ Gratuita e sem autenticação
- ✅ Já documentada na suposição A11
- ✅ Suporta geocoding e forecast em uma única API
- ✅ Reduz complexidade de integração

**O que resolve:**
- ✓ R1 (API indisponibilidade) → Escolha explícita; fallback e cache são responsabilidade de implementação
- ✓ OQ #6 (Fonte de dados de ícones?) → Open-Meteo fornece códigos de clima; mapearemos para SVGs locais

**Consequências:**
- ⚠️ Se Open-Meteo cair: app fica offline (mitigado com cache 24h)
- ⚠️ Limite de requisições: ~10K/dia (suficiente para MVP)

**Próximas ações:**
- Validar endpoints: `/geocoding/search` e `/forecast` na Spec
- Documentar fallback + retry strategy

---

### D2: Escopo Temporal — "5 Dias" = Hoje + 4 Dias

**Decisão:** Previsão cobrir **5 dias** (hoje + próximos 4 dias) com **granularidade diária**

**Justificativa:**
- ✅ Atende Marina (commuter) e Sofia (mãe planejadora)
- ✅ Suficiente para João (agricultor) em MVP
- ✅ Open-Meteo retorna até 16 dias; 5 é um bom MVP
- ✅ Escopo claro para diferente de v2

**O que resolve:**
- ✓ R2 (Escopo "5 dias" indefinido) → **RESOLVIDO**: 5 dias = 120 horas, exibição diária
- ✓ OQ #4 (Previsão horária vs. diária?) → MVP = diária; horária é v2

**Consequências:**
- ⚠️ João (agricultor) pode pedir 10 dias em feedback
- ⚠️ Usuários em tropical zones querem ver mais dias (rainy season)

**Próximas ações:**
- Documentar na Spec: "Previsão diária de 5 dias (hoje + D+1 a D+4)"
- Criar issue v2 para "Previsão estendida (10+ dias)"

---

### D3: Unidade de Temperatura Padrão — Celsius

**Decisão:** Unidade padrão é **Celsius**; toggle para Fahrenheit sempre disponível

**Justificativa:**
- ✅ Brasil (público-alvo primário) usa Celsius
- ✅ ISO 8601 padrão internacional é Celsius
- ✅ Toggle C↔F em localStorage (opcional na Spec, mas implementaremos)

**O que resolve:**
- ✓ R8 (Conversão C/F imprecisa) → Decisão sobre padrão elimina ambiguidade
- ✓ OQ #5 (Dados adicionais? Conversão de unidades?) → Conversão está IN SCOPE
- ✓ R29 (Unidade de vento não definida) → Vento em km/h (Brasil)

**Consequências:**
- ⚠️ Usuários em USA podem não perceber o toggle de imediato
- ⚠️ Vento em km/h, não m/s ou mph

**Próximas ações:**
- Documentar na Spec: "Padrão Celsius; toggle salva em localStorage"
- Conversão: F = (C × 9/5) + 32 (unit tests rigorosos)
- Vento: sempre km/h (sem conversão; MVP simplificado)

---

### D4: Persistência e Autenticação — Nenhuma

**Decisão:** **Sem autenticação** e **sem persistência em servidor** no MVP

**Justificativa:**
- ✅ Reduz complexity significativamente (nenhum backend necessário)
- ✅ MVP stateless = melhor para performance
- ✅ Preferência do usuário (C/F) salva em localStorage apenas
- ✅ Conforma com suposição A31

**O que resolve:**
- ✓ R10 (LGPD: localStorage = PII?) → localStorage apenas para preferência C/F, não identificável
- ✓ OQ #1 (Favoritos/Histórico?) → OUT OF SCOPE em MVP; localStorage se adicionado em v2
- ✓ OQ #9 (Analytics?) → Analytics agregado (sem PII) é OUT OF SCOPE

**Consequências:**
- ⚠️ Usuários não conseguem sincronizar favoritos entre devices
- ⚠️ Sem histórico persistido
- ⚠️ v2 precisará de backend se implementar favoritos/sincronização

**Próximas ações:**
- Documentar na Spec: "MVP sem autenticação; localStorage apenas para preferência C/F"
- Criar issue v2: "Sincronização de favoritos com backend (requer autenticação)"
- LGPD: informar usuário que localStorage armazena preferência (cookie banner opcional)

---

### D5: Idioma da Interface — Português Brasileiro (pt-BR)

**Decisão:** Toda a interface em **português brasileiro** (pt-BR)

**Justificativa:**
- ✅ Brasil é mercado primário (público-alvo)
- ✅ Dados de API (Open-Meteo) retornam em en-US; localizaremos na UI
- ✅ Descrição de clima traduzida (ex: "Céu limpo", "Nublado", não "Clear sky")
- ✅ Conforma com copilot-instructions.md (documentação em pt-BR)

**O que resolve:**
- ✓ R17 (SEO não mencionado) → pt-BR significa melhor ranking em buscas Brasil
- ✓ OQ #13 (i18n?) → MVP = pt-BR apenas; i18n é v2

**Consequências:**
- ⚠️ Usuários não-brasileiros verão interface em português
- ⚠️ Sem i18n framework; adição de en-US requer refactor
- ⚠️ Cidades e descrições traduzidas manualmente

**Próximas ações:**
- Documentar na Spec: "Toda interface em pt-BR"
- Criar arquivo de traduções (pt-BR.json): clima, labels, mensagens de erro
- Criar issue v2: "i18n support (en-US, es-ES, etc.)"

---

## Matriz de Impacto: Decisões vs. Riscos & Perguntas

| Decisão | Riscos Resolvidos | Open Questions Resolvidas | Novos Riscos Mitigados |
|---------|-------------------|---------------------------|------------------------|
| **D1: Open-Meteo** | R1, R3 | OQ #6 | R1 (fallback + cache) |
| **D2: 5 Dias** | R2 | OQ #4 | R27 (v2 para 10 dias) |
| **D3: Celsius** | R8, R29 | OQ #5 | Nenhum |
| **D4: Sem Auth** | R10 | OQ #1, OQ #9 | R10 (localStorage LGPD) |
| **D5: pt-BR** | R17 | OQ #13 | Nenhum |

---

## Status: Decisões Críticas

✅ **R2** (Escopo 5 dias) — **RESOLVIDO** por D2  
✅ **R3** (Mobile vago) — **PENDENTE** (será na Spec)  
✅ **R6** (Volume usuários) — **PENDENTE** (será na Spec; usar defaults 1K-10K/dia)  
✅ **R10** (LGPD) — **RESOLVIDO** por D4 (localStorage não-identificável)  
✅ **R20** (SLA) — **PENDENTE** (será na Spec; default 99%)  
✅ **R24** (Política Privacidade) — **RESOLVIDO** (usar template LGPD para localStorage)

**Próximo passo:** Com estas 5 decisões, o **Spec Agent** pode produzir `specs/weather-app-spec.md` com confiança. ✓

## 5. Perguntas em Aberto (Open Questions)

### Funcionalidade

1. **Favoritos/Histórico?**
   - Devem ser salvos localmente (localStorage)?
   - Ou sincronizados com um servidor (requer autenticação)?

2. **Limite de cidades na busca?**
   - Quantas sugestões mostrar no dropdown?
   - Qual critério de ordenação (população, relevância, distância)?

3. **Atualização de dados?**
   - Atualizar automaticamente a cada X minutos?
   - Ou apenas quando o usuário clicar "Atualizar"?

4. **Previsão horária vs. diária?**
   - MVP será apenas diária (5 dias)?
   - Versão futura com previsão horária?

5. **Dados adicionais?**
   - Índice UV, visibilidade, pressão atmosférica?
   - Ou apenas temperatura, condição e vento?

### Técnica

6. **Fonte de dados de ícones?**
   - Open-Meteo fornece URLs de ícones?
   - Ou criaremos SVGs locais mapeados por código da API?

7. **Geolocalização do usuário?**
   - Usar GPS do dispositivo para sugerir cidade inicial?
   - Ou começar sempre em branco?

8. **Offline mode?**
   - Service Worker para funcionar offline com último dado?
   - Ou requer internet obrigatoriamente?

### Negócio

9. **Analytics?**
   - Rastrear cidades mais pesquisadas?
   - Rastrear erros de API?

10. **Hospedagem & Deploy?**
    - Vercel? GitHub Pages? Servidor próprio?
    - CI/CD pipeline?

---

## 6. Suposições

### Técnicas

- **A11. A API Open-Meteo é gratuita e não requer autenticação**
  - Confirmado: Sim, endpoints `/geocoding/search` e `/forecast`

- **A12. Vamos usar React 19.2.7 + TypeScript strict**
  - Confirmado: Definido em package.json e copilot-instructions.md

- **A13. Tailwind CSS com tema dark e glassmorphism**
  - Confirmado: Definido em tailwind.config.js

- **A14. Sem persistência de dados no backend (MVP)**
  - Suposição: Dados salvos apenas em localStorage ou session storage

### Funcionais

- **A21. Busca por nome de cidade (não por coordenadas)**
  - Suposição: Usuários preferem digitar nomes

- **A22. Uma única cidade por vez na tela principal**
  - Suposição: Não há comparação de múltiplas cidades no MVP

- **A23. Moeda padrão: Celsius (com toggle para Fahrenheit)**
  - Suposição: Brasil usa Celsius

### Negócio

- **A31. MVP sem login ou autenticação**
  - Suposição: Público anônimo, sem conceito de "usuários salvos"

- **A32. Prazo realista: 2-3 semanas de desenvolvimento**
  - Suposição: Equipe de 1-2 engenheiros em tempo integral

---

## Próximos Passos

1. ✅ **Esta análise (Discovery)** — concluída
2. ⏳ **Spec Agent** → Transformar discovery em especificação formal (`specs/weather-app-spec.md`)
3. ⏳ **Validação com stakeholders** → Confirmar riscos e perguntas abertas
4. ⏳ **Resolução de Open Questions** → Definir decisões antes de planejar técnico

---

**Versão:** 1.0  
**Próxima revisão:** Após feedback do Spec Agent