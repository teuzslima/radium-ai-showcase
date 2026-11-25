# Decisões técnicas (resumo das ADRs)

As decisões completas ficam no repositório privado, em `docs/adr/`. Aqui vai o resumo do que foi
decidido e por quê.

## 1. Monólito modular, e não microsserviços
**Contexto:** um único desenvolvedor, uma cliente no piloto, cerca de 430 exames por dia.
**Decisão:** uma API FastAPI dividida em módulos (`auth`, `exams`, `templates`, `metrics`…),
PostgreSQL e um único serviço separado para a geração com IA, que roda em segundo plano.
**Por quê:** menos coisas para operar e testar. Os módulos têm fronteiras claras para separar
depois, se houver mais clientes.

## 2. Extensão do Chrome em vez de integração com a plataforma
**Contexto:** a plataforma de terceiros não exporta exames, e não deve haver automação sobre ela.
**Decisão:** extensão Manifest V3 com painel lateral. A captura só acontece por ação da usuária
e só na aba do visualizador configurado. Nada é preenchido automaticamente na plataforma: o laudo
aprovado é copiado e colado por ela.
**Descoberta:** `captureVisibleTab` exige `activeTab` ou `<all_urls>`. A permissão do domínio
não basta (testado em Chromium). Por isso o padrão é o atalho de teclado, que concede
`activeTab` sem permissão ampla. O botão é opcional e mediante permissão explícita.

## 3. A IA preenche as máscaras da médica
**Contexto:** a médica tem máscaras próprias (frases padrão, achados opcionais com `*`,
alternativas com `/`, lacunas). Um chatbot genérico escreve no próprio estilo e pode inventar.
**Decisão:**
- as máscaras são lidas por um parser e enviadas numeradas à IA;
- a IA devolve frases estruturadas, cada uma com origem (linha da máscara ou texto novo),
  confiança, uma lista de não avaliável e alertas;
- o servidor valida a resposta antes de montar o texto.

**Por quê:** o laudo fica rastreável frase a frase até a máscara. Isso reduz invenções e mede a
taxa de correção.

## 4. Validação pós-IA no servidor
**Contexto:** nos testes com imagens reais, o modelo às vezes incluiu frases de normalidade sobre
estruturas que ele mesmo disse não conseguir avaliar.
**Decisão:**
- essas frases são retiradas do laudo e viram alerta;
- frases que contradizem alertas viram ponto de atenção;
- a técnica não confirmada (uso de contraste) entra na lista de conferência.

**Por quê:** regras no código são determinísticas e testáveis. O prompt sozinho não garante isso.

## 5. Provedores de IA intercambiáveis e modelo por modalidade
**Decisão:**
- uma camada de provedor (Claude, ou um provedor de teste sem custo), com modelo e esforço
  configuráveis por modalidade;
- padrão: Haiku no raio-X e Sonnet em ressonância e tomografia, com esforço baixo;
- a parte estável do pedido (máscara e exemplos) vai antes das imagens, com cache de prompt.

**Resultado medido:** projeção mensal de cerca de US$ 140 com Sonnet em tudo contra cerca de
US$ 20 na configuração padrão. O cache sozinho reduz uns 35% a partir do segundo exame do mesmo
tipo. A escolha final depende da avaliação às cegas pela radiologista, numa ferramenta de
comparação que gera rascunhos identificados só por letras.

## 6. Contra a aprovação automática
**Contexto:** com 400 exames por dia, o maior risco é aprovar um erro da IA por pressa.
**Decisão:**
- os pontos de atenção precisam ser conferidos antes de aprovar;
- casos "em atenção" vão para o topo da fila;
- a similaridade entre o rascunho e o laudo final é registrada.

## 7. Privacidade por padrão
**Decisão:**
- dados do paciente ocultos no visualizador antes da captura;
- imagens cifradas e apagadas após 30 dias;
- backups só do banco;
- 2FA obrigatório e auditoria completa;
- API local (127.0.0.1) ou atrás de proxy com HTTPS.
