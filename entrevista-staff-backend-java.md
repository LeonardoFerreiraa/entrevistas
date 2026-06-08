# Entrevista — Staff Backend Java

**Data:** 2026-06-04
**Vaga:** Staff Backend Engineer — Java
**Temas:** Java core, Spring, Redis, PostgreSQL, Kafka, RabbitMQ, microsserviços, design patterns (código + arquitetura), resiliência (rate limit, retry, circuit breaker) _(adicionado a pedido do candidato)_

**Modelo:** perguntas por tema, do básico ao teto do candidato. Tema travado → máx +2 perguntas → próximo tema. Observações do entrevistador registradas para estudo posterior.

---

## Tema 1 — Java Core

### P1.1 — Heap vs stack; onde fica objeto criado com `new`
**Resposta:** Stack = memória estática (constantes, declarações de métodos). Heap = memória "alocável", variáveis criadas com `new`.

**Observação do entrevistador:** Parcial. Acertou heap (objetos via `new`). Stack impreciso: stack guarda *frames de chamada de método*, variáveis locais e referências — não constantes nem "declarações de método". Constantes/metadados de classe ficam no *Method Area / Metaspace*. Não separou claramente: o objeto vai pra heap, a *referência* (variável local) vive na stack. Confundiu stack com metaspace.

---

### P1.2 — Como o GC decide coletar; gerações young/old
**Resposta:** GC coleta objetos órfãos (sem referência). Ex: variável local fica elegível ao fim do método. Não conhece young/old; chutou pelo nome — old = mais antigos, young = mais novos.

**Observação do entrevistador:** Conceito central correto (reachability / objeto sem referência alcançável). Chute de young/old certo no nome, mas sem saber o *porquê*: a divisão existe pela *weak generational hypothesis* (maioria dos objetos morre jovem). Young (Eden + survivor) é coletado rápido e frequente (minor GC); objetos que sobrevivem são promovidos pra old (major/full GC, mais caro). Não conhece o mecanismo. **Tópico parcialmente travado** — conceito ok, internals não.

---

### P1.3 — Memory leak mesmo com GC
**Resposta:** Acontece quando a referência ao objeto não morre. Ex: lista onde adiciono itens e nunca limpo.

**Observação do entrevistador:** Correto. Entendeu o ponto: leak em Java = referência viva indevidamente (objeto alcançável mas inútil), não falha do GC. Exemplo da coleção que só cresce é clássico e válido. Outros casos staff-level (não citados, ok): listeners não removidos, `ThreadLocal` em pool de threads, chaves em `HashMap` sem `equals/hashCode`, cache sem eviction.

**Teto Java Core:** conceitos corretos; falta profundidade de internals da JVM (gerações, metaspace, stack frames).

---

## Tema 2 — Design Patterns (código)

### P2.1 — Cite 3 design patterns usados na prática + problema que resolvem
**Resposta:**
- **Singleton:** beans do Spring são singleton por padrão; padrão mais usado em Java/Spring.
- **Chain of Responsibility:** encadear coisas (ex: validações), reaproveitar e rearranjar nós.
- **Decorator:** bean é encapsulado por classe filha (herança) que permite AOP; também usaria pra adicionar comportamento extra (ex: `StringWrapper` com funções que `String` não tem).

**Observação do entrevistador:** Boa resposta, 3 patterns reais com contexto. Singleton e Chain corretos. Nuance no Decorator: o proxy do Spring AOP é mais precisamente o **Proxy pattern** (CGLIB = subclasse; JDK dynamic proxy = via interface), parente próximo do Decorator mas com intenção diferente (controlar acesso vs. agregar comportamento). E Decorator "clássico" exige *mesma interface* do objeto decorado, envolvendo (composição), agregando comportamento de forma transparente — o exemplo `StringWrapper` que adiciona métodos novos é mais "wrapper/utility" do que Decorator estrito. Demonstrou bom faro de aplicação prática.

---

### P2.2 — Estado em bean singleton sob concorrência; como resolver
**Resposta:** Problema = instância compartilhada por todas as threads; race condition (uma escreve, outra lê valor inesperado; pior caso escrita sobreposta). Soluções: (1) passar o valor como parâmetro em todos os métodos; (2) `ThreadLocal` — cada thread guarda suas próprias "globais" (usado no Spring Security); (3) store externo (ex: Redis), fora do escopo.

**Observação do entrevistador:** Resposta forte, nível staff. Identificou race condition com precisão e deu leque de soluções do simples ao robusto. Citou corretamente `ThreadLocal` + Spring Security (`SecurityContextHolder`). Não citou a opção *Spring-nativa*: `@Scope("request")` / `@RequestScope` (bean por request) — seria a resposta idiomática. Também não mencionou o risco do `ThreadLocal` em *thread pools* (se não limpar, vaza estado pra próxima request e causa leak — conecta com P1.3).

---

### P2.3 — Risco do `ThreadLocal` em pool de threads
**Resposta:** Próxima request recebe "dirty context" (valor da request anterior). Cuidado: limpar o `ThreadLocal` ao fim de toda request. Prefere limpar no fim (cada um limpa a própria sujeira) a limpar no começo.

**Observação do entrevistador:** Excelente. Identificou o "dirty context" e a regra correta (limpar ao fim). Raciocínio sobre responsabilidade é maduro. Faltou só explicitar o *como* garantir: bloco `try/finally` (limpar no `finally` pra resistir a exceção) e usar `remove()` em vez de `set(null)` — `remove()` evita também o leak do entry no mapa. No Spring isso costuma ficar num `Filter`/`Interceptor`.

**Teto Design Patterns (código):** alto. Aplica patterns com consciência de trade-offs e concorrência. Lacuna pequena em terminologia GoF estrita (Decorator vs Proxy).

---

## Tema 3 — Patterns de Arquitetura

### P3.1 — Como organiza camadas/pacotes de um microsserviço; estilo nomeado
**Resposta:** Depende. Preferência: **layered** (controller > service > repository | client). Hexagonal/Clean são ótimos pra projetos multi-domínio (ex: monolito); em microsserviço, cada app já deveria ter um único domínio. Se o microsserviço tiver domínio grande, divide por *casos de uso*: em vez de `OrderService` gigante, ter `CreateOrderUseCase` (SRP). Gera use cases pequenos, mas melhor que services enormes difíceis de manter.

**Observação do entrevistador:** Resposta opinativa e bem fundamentada, com trade-offs. SRP via use case é maduro e prático. **Ponto a desafiar:** a tese "microsserviço = 1 domínio, logo layered basta" subvende o hexagonal. O valor central do hexagonal/ports & adapters *não* é multi-domínio — é **isolar a lógica de domínio de frameworks/IO** (DB, broker, HTTP) para testabilidade e pra trocar adapters sem tocar no core. Em layered puro, a regra de negócio tende a "vazar" pra dentro do framework. Vale ver no P3.2.

---

### P3.2 — Problema do service depender direto do repository; o que hexagonal evita
**Resposta:** Não chama isso de problema. O argumento pró-hexagonal (trocar JPA→JDBC, ou trocar de banco) cobra preço alto (DTOs, mappers/transformers de transporte) por uma migração rara que na maioria nunca acontece. No layered dele, com papéis claros (controller/schedulers/listeners = entradas; clients/repositories = fontes de dados; services = regra de negócio), a regra de negócio já fica num lugar que sofre pouco com mudança de infra.

**Observação do entrevistador:** Posição staff legítima e bem defendida — reconheceu o trade-off custo (boilerplate) vs. benefício (flexibilidade) e fez a escolha consciente. Pragmatismo correto: não adota hexagonal por dogma. **Porém** rebateu só o argumento *mais fraco* do hexagonal (troca de tecnologia/banco). O argumento mais forte, que ele não endereçou, é **testabilidade**: domínio dependendo de *portas* (interfaces) permite testar regra de negócio com fakes/stubs em memória, sem subir Postgres/Kafka/Testcontainers — testes rápidos e isolados. Em layered onde `service` importa o repositório JPA concreto, esse isolamento é mais difícil. Vale fechar com P3.3.

---

### P3.3 — Testabilidade como argumento pró-hexagonal; como testa hoje
**Resposta:** Não concorda que testabilidade seja o ponto forte. Em teste unitário, todas as dependências são mockadas (Mockito) — igual em layered ou hexagonal. Como em produção roda com todas as dependências reais, é importante ter testes de integração que integrem as camadas: subir contexto Spring completo, `RestTemplate`/requisições simuladas, Postgres em Testcontainers. Detalhe operacional: uma instância de Postgres pra suíte integrada, limpando a base ao fim de cada teste — ou cada teste preparado pra lidar com base suja.

**Observação do entrevistador:** Rebatida afiada e correta — desarmou meu argumento. Ponto certeiro: no Spring, o `@Service` já depende de uma **interface** Spring Data (`JpaRepository`), então a mockabilidade do unitário independe de hexagonal vs layered. Logo o ganho de testabilidade do hexagonal é marginal nesse stack. Ótima consciência da pirâmide de testes e de detalhes operacionais de teste de integração (isolamento de base, dado sujo). Mostrou senioridade real ao discordar com fundamento, não por teimosia.

**Teto Patterns de Arquitetura:** alto. Decisões arquiteturais conscientes, pragmáticas e defensáveis com trade-offs; não segue dogma. Forte.

---

## Tema 4 — Spring

### P4.1 — O que é IoC/DI; o que o Spring faz por você
**Resposta:** Spring gere o ciclo de vida dos beans. IoC = controle sobre as instâncias (criação e injeção) fica com o framework. Spring cria os beans na ordem certa do grafo de dependências (primeiro os que não dependem de ninguém, encadeando até criar tudo) e injeta os beans já criados do `ApplicationContext` em outros que precisam deles.

**Observação do entrevistador:** Mecânica correta (lifecycle, ordem topológica do grafo, injeção via container). **Crítica:** resposta ficou na mecânica do "como"; não articulou o **porquê** do DI — desacoplamento, programar contra interface, trocar implementação e testabilidade. Pra Staff, eu esperava o benefício de design, não só "o Spring instancia e entrega". Adequado, mas raso no valor arquitetural.

---

### P4.2 — Dependência cíclica A↔B
**Resposta:** Cenário de dependência cíclica. Spring dá erro quando a injeção é por construtor; com setter ou field injection ele instancia a classe e depois coloca a referência do outro objeto.

**Observação do entrevistador:** Correto e preciso. Construtor falha (`BeanCurrentlyInCreationException` — não dá pra construir A sem B pronto e vice-versa); setter/field resolve via *early reference* (Spring expõe a instância semi-criada antes de popular os campos). **Addendum:** desde Spring Boot 2.6 ciclos são proibidos por padrão (precisa `spring.main.allow-circular-references=true`), e ciclo costuma ser *code smell* — sinal pra extrair uma terceira colaboração. Bom domínio.

---

### P4.3 — `@Transactional` em self-invocation (`this.salvar()`)
**Resposta:** Gera muito bug em produção. Pro AOP funcionar a chamada precisa passar pelo proxy (classe gerada em runtime pelo CGLIB que `extends` o bean). Em OO Java, quando a "super class" chama um método declarado nela mesma, não passa pela subclasse.

**Observação do entrevistador:** Conclusão correta — `@Transactional` **não** aplica na self-invocation; a transação não abre. Mecanismo certo (proxy CGLIB intercepta só chamadas externas). **Crítica de precisão:** a moldura "superclasse chamando método nela mesma não passa pela subclasse" está *imprecisa*. O real: o proxy é um *objeto separado* que envolve o target; `this` dentro do bean aponta pro **target cru**, não pro proxy, então a chamada interna nunca cruza o proxy. Não é polimorfismo super/sub — é referência `this` ≠ referência proxy. Não citou o fix → cobrado no P4.4.

---

### P4.4 — Como fazer a transação funcionar mesmo na chamada interna
**Resposta:** Três opções: (1) `TransactionTemplate` injetado, gerindo a transação programaticamente (mas mistura infra com regra de negócio); (2) a classe injetar a si mesma (setter/field injection) e chamar `orderService.save` pra cruzar o proxy; (3) **preferida:** extrair o processamento em lote pra outro service (ex: `BatchOrderService`) que chama `orderService.save` — sem problema de proxy. Não gosta das duas primeiras.

**Observação do entrevistador:** Forte. Três soluções válidas e ranqueadas com critério (a extração pra outro bean é a mais idiomática e limpa — chamada entre beans distintos cruza o proxy naturalmente). **Crítica/lacuna staff:** ninguém tocou na *semântica transacional*, que é a pergunta de design por trás. `@Transactional` por item = uma transação por `salvar` (um pedido falha, os outros já commitados); `@Transactional` em volta do laço = tudo-ou-nada. O "fix certo" depende do requisito (continuar no erro? rollback total?), não só de "fazer a anotação disparar". Faltou explicitar essa decisão.

**Teto Spring:** alto. Domina proxy/AOP, transações, ciclo de vida, DI. Lacunas: articular o *valor de design* do DI e a *semântica* transacional além do mecanismo.

---

## Tema 5 — Mensageria (Kafka & RabbitMQ)

### P5.1 — Quando Kafka vs RabbitMQ; diferença fundamental de entrega/consumo
**Resposta:** Acha a diferença sutil; comum "forjar" mensageria no Kafka. Kafka = streaming, nº máx de consumidores depende do nº de instâncias; Rabbit = mensageria, "infinito teórico" de consumidores (limitado por canais/memória). Kafka permite reprocessamento (mensagem não é removida, só incrementa offset do consumer group); Rabbit remove a mensagem ao consumir. Fan-out: Rabbit precisa de exchange roteando pra múltiplas queues; Kafka já faz com offsets/consumer groups. Consumo Kafka mais eficiente em lote (zero-copy: dado vai do HD pra rede sem passar por memória da app). Erros: consumidor Kafka fica "preso" na mensagem e em algum momento é forçado a avançar offset mesmo sem processar (DLT existe, mas não gosta); Rabbit permite DLQ com reconsumo/retentativa isolada sem travar o processo.

**Observação do entrevistador:** Resposta densa e de bom nível — zero-copy, retention/offset, fan-out, head-of-line blocking. **Críticas de precisão (importam pra Staff):**
1. O teto de paralelismo de um consumer group no Kafka é o **nº de partições**, não de instâncias. Instâncias além do nº de partições ficam *ociosas*. Foi impreciso aqui.
2. **Faltou o critério mais fundamental de escolha: ordenação.** Kafka garante ordem *dentro de uma partição* (por chave); Rabbit com competing consumers tem ordenação fraca. Isso, e não retention, costuma ser o divisor de águas (ex: event sourcing, CDC).
3. Resumo de seleção que faltou explicitar: Kafka = log de eventos/replay/alto throughput/streaming; Rabbit = task queue/distribuição de trabalho/roteamento complexo/ack por mensagem.
Domínio real, mas com imprecisão em partições e lacuna em ordenação.

---

### P5.2 — Key nula em tópico particionado; garantir ordem dos eventos do pedido
**Resposta:** (Corrigiu P5.1: quis dizer partições, não instâncias.) Com chave nula as mensagens vão em round-robin entre as partições, então cada estado do pedido cai numa partição diferente e são consumidos em paralelo (cada partição tem 1 e só 1 consumidor por consumer group) → perde ordem. Correto: usar o id do pagamento como chave de roteamento, forçando os 3 estados na mesma partição e garantindo ordem de consumo.

**Observação do entrevistador:** Correto e completo. Causa (RR → espalha → paralelo → desordem), regra "1 consumidor por partição por grupo" e o fix (key = id) — tudo certo. (Nota: clients novos usam *sticky partitioner* em vez de RR puro com key nula, mas o efeito de espalhar mantém.) Bom domínio do particionamento.

---

### P5.3 — Como a ordem ainda pode quebrar mesmo com chave certa
**Resposta:** Imagina que pode acontecer no consumo em lote pelo consumidor. Do lado do produtor, talvez replicação entre nós ou a fila em memória que o Spring usa pra enviar ao Kafka.

**Observação do entrevistador:** Está circulando a área certa mas não aterrissou. **Crítica:**
- "Consumo em lote" só quebra ordem *se* o consumidor processar o lote em paralelo/multi-thread; consumir em lote e processar sequencialmente preserva ordem. Faltou essa condição.
- "Fila em memória do produtor" tá perto, mas o mecanismo exato não saiu. Tópico no teto — segue P5.4 (nudge).

---

### P5.4 — Reorder por retry com in-flight; config que controla
**Resposta:** Ficam em ordem M2, M1 (não M1, M2). Não sabe o nome exato, mas arrisca "max in flight" que faria o segundo esperar o primeiro ter sucesso. Acha que serializar seria problemático na maioria dos casos: atraso grande na entrega ou perda de mensagem se a instância morrer com registros em memória.

**Observação do entrevistador:** Recuperou bem no nudge. Reorder M2,M1 correto e aterrissou o conceito: `max.in.flight.requests.per.connection=1` serializa e garante ordem, ao custo de throughput/latência. Raciocínio de trade-off bom. **Lacuna (o fix moderno):** `enable.idempotence=true` — com produtor idempotente o broker numera/deduplica e você mantém ordem **mesmo com `max.in.flight` até 5**, sem sacrificar throughput. É a resposta staff (não precisa escolher entre ordem e vazão). A preocupação com perda se a app morre com buffer em memória é válida e conecta com `acks`/durabilidade.

**Teto Mensageria:** alto, mas com bordas. Domina modelo, particionamento, ordenação e head-of-line. Precisou de nudge nas armadilhas finas de produtor (in-flight/retry/idempotência). Imprecisões pontuais (instâncias vs partições, RR vs sticky).

---

## Tema 6 — Redis

### P6.1 — Usos do Redis além de cache
**Resposta:** (1) **Contador** — ex: saber quando todos os itens de um lote foram processados; chave com ID do lote + `INCR`, ao atingir o valor dispara a operação finalizadora. Funciona porque Redis roda single-thread → sem concorrência no acesso à mesma chave. (2) **Rate limit** — usa o controle de concorrência do Redis pra gerir os buckets. (Pediu pra adicionar tema futuro de resiliência: rate limit, retry, circuit breaker.)

**Observação do entrevistador:** Dois usos válidos com a razão certa (atomicidade do `INCR`). Precisão: o motivo exato é a **atomicidade** das operações, não só "single-thread" — pra sequências multi-passo (read-modify-write) precisa `MULTI/EXEC` ou script **Lua**, single-thread sozinho não garante. **Lacuna prática (staff):** o contador "todos processados" sofre com entrega *at-least-once* — worker que reprocessa incrementa duas vezes e estoura/erra a contagem; precisa de **idempotência** (ex: `SADD` do id do item num set e checar cardinalidade, em vez de `INCR` cego). Não citou outros clássicos: **lock distribuído** (próxima pergunta), pub/sub, sorted sets (ranking), session store.

---

### P6.2 — Lock distribuído no Redis (1 instância roda o job)
**Resposta:** Chave = nome do job, valor = ID da instância. Quem conseguir o set garante a execução. Não lembra a instrução, mas é "set if absent" (insere se não existe, senão não faz nada). Evita deadlock com **TTL** (instância morre → lock expira). Em cenários complexos, **heartbeat**: thread extra que a cada X tempo estende o TTL por Y, pra jobs sem fim previsível.

**Observação do entrevistador:** Resposta forte e idiomática. Comando = `SET key value NX PX <ms>` (NX = set if absent; PX = TTL atômico no mesmo comando — importante ser atômico, não `SETNX` + `EXPIRE` separados, que abre janela de deadlock). TTL e heartbeat corretos e bem pensados. **Lacuna staff (a parte traiçoeira que faltou): liberar o lock com segurança.** Se o TTL de A expira e B adquire o lock, A não pode dar `DEL` cego (deletaria o lock de B). Por isso valor = token único (ele usou instanceId, bom!) e o release tem que ser **check-and-delete atômico via Lua** (`GET`→compara→`DEL`). Citou o token mas não o release seguro. **Nível mais profundo (não citado):** Redis único é SPOF — em failover (master cai, réplica promovida sem o lock replicado) dois nós podem segurar o lock; daí Redlock e o debate sobre **fencing tokens** (token monotônico que o recurso protegido valida). Pra "só 1 job", lock Redis é ok pragmaticamente, mas correção forte exige fencing.

---

### P6.3 — (P6.3 original pulada: candidato leu a resposta no markdown) Cache stampede
**Resposta:** Não sabe o nome. Soluções: (1) TTL baseado na última leitura, não na escrita (com ressalva: chave muito quente nunca atualiza, só serve se houver invalidação/atualização via fila/streaming); (2) semáforo/mutex na busca do registro, impedindo o burst no momento da expiração; (3) backpressure tipo Hystrix, um funil que limita a concorrência do comando de busca.

**Observação do entrevistador:** Não sabe o nome (**cache stampede** / *thundering herd* / *dogpile*), mas as mitigações são boas e práticas. O **mutex por chave** (só um recalcula, os outros esperam ou servem stale) é a resposta canônica — acertou. Backpressure pra proteger o banco é válido. Ressalva certa sobre sliding TTL. **Faltaram (não citados):** *probabilistic early expiration* (XFetch — recalcula um pouco antes do TTL com probabilidade crescente) e servir *stale-while-revalidate* (devolve o valor velho e atualiza async). Bom distinguir dos primos: *cache penetration* (consulta a chave inexistente) e *cache avalanche* (muitas chaves expiram juntas → resolve com **jitter** no TTL).

**Teto Redis:** alto. Domina atomicidade, lock distribuído, contadores, rate limit, padrões de cache. Lacuna mais de *nomenclatura* do que de solução.

---

## Tema 7 — PostgreSQL

### P7.1 — Como índice B-tree acelera; caso em que é inútil/prejudicial
**Resposta:** Índice dá chave de acesso direta sem varrer comparando registros (como busca por ID, vai direto à posição). Inútil quando a query tem modificador na coluna (ex: `trunc`) → Postgres ignora o índice. Prejudicial quando escrita > leitura, ou tabela já tem muitos índices: a indexação acontece junto com o insert; se o serviço faz mais insert que select, onera; muitos índices = mais CPU recalculando índice, deixando o resto lento.

**Observação do entrevistador:** Boa resposta com casos certos. O ponto da **função na coluna** (`date_trunc`, `lower`, cast) matando o índice é staff-correto — exige índice de *expressão/funcional* pra resolver. Write amplification (índice mantido a cada insert/update) também certo. **Precisão:** B-tree não é acesso "direto à posição" como hash O(1); é árvore *ordenada*, travessia O(log n) — evita o seq scan, mas não é endereçamento direto. **Lacuna (o outro grande 'inútil'):** **baixa seletividade/cardinalidade** — índice em coluna booleana ou status com 2 valores; o planner ignora porque ler o índice + heap sai mais caro que seq scan. Seletividade é o critério central que não apareceu — base pra P7.2.

---

### P7.2 — Por que o planner ignora o índice e faz Seq Scan
**Resposta:** Disse "não sei", mas raciocinou: `status = 'ATIVO'` casa com muitos registros, e o Postgres entende que é melhor full scan do que olhar uma lista grande de registros pelo índice.

**Observação do entrevistador:** Acertou apesar de dizer que não sabia — é exatamente **seletividade**. **Camada staff a completar:** o planner é *cost-based*; estima quantas linhas o predicado retorna usando **estatísticas** (`pg_statistic`, atualizadas por `ANALYZE`/autovacuum). Quando a fração estimada é grande, o custo de N acessos *aleatórios* ao heap (random I/O, um por hit do índice) supera o de **um seq scan sequencial** — então escolhe seq scan. Corolário prático: **estatística desatualizada** (sem `ANALYZE`) também leva a plano ruim mesmo com boa seletividade. Bom faro; subvalorizou a própria resposta.

---

### P7.3 — Duas formas de impedir lost update (uma trava, outra detecta)
**Resposta:** Lock **pessimista** e **otimista**. Pessimista: `SELECT FOR UPDATE` — o registro é meu até o fim da transação. Otimista: campo `version`; no update, `WHERE version = {valor lido} SET version = version + 1` — quando ambas as threads atualizam, uma não satisfaz o `WHERE` do version.

**Observação do entrevistador:** Resposta perfeita e precisa. Mapeia direto pro JPA: `@Lock(PESSIMISTIC_WRITE)` (= `SELECT ... FOR UPDATE`) vs `@Version` (otimista). Mecânica do otimista exata (update afeta 0 linhas → conflito detectado). **Complemento de trade-off (não dito):** pessimista *bloqueia* — serializa, reduz throughput, risco de deadlock; bom pra **alta contenção**. Otimista é *não-bloqueante*, mas exige **lógica de retry** ao detectar o conflito; bom pra **baixa contenção**. Faltou só explicitar *quando escolher cada um* e o tratamento do conflito (retry/abort).

**Teto PostgreSQL:** alto. Índices, planner/seletividade, controle de concorrência sólidos. Pequenas imprecisões de internals (B-tree vs hash) e faltou nomear seletividade/estatísticas sem ajuda.

---

## Tema 8 — Microsserviços

### P8.1 — Consistência distribuída no checkout (3 serviços, 3 bancos)
**Resposta:** O mais comum é **Saga**. Opção 1: coordenador central; cada serviço expõe a operação e um jeito de fazer rollback (ruim porque se o coordenador morre perde-se o controle do estado → coordenador precisa ser muito resiliente). Opção 2: "saga orquestrado", perto de arquitetura orientada a eventos, com awareness do estado dos outros serviços.

**Observação do entrevistador:** Conceito central certo (Saga + compensação no lugar de rollback distribuído). **Crítica de terminologia (trocou os nomes):** coordenador central = **Orchestration**; o modelo event-driven sem coordenador, onde cada serviço reage a eventos = **Choreography** — ele chamou o event-driven de "orquestrado", invertido. Conceitos ok, rótulos trocados. **Lacuna staff importante:** não citou **transações compensatórias** pelo nome (refund, restock) nem — crítico — o problema do **dual-write**: como gravar no banco *e* publicar o evento atomicamente? Sem isso a saga fura. É a P8.2.

---

### P8.2 — Dual write (save no banco + publish no Kafka)
**Resposta:** Não é exatamente sobre saga; é **dual write**, acontece em qualquer situação com duas fontes de verdade. Resolve com **transactional outbox**: salvo o registro na tabela principal (ex: `payment`) e também numa tabela `outbox` (nome do tópico Kafka, o Avro a publicar etc.). Pra publicar, um processo separado via Scheduler ou infra de **CDC** que garante *at-least-once* na publicação.

**Observação do entrevistador:** Resposta staff completa e correta. Nomeou (**dual write**) e deu a solução canônica (**transactional outbox**). O ponto-chave que ele acertou: a linha da outbox é gravada **na mesma transação local** da escrita de negócio → atomicidade vem do único commit ACID; depois um *poller* (Scheduler) ou **CDC (ex: Debezium)** lê a outbox e publica. Mencionou Avro/tópico na tabela e CDC — ótimo. **Corolário a fechar:** como a entrega é *at-least-once*, o **consumidor precisa ser idempotente** (deduplicar por id do evento), senão processa duas vezes — fecha o ciclo com P8.3.

---

### P8.3 — Consumidor idempotente (evento chega 2x)
**Resposta:** Mesmo sem outbox, todo consumidor Kafka precisa ser **idempotente** (consumir 2x o mesmo evento → executar a operação uma vez só). Cada consumidor tem seu jeito; o básico é garantir que o registro está no estado esperado antes de avançar; se não estiver, avançar o offset sem quebrar o estado.

**Observação do entrevistador:** Termo certo (**idempotência**) e abordagem válida — *state-based* (checar o estado atual antes de aplicar; ex: `UPDATE ... WHERE status = 'PENDENTE'`). **Crítica/lacuna staff:** o "checar estado" só funciona pra operações *naturalmente idempotentes* ou condicionais. Pra um **delta não-idempotente** (ex: "baixar estoque em 1") o estado não revela se você já aplicou — dois eventos legítimos e um reprocessamento ficam indistinguíveis. A solução geral é a **tabela de dedup / inbox** (*processed events*): guardar o `eventId` com **unique constraint**, na **mesma transação** da escrita de negócio; se o id já existe, descarta. Citou o conceito, faltou esse mecanismo pra casos não-idempotentes.

**Teto Microsserviços:** alto. Domina saga, dual-write, outbox, CDC, idempotência — mecânica de sistemas distribuídos forte. Lapsos em *nomenclatura* (orchestration↔choreography invertidos) e no mecanismo de dedup pra operações não-idempotentes.

---

## Resumo final da entrevista

**Candidato:** mail@leonardoferreira.com.br
**Resultado geral:** Perfil **forte de Staff Backend Java**, com domínio prático e raciocínio de trade-offs maduro. Brilha em concorrência, mensageria e sistemas distribuídos. Pratica > teórico: às vezes acerta o conceito mas erra/desconhece o *nome* canônico.

### Teto por tema
| Tema | Teto | Comentário curto |
|---|---|---|
| Java Core | Médio-alto | Conceitos certos; fraco em internals da JVM (gerações, metaspace, stack frames). |
| Design Patterns (código) | Alto | Aplica com consciência de concorrência; lapso GoF (Decorator vs Proxy). |
| Patterns de Arquitetura | Alto | Decisões pragmáticas e defensáveis; não segue dogma. Desarmou argumento do entrevistador. |
| Spring | Alto | Domina proxy/AOP, transações, DI, ciclo de vida. Faltou *valor de design* do DI e semântica transacional. |
| Mensageria (Kafka/Rabbit) | Alto | Particionamento, ordenação, head-of-line. Precisou nudge em in-flight/idempotência de produtor. |
| Redis | Alto | Lock distribuído, atomicidade, padrões de cache. Lacuna de nomenclatura (cache stampede). |
| PostgreSQL | Alto | Índices, planner/seletividade, lock otimista/pessimista. Subvaloriza as próprias respostas. |
| Microsserviços | Alto | Saga, outbox, CDC, idempotência. Inverteu orchestration↔choreography. |

### Padrões observados
- **Força:** concorrência (race condition, ThreadLocal, lock otimista/pessimista, idempotência) e sistemas distribuídos (saga, outbox, dual-write, ordenação Kafka). Pensa em produção e falhas reais.
- **Maturidade:** discorda com fundamento, reconhece trade-offs, escolhe consciente — comportamento de Staff.
- **A melhorar:** (1) *nomenclatura canônica* — sabe a coisa, erra o nome (orchestration/choreography, cache stampede, seletividade); (2) *internals* da JVM e detalhes de profundidade quando o "como funciona por baixo" é cobrado; (3) tende a parar no mecanismo e não explicitar o *porquê de design* / a *semântica*.

### Estudar a seguir (gaps concretos)
- JVM internals: gerações da heap (young/old, Eden/survivor), metaspace, stack frames, tipos de GC.
- Spring: `@Scope("request")`, semântica de propagação de `@Transactional`.
- Kafka: `enable.idempotence`, `acks`, `max.in.flight`, sticky partitioner.
- Redis: nomes — cache stampede, XFetch, stale-while-revalidate, cache avalanche/jitter; release de lock via Lua + fencing tokens.
- Postgres: seletividade/estatísticas/`ANALYZE`, índices funcionais e parciais, custo random vs sequential I/O.
- Distribuídos: orchestration vs choreography (rótulos), transações compensatórias, inbox/dedup table pra idempotência de operações não-idempotentes.

---

## Temas para próximas sessões (reexecução da dinâmica)
Anotados a pedido do candidato; **não cobertos nesta sessão**.

- **Resiliência** — rate limit, retry, circuit breaker, bulkhead, timeout, fallback.
- **Tarefas agendadas** — scheduling distribuído, lock de job, idempotência de execução, jitter, cron vs fixed-delay.
