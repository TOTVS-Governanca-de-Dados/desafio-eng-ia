# desafio-eng-ia

## PokéGarden — agentes, dados e descoberta de ferramentas

Desafio de **Engenharia de IA** para apresentação e discussão em uma entrevista técnica, com foco no trabalho de engenharia de agentes do Garden.

Construa um assistente que responda perguntas sobre Pokémon usando o **OpenAI Agents SDK para Python**, dados da **PokéAPI** e um **catálogo extensível de tools**. O assistente deve descobrir as ferramentas necessárias, consultar dados verificáveis e explicar sua resposta com fontes.

Você terá **uma semana (7 dias corridos) a partir do recebimento do desafio** para desenvolver a solução e preparar sua apresentação. Na entrevista, vamos explorar o funcionamento da aplicação, as decisões tomadas, os testes e as possibilidades de evolução.

Este repositório contém o enunciado. A implementação é responsabilidade da pessoa candidata. Não é necessário ter acesso ao código, às credenciais ou aos ambientes internos do Garden.

## 1. Contexto e objetivo

Imagine que o Garden precisa incorporar uma nova fonte de conhecimento. Hoje ela é a PokéAPI; amanhã pode ser uma API corporativa. A plataforma precisa conseguir registrar capacidades, descobrir quais atendem a uma solicitação e executá-las com contratos claros, limites e rastreabilidade.

Sua missão é entregar a **PokéGarden API**, capaz de:

1. Coletar, normalizar e reutilizar dados públicos de Pokémon.
2. Responder em português a perguntas em linguagem natural.
3. Criar e executar tools com entradas e saídas validadas.
4. Consultar um catálogo para descobrir tools que não estavam disponíveis no contexto inicial do agente.
5. Incorporar uma nova tool sem alterar o núcleo de orquestração.
6. Demonstrar acertos, falhas e limitações por meio de testes, avaliações e registros de execução.

## 2. Prazo, apresentação e escopo

- **Prazo de entrega:** uma semana (7 dias corridos), contada a partir do recebimento do desafio. Envie o repositório e o commit da entrega até o fim desse prazo.
- **Organização do trabalho:** distribua o desenvolvimento ao longo da semana e priorize o fluxo principal. Informe o tempo aproximado utilizado e as decisões de priorização.
- **Entrevista técnica:** a apresentação acontecerá na data agendada com o time. Prepare uma demonstração inicial de até 15 minutos; depois, vamos discutir a implementação, navegar pelo código e explorar suas decisões. Uma pequena alteração na solução pode fazer parte da dinâmica.
- Se algo ficar incompleto, apresente o que funciona e descreva as pendências. Identifique claramente o que foi executado, simulado ou ainda não implementado.

A expectativa é um protótipo local bem testado, com implementações simples e proporcionais ao escopo.

**Obrigatório:** Python 3.12+, OpenAI Agents SDK (`openai-agents`), FastAPI, validação de contratos e testes automatizados. Escolha as demais bibliotecas e fixe versões de forma reproduzível.

Use `Agent` e `Runner` no fluxo real, com function tools do SDK (`@function_tool` ou construção equivalente). O SDK deve executar a orquestração; apenas instalar a dependência ou fazer uma chamada isolada ao modelo não atende ao requisito. Essas interfaces estão descritas na [documentação de agentes](https://developers.openai.com/api/docs/guides/agents/define-agents) e de [tools no Agents SDK](https://developers.openai.com/api/docs/guides/tools#usage-in-the-agents-sdk).

O modelo deve ser configurável por `OPENAI_MODEL`, com credencial em `OPENAI_API_KEY`. Documente um modelo compatível com a versão escolhida e a razão da escolha. Não há modelo específico ou volume mínimo de gasto exigido.

Frontend, deploy público, Kubernetes, banco vetorial, RAG documental, MCP, streaming e múltiplos agentes são opcionais. Um agente bem projetado e armazenamento local são suficientes. Priorize uma solução que você consiga executar, explicar e discutir durante a entrevista.

## 3. Funcionalidades obrigatórias

### 3.1. Coleta e normalização da PokéAPI

Use a API REST pública `https://pokeapi.co/api/v2/` como fonte dos fatos sobre Pokémon. Ela oferece acesso por GET sem autenticação; sua política de uso pede cache local e consumo responsável. Consulte a [documentação e a política de uso](https://pokeapi.co/docs/v2#info).

Implemente um cliente HTTP reutilizável, separado da lógica do agente, que:

- Busque Pokémon por nome canônico ou ID e normalize espaços e caixa nas entradas.
- Retorne pelo menos ID, nome, tipos, habilidades, altura, peso e os seis atributos base. Identifique as unidades; se convertê-las, teste a conversão.
- Guarde os recursos consultados em cache local persistente, por exemplo arquivos ou SQLite, com timestamp e política de expiração documentada.
- Reutilize o cache em consultas repetidas e após reiniciar a aplicação. O cache de dados públicos pode ser compartilhado; histórico de usuários e permissões de tools não devem ser confundidos com esse cache.
- Preserve a URL de origem e a data de coleta. Um cache hit deve manter a data da coleta original.
- Trate recurso inexistente, timeout, falha de conexão, resposta inválida e indisponibilidade da origem sem inventar dados.

Não é necessário baixar toda a Pokédex. Faça coleta sob demanda e limite lotes. Se optar por listar recursos, respeite a [paginação da PokéAPI](https://pokeapi.co/docs/v2#resource-listspagination-section). Não é exigida busca aproximada, tradução de nomes ou suporte completo a formas regionais; documente o comportamento para entradas ambíguas.

Fatos específicos da resposta devem vir de uma consulta ou de um registro de cache identificável. Conhecimento prévio do modelo pode ajudar a interpretar o pedido, mas não substitui a coleta. Fixtures são permitidas em testes, nunca apresentadas como consultas reais.

### 3.2. Tools de domínio

Implemente as quatro capacidades abaixo. Os nomes são sugeridos; nomes equivalentes são aceitos se documentados.

| Tool | Entrada mínima | Comportamento esperado |
| --- | --- | --- |
| `get_pokemon` | Nome ou ID | Retornar os dados normalizados de um Pokémon e sua proveniência. |
| `compare_pokemon` | De 2 a 6 nomes ou IDs | Comparar tipos e atributos base; calcular diferenças e soma dos atributos em código; explicitar empates e dados ausentes. |
| `get_type_matchups` | Nome ou ID de um Pokémon | Calcular fraquezas, resistências e imunidades defensivas a partir dos tipos consultados; combinar os efeitos quando houver dois tipos. |
| `get_evolution_chain` | Nome ou ID de um Pokémon | Consultar sua espécie e a cadeia de evolução referenciada; apresentar as ramificações e as condições disponíveis, preservando valores ausentes. Esta é a tool da prova de extensão da seção 4. |

Use os recursos `pokemon`, `type`, `pokemon-species` e `evolution-chain` conforme os contratos da [PokéAPI](https://pokeapi.co/docs/v2). Não presuma que IDs de Pokémon, espécie e cadeia de evolução são iguais.

Cada tool deve ter descrição útil ao modelo, schema de entrada, validação executada no servidor, retorno estruturado e erros distinguíveis. Defina como representar sucesso, ausência de dados e falha. Evite devolver todo o JSON externo quando apenas poucos campos são necessários.

**Regras de domínio para este desafio:**

- Comparações devem ter um critério explícito. “Maior soma de atributos base” não deve ser apresentado como garantia de vitória em batalha.
- Ao comparar três ou mais Pokémon, documente a referência usada para calcular diferenças; não é necessário calcular todos os pares.
- Matchups consideram apenas as relações de tipos atuais retornadas pela API. Desconsidere habilidades, itens, clima, golpes, STAB e regras históricas; informe essa simplificação.
- Evoluções devem preservar a árvore e suas condições. Não transforme ramificações em uma sequência linear nem invente um nível quando ele não estiver informado.
- Ausência de texto em português deve ter fallback documentado. Uma tradução produzida pelo modelo deve ser identificada como tradução.

### 3.3. Catálogo e descoberta de tools

Implemente um registro de ferramentas consultável pelo agente. Cada entrada deve conter pelo menos:

| Campo | Finalidade |
| --- | --- |
| `name` e `version` | Identificação estável da capacidade. |
| `description` e `tags` | Explicar quando a ferramenta é útil e permitir sua busca. |
| `input_schema` | Descrever os argumentos aceitos. |
| `output_schema` | Descrever o contrato do resultado, incluindo erros. |
| `enabled` | Controlar se a ferramenta pode ser descoberta e executada. |

Ofereça as capacidades de descoberta `search_tools(query)` e `get_tool_schema(name)`, ou equivalentes. A busca pode ser lexical: não é obrigatório usar embeddings. Ela deve retornar metadados relevantes e lidar com consultas sem resultado.

**No início de uma requisição, o agente deve receber somente as ferramentas de descoberta e, se necessário, um executor controlado. Os schemas completos das tools de domínio não podem estar todos no prompt ou na lista inicial de tools.**

O fluxo observável deve mostrar:

1. O agente identifica a necessidade e consulta o catálogo.
2. Obtém o contrato da ferramenta selecionada, pela busca ou por consulta específica.
3. A aplicação verifica disponibilidade e valida os argumentos.
4. A ferramenta é disponibilizada ou executada por um mecanismo controlado.
5. O agente usa o resultado para responder, incluindo as fontes.

São aceitas, por exemplo, atualização do conjunto de tools entre etapas do `Runner`, execução controlada de tools registradas por um dispatcher ou descoberta nativa compatível com o SDK/modelo escolhido. Em qualquer alternativa, use function tools do SDK no fluxo e mantenha a validação específica de cada ferramenta. Um dispatcher com argumentos JSON livres, sem validação pelo schema da tool selecionada, não atende.

Este requisito é comportamental: **não depende de um recurso específico de tool search da OpenAI ou de MCP**. Documente como o estado da descoberta é mantido durante a requisição e como os limites são aplicados ao fluxo completo. Apenas listar tools em um endpoint, mantendo todas pré-carregadas no agente, não comprova descoberta.

Uma tool inexistente, desabilitada ou ainda não disponibilizada para a requisição não pode ser executada por uma tentativa direta de chamada. A restrição deve existir em código, além das instruções do agente.

Para este desafio, uma tool habilitada fica disponibilizada quando seu contrato é obtido pelo mecanismo de descoberta durante a requisição atual. A busca pode fornecer esse contrato diretamente; um resultado contendo apenas nome e descrição exige a obtenção posterior do schema antes da execução.

## 4. Prova de extensão: uma nova tool

Entregue `get_evolution_chain` como uma extensão local confiável, acompanhada de seu contrato e testes. Disponibilize duas configurações reproduzíveis:

- **Configuração A:** somente as três primeiras tools de domínio estão registradas ou habilitadas.
- **Configuração B:** a tool de evolução foi adicionada ou habilitada no catálogo.

Na configuração A, as demais tools não devem retornar cadeias evolutivas nem permitir consultas genéricas que substituam a capacidade desabilitada.

Nas duas configurações, faça a mesma pergunta: **“Quais são as evoluções de Eevee e quais condições estão disponíveis para cada caminho?”**

Na configuração A, o assistente deve informar que não tem a capacidade necessária e não inventar a resposta. Na configuração B, deve descobrir a tool e responder usando os dados coletados.

Para passar, a extensão deve exigir apenas seu módulo/manifesto, seus testes e eventual registro declarativo. **Não altere o prompt principal, o endpoint de chat, o loop de orquestração ou um roteador com condições específicas para “evolução”.** Um registro explícito em arquivo de configuração é permitido; autodiscovery por import não é obrigatório.

O processo pode ser reiniciado entre A e B. Não é exigido hot reload. “Nova tool” significa código local escrito e registrado pela pessoa candidata, não código gerado, baixado ou executado autonomamente pelo modelo.

Inclua comandos para executar A e B e evidências dos dois resultados. Um commit separado, um diff ou um teste de extensão deve mostrar exatamente quais arquivos foram necessários para acrescentar a capacidade.

## 5. Contrato mínimo da API

| Método e rota | Contrato |
| --- | --- |
| `GET /health` | Informar se o processo está disponível; não deve consumir tokens nem consultar a PokéAPI. |
| `GET /tools?query=...` | Listar ou buscar as tools habilitadas e seus metadados. Pode ser determinístico, sem LLM. |
| `GET /tools/{name}` | Retornar o contrato de uma tool habilitada; responder com erro documentado para ausente/desabilitada. |
| `POST /chat` | Receber `{"message": "..."}` e executar o fluxo pelo Agents SDK. |

O endpoint de chat pode ser **stateless**; memória de conversação não é obrigatória. Aceite apenas as entradas necessárias ao escopo e limite o tamanho da mensagem.

A resposta de `/chat` deve ser JSON validado com, no mínimo:

| Campo | Conteúdo |
| --- | --- |
| `request_id` | Identificador para correlacionar resposta, logs e execução. |
| `status` | `ok`, `partial`, `not_found`, `unsupported` ou `error`, com semântica documentada. |
| `answer` | Resposta legível em português, distinguindo fatos, cálculos e limitações. |
| `sources` | Lista das URLs efetivamente utilizadas, timestamp da coleta e indicação de cache. Vazia quando não houver dados. |
| `tools_used` | Nomes das tools de descoberta e domínio realmente executadas. |
| `warnings` | Limitações, dados ausentes, falhas parciais ou uso de cache expirado, quando aplicável. |

Gere metadados de execução e proveniência a partir do runtime; não confie ao modelo a invenção de URLs, timestamps ou tools executadas. Documente o mapeamento entre status de negócio e códigos HTTP. Falhas de validação e indisponibilidade não devem virar um HTTP 200 com aparência de sucesso integral.

Exemplo de chamada, **a ser suportado pela implementação entregue**:

```bash
curl -sS http://localhost:8000/chat \
  -H 'Content-Type: application/json' \
  -d '{"message":"Compare Charizard e Blastoise pelos atributos base e explique as limitações da comparação."}'
```

## 6. Confiabilidade, limites e segurança

Implemente controles proporcionais a uma aplicação local:

- Timeout por chamada HTTP, prazo total por requisição e limite de tentativas com backoff para falhas transitórias. Não repita indefinidamente um 404 ou argumento inválido. Teste 429 com `Retry-After` por simulação; isso não pressupõe que a PokéAPI atualmente imponha esse status.
- Limites configuráveis para turnos/chamadas de modelo, execuções de tools e requisições HTTP, incluindo retries. Se houver mais de um `Runner.run`, o orçamento deve valer para a requisição completa. Documente os valores padrão e o comportamento quando acabarem.
- Limites de entrada, tamanho de retorno das tools e concorrência. Consultas paralelas não podem compartilhar indevidamente tools habilitadas durante a execução ou estado conversacional.
- Acesso da camada de coleta restrito à PokéAPI. Não aceite URLs arbitrárias do usuário ou do modelo. Valide URLs vindas dos próprios recursos e eventuais redirecionamentos antes de acessá-los.
- Validação de nome, ID e argumentos antes de executar a tool. Não exponha `eval`, shell, import arbitrário, leitura de arquivos ou acesso a variáveis de ambiente como ferramentas do agente.
- Conteúdo externo e mensagens do usuário não podem redefinir permissões ou instruções da aplicação. Demonstre uma tentativa de prompt injection em texto simulado de uma resposta da API e outra na mensagem do usuário.
- Segredos somente em variáveis de ambiente; `.env` ignorado pelo Git e `.env.example` sem valores reais. Não inclua dados, endpoints privados ou credenciais do Garden/TOTVS.

Cache expirado pode ser usado como fallback se a política estiver documentada e a resposta indicar a idade do dado e a falha de atualização. Também é aceitável recusar esse fallback e retornar indisponibilidade com clareza.

Não é necessário implementar login ou infraestrutura de produção. A execução local pode ficar restrita a loopback; publique a aplicação somente se também definir os controles adequados para isso.

## 7. Observabilidade e custo

Registre eventos estruturados com `request_id`, ferramenta/versão, duração, resultado ou código de erro, cache hit/miss e contagem de chamadas externas. Registre modelo e consumo de tokens quando fornecidos pelo SDK; valores indisponíveis devem aparecer como indisponíveis, nunca como zero medido.

Entregue evidências de pelo menos um fluxo completo de descoberta, um erro tratado e a comparação A/B da extensão. Logs JSON locais são suficientes. O [tracing do Agents SDK](https://developers.openai.com/api/docs/guides/agents/integrations-observability#tracing) também pode ser utilizado; documente sua configuração e evite registrar segredos ou exportar dados desnecessários. Os testes offline devem desativar exportações externas.

Para uma pequena execução ao vivo, registre quantidade de casos, modelo, versão do SDK, tokens e latência observados. Se estimar custo financeiro, identifique-o como estimativa, com data e referência de preço. Não é necessário fazer benchmark de carga ou usar um modelo caro.

## 8. Testes e avaliação do agente

Separe dois tipos de verificação:

1. **Testes determinísticos offline:** cliente HTTP, normalização, cache, schemas, cálculos, catálogo, limites, falhas e montagem da resposta. Devem rodar sem chave OpenAI e sem rede, usando fixtures/mocks identificados. É permitido simular o modelo para testar a orquestração.
2. **Avaliação do comportamento do agente:** casos em linguagem natural com critérios verificáveis, ferramentas esperadas ou permitidas e invariantes da resposta. Um replay simulado testa integração, mas não comprova a qualidade da decisão de um modelo real; identifique o modo de execução.

Crie um conjunto versionado em `evals/` que cubra os cenários abaixo. O formato é livre (JSON, YAML ou código), desde que haja um comando para executá-lo e produzir um relatório por caso. Não compare apenas a igualdade literal de textos gerados.

| ID | Cenário | Evidência de aceitação |
| --- | --- | --- |
| E01 | “Consulte Pikachu” e uma consulta equivalente por ID. | Mesma entidade, campos normalizados e fontes rastreáveis. |
| E02 | Repetir a consulta, inclusive após reiniciar o processo. | Cache persistente reutilizado dentro do TTL e sem novo GET para o recurso já armazenado. |
| E03 | “Compare Charizard e Blastoise pelos atributos base.” | Dados coletados, cálculos corretos, critério explícito e nenhuma promessa de vitória. |
| E04 | Consultar matchup de um Pokémon com dois tipos. | Efeitos combinados corretamente, incluindo imunidade quando presente no fixture; premissas declaradas. |
| E05 | Perguntar sobre evolução de Eevee nas configurações A e B. | Em A, capacidade indisponível; em B, descoberta antes da execução, árvore e condições baseadas nos dados. |
| E06 | Consultar um nome inexistente e uma lista com item inválido. | Sem alucinação; política de falha total ou parcial explícita e consistente. |
| E07 | Buscar uma capacidade ausente e tentar executar uma tool desabilitada ou não descoberta. | Busca sem resultado e bloqueio no runtime, sem executar o handler proibido. |
| E08 | Simular timeout, 429, 5xx e JSON inválido. | Retentativas limitadas, timeout respeitado e erro observável. |
| E09 | Solicitar acesso a URL arbitrária ou enviar argumentos inválidos. | Rejeição antes de acesso externo; inclua um redirecionamento fora da origem em teste. |
| E10 | Inserir instruções maliciosas no pedido e em texto externo simulado. | Sem execução fora do catálogo, mudança de permissões ou exposição de configuração. |
| E11 | “Quem ganha?” sem informar Pokémon ou critério. | Solicitação de esclarecimento ou limitação explícita, sem inventar premissas. |
| E12 | Esgotar o orçamento e executar duas requisições independentes em paralelo. | Encerramento controlado e limites preservados por toda a execução de cada requisição, sem compartilhar orçamento ou estado de descoberta entre elas. |

Um cenário pode conter vários testes, e o mesmo fixture pode servir a mais de um caso. E02, E07–E10 e E12 podem ser comprovados pelos próprios testes determinísticos, referenciados no relatório; não é necessário duplicá-los em outro framework. Inclua pelo menos um fluxo offline completo que passe pelo SDK com modelo simulado e comprove descoberta, validação e execução. Para prompt injection, explique o que o teste comprova e suas limitações; nenhum conjunto pequeno de exemplos comprova imunidade universal.

Registre um relatório com resultado por caso (`pass`, `fail` ou `not_run`), modo offline/live, motivo, duração e referência às evidências. Não esconda falhas escolhendo apenas execuções bem-sucedidas.

**Demonstração ao vivo:** quando houver credencial disponível, execute ao menos três pedidos com o modelo real: consulta simples, comparação e descoberta da extensão de evolução. Registre separadamente a execução da configuração A. Os demais cenários podem ser cobertos offline para evitar custo e tráfego desnecessários.

Se não houver crédito, acesso ao modelo ou disponibilidade externa, entregue os testes offline, o roteiro live executável e marque os casos live como `not_run`, explicando o motivo. Não é necessário contratar plano ou gastar para compensar indisponibilidade; o comportamento live poderá ser validado na entrevista com acesso combinado previamente. A limitação de acesso, por si só, não reduz a avaliação técnica.

## 9. Entregáveis

Entregue um repositório que permita a outra pessoa instalar, executar e avaliar a solução:

- Código-fonte e dependências fixadas, com versão do Python e do Agents SDK identificadas.
- `.env.example` com as configurações necessárias, sem segredos, e `.gitignore` apropriado.
- Instruções de instalação e comandos para subir a API, rodar testes offline, executar evals e demonstrar as configurações A/B.
- Exemplos reproduzíveis de chamadas HTTP e respostas reais ou fixtures claramente identificadas.
- Catálogo, schemas e mecanismo de registro da nova tool documentados.
- Testes, fixtures mínimas, casos de avaliação e relatório com resultados e limitações.
- Evidências sanitizadas de execução, sem depender de acesso à sua conta de observabilidade.
- Uma nota de arquitetura de **até duas páginas**, com desenho simples, decisões, alternativas descartadas, riscos e próximos passos.
- Registro do tempo gasto, pendências e uso de assistentes de IA, se houver.

Organização sugerida, adaptável à sua solução:

```text
README.md                  # Enunciado preservado e link para a documentação da solução
SOLUCAO.md                 # Instalação, execução, exemplos e limitações
.env.example
.gitignore
pyproject.toml             # Ou manifesto equivalente, com versões reproduzíveis
src/                       # API, agente, cliente, catálogo e tools
tests/                     # Testes offline e fixtures
evals/                     # Casos e comando de avaliação
docs/arquitetura.md         # Decisões e evolução para o Garden
evidence/                  # Relatórios e exemplos sanitizados
```

Os diretórios acima são uma sugestão para a entrega, não arquivos já fornecidos neste repositório. Não envie ambiente virtual, dependências vendorizadas, dumps completos da PokéAPI ou caches locais volumosos.

## 10. Regras de participação e entrega

1. **Trabalho individual:** você deve compreender e conseguir explicar todo o código entregue.
2. **Uso de IA permitido:** ChatGPT, Codex, Copilot e outras ferramentas podem ser usados. Informe quais utilizou, em quais partes e como verificou os resultados. Não é exigido compartilhar conversas privadas ou raciocínio interno.
3. **Referências permitidas:** bibliotecas, exemplos públicos e trechos de terceiros são permitidos com atribuição e respeito às licenças. Não apresente uma solução pronta de outra pessoa como autoria sua.
4. **Proteja informações:** use somente dados públicos ou sintéticos. Revise commits, logs, fixtures e capturas para evitar segredos. Não reutilize credenciais ou código privado de empregadores.
5. **Preserve o enunciado:** mantenha este README e documente a implementação em `SOLUCAO.md`, adicionando um link aqui. Não reduza requisitos para fazer a entrega parecer completa.
6. **Envie sua solução em repositório próprio:** use fork se permitido pela visibilidade deste repositório, ou uma cópia autorizada. Por padrão, mantenha a solução privada e conceda acesso apenas às pessoas avaliadoras indicadas pelo recrutamento. Não torne conteúdo privado público.
7. **Não faça push na `main` deste repositório nem abra PR com sua solução aqui**, salvo orientação expressa do time. As regras valem para a pessoa candidata; este é o repositório do enunciado.
8. **Submissão:** envie pelo canal combinado o link do repositório, branch, SHA do commit de entrega, instruções, tempo gasto e limitações conhecidas. A avaliação usa esse commit; sinalize eventuais correções posteriores.
9. **Sem dependência de infraestrutura privada:** a avaliação deve conseguir rodar localmente. Chaves são fornecidas por ambiente; nunca as envie junto da entrega. Uma execução live pode depender de acesso externo explicitamente documentado.
10. **Dúvidas e adaptações:** registre suposições em `SOLUCAO.md`. Prazo, acessibilidade e indisponibilidade de serviços devem ser alinhados com o recrutamento, sem necessidade de expor dados pessoais no repositório.

## 11. Apresentação na entrevista técnica

Prepare o ambiente para executar a solução e compartilhar o código durante a entrevista. A implementação será a base de uma conversa técnica sobre como você resolveu o problema e quais decisões tomou.

Organize a demonstração inicial para mostrar:

1. **Um fluxo completo:** uma pergunta em linguagem natural, a descoberta da tool, a coleta dos dados e a resposta com fontes.
2. **A extensão A/B:** como a capacidade de consultar evoluções é adicionada sem alterar o núcleo do agente.
3. **Um erro tratado:** uma entrada inválida, falha externa ou limite atingido, com o comportamento da aplicação e os registros correspondentes.
4. **Os testes e as evidências:** como executar as verificações, quais resultados foram obtidos e o que ainda precisa ser validado.

As pendências devem estar documentadas e podem ser discutidas durante a dinâmica. Se uma dependência externa estiver indisponível, use as evidências e os testes offline, identificando claramente o que foi simulado.

## 12. Referências

- [OpenAI — definição de agentes e execução com Agent/Runner](https://developers.openai.com/api/docs/guides/agents/define-agents).
- [OpenAI — tools no Agents SDK](https://developers.openai.com/api/docs/guides/tools#usage-in-the-agents-sdk).
- [OpenAI — integrações e observabilidade](https://developers.openai.com/api/docs/guides/agents/integrations-observability).
- [PokéAPI — documentação REST v2](https://pokeapi.co/docs/v2).
- [PokéAPI — política de uso](https://pokeapi.co/docs/v2#fairuse).

Consulte a documentação correspondente às versões utilizadas e registre essas versões na entrega. Este desafio não depende de funcionalidades experimentais específicas.
