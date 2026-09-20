# Custom STL Containers — Vector + HashMap

> Especificação técnica detalhada. Duas fases independentes (Vector primeiro, HashMap depois), cada uma com API, decisões de design, casos de teste e critérios de conclusão.

![](./doc/image/9ikdn0pplhd25i7run21.png)

## Objectivo Geral

Perceber, por dentro, o que `std::vector` e `std::unordered_map` fazem de facto — estratégia de realloc, invalidação de iteradores, hashing, colisões — implementando versões próprias com a mesma interface essencial, e comparando-as com as da standard library em correcção e performance.

## Pré-requisitos

- C++ confortável com classes, templates básicos e ponteiros.
- Não é preciso já saber move semantics ou RAII em profundidade — são conceitos que este projecto ensina na prática.
- Compilador com suporte a C++17 ou C++20 (para `if constexpr`, structured bindings, etc. — úteis mas não obrigatórios).

## Ferramentas recomendadas

- **Google Test** ou **Catch2** para a suite de testes.
- **Google Benchmark** para os benchmarks de performance (dá resultados muito mais fiáveis do que medir `std::chrono` à mão).
- **AddressSanitizer** e **UndefinedBehaviorSanitizer** (`-fsanitize=address,undefined`) — liga-os desde o primeiro dia. Um container próprio é o sítio perfeito para introduzir memory bugs subtis, e estas ferramentas apanham a maioria antes de te custarem horas de debugging.
- **Valgrind** (opcional, mais lento que os sanitizers mas útil para leak detection detalhado).

---

# Parte 1 — Vector Próprio

## 1.1 API mínima a implementar

```cpp
template <typename T>
class Vector {
public:
    // Construtores / destrutor / atribuição
    Vector();
    explicit Vector(size_t count);
    Vector(std::initializer_list<T> init);
    Vector(const Vector& other);           // cópia
    Vector(Vector&& other) noexcept;       // move
    Vector& operator=(const Vector& other);
    Vector& operator=(Vector&& other) noexcept;
    ~Vector();

    // Acesso a elementos
    T& operator[](size_t index);
    const T& operator[](size_t index) const;
    T& at(size_t index);                   // com bounds checking, lança excepção
    T& front();
    T& back();

    // Capacidade
    size_t size() const noexcept;
    size_t capacity() const noexcept;
    bool empty() const noexcept;
    void reserve(size_t new_capacity);
    void shrink_to_fit();

    // Modificadores
    void push_back(const T& value);
    void push_back(T&& value);
    template <typename... Args>
    T& emplace_back(Args&&... args);
    void pop_back();
    void clear();
    void resize(size_t count);
    void insert(iterator pos, const T& value);
    void erase(iterator pos);

    // Iteradores
    iterator begin() noexcept;
    iterator end() noexcept;
    const_iterator begin() const noexcept;
    const_iterator end() const noexcept;
};
```

Não precisas de implementar tudo o que `std::vector` tem — esta é a API mínima que já força a lidar com todos os conceitos-chave. Deixa `insert`/`erase` genéricos (para qualquer posição) para o fim; são os mais delicados por causa de invalidação de iteradores.

## 1.2 Growth strategy — o coração do projecto

Quando `push_back` excede a capacidade actual, o vector tem de realocar. As decisões que tomas aqui são o que separa "um array dinâmico qualquer" de "perceber a estratégia amortizada":

- **Growth factor**: usa um factor de crescimento fixo (tipicamente **1.5x ou 2x**). Implementa ambos como opção de template ou constante configurável, e mede o impacto de cada um no benchmark final — é um dos resultados mais interessantes do projecto.
- **Amortised O(1)**: implementa e depois demonstra por análise (não só por benchmark) porquê `push_back` é O(1) amortizado apesar de ocasionalmente custar O(n). Escreve essa análise no teu README — é um raciocínio que aparece em entrevistas técnicas.
- **Realloc real vs. realloc + copy**: para tipos triviais (`int`, `float`, structs sem destrutor customizado), podes usar `realloc` de C. Para tipos não-triviais, tens de alocar nova memória, mover/copiar cada elemento, e destruir os antigos — não podes usar `realloc` porque isso não chama construtores/destrutores. Implementa a versão genérica (aloca + move + destrói) primeiro; otimizar para tipos triviais é um extra opcional.

## 1.3 Move semantics — o que tens de garantir

Este é o conceito mais importante do projecto Vector. Regras a cumprir:

1. O construtor de move e o `operator=` de move devem ser **`noexcept`** — isto é o que permite ao `std::vector` real (e ao teu) usar `move` em vez de `copy` durante um realloc. Verifica isto explicitamente: se o teu `push_back` estiver a copiar em vez de mover elementos durante um realloc, o benchmark contra `std::vector` vai mostrar isso claramente (muito mais lento com tipos "caros" de copiar, como `std::string`).
2. Depois de um move, o objecto de origem deve ficar num **estado válido mas não especificado** — tipicamente, tamanho 0 e ponteiro `nullptr`. Nunca deixes o destrutor de um objecto "movido de" tentar libertar memória que já não é dele.
3. Usa `std::move_if_noexcept` internamente no realloc: se o construtor de move de `T` não for `noexcept`, o vector deve preferir copiar em vez de mover (para manter a garantia forte de excepção — se copiar falhar a meio, o vector original continua intacto).

**Teste específico a escrever**: cria um tipo de teste `TrackedType` que incrementa contadores globais em cada construção, cópia, move e destruição. Usa-o para verificar, com asserts, que um `push_back` que causa realloc de N elementos resulta em exactamente N moves (não N copies) quando o tipo tem move constructor `noexcept`.

## 1.4 Iteradores válidos

- Implementa `iterator` como um wrapper simples à volta de `T*` (não precisas de um iterador "fancy" com todas as categorias — um **random access iterator** básico chega).
- Suporta: `operator++`, `operator--`, `operator+`, `operator-`, `operator*`, `operator->`, `operator==`, `operator!=`, `operator<` (para permitir usar `std::sort` e outros algoritmos da STL directamente sobre o teu vector — isto é um bom teste de integração).
- **Regras de invalidação a implementar e testar explicitamente**:
  - Qualquer `push_back` que cause realloc invalida **todos** os iteradores.
  - `push_back` sem realloc invalida apenas o `end()` (o novo elemento é criado depois dele).
  - `erase` invalida o iterador apagado e todos os que vêm depois dele (porque tudo desloca uma posição).
  - `insert` invalida o iterador de inserção e todos os que vêm depois.

**Teste específico**: com sanitizers ligados, escreve um teste que guarda um iterador, causa deliberadamente um realloc, e tenta desreferenciar o iterador antigo — deve dar erro do AddressSanitizer (use-after-free), confirmando que o teu vector realmente liberta a memória antiga em vez de ficar com um leak "acidentalmente seguro".

## 1.5 Casos de teste específicos (Vector)

| # | Cenário | O que verificar |
|---|---------|------------------|
| 1 | `push_back` de 0 até 100.000 elementos | `size()` correcto em cada passo; nenhum crash com sanitizers |
| 2 | Construir com tipo não-trivial (`std::string`) | Nenhum leak, nenhum double-free |
| 3 | Copiar um vector com 1000 elementos | Vector original intacto após a cópia; deep copy confirmado (mudar um não afecta o outro) |
| 4 | Mover um vector com 1000 elementos | Custo O(1) (verifica que não houve N cópias — usa `TrackedType`); origem fica vazia e segura de destruir |
| 5 | `reserve` seguido de N `push_back` (N ≤ capacidade reservada) | Zero reallocs (conta quantas vezes o construtor de move é chamado — deve ser zero) |
| 6 | `insert` a meio de um vector com elementos não-triviais | Ordem final correcta; todos os elementos deslocados correctamente movidos, não copiados |
| 7 | `erase` a meio, depois `push_back` | `size()`/`capacity()` coerentes; sem leak do elemento apagado |
| 8 | Vector de Vectors (`Vector<Vector<int>>`) | Nested moves funcionam correctamente — bom teste de recursão de move semantics |
| 9 | Excepção no construtor de cópia de `T` a meio de um realloc | Vector original permanece intacto (strong exception guarantee) — teste avançado, opcional mas valioso |
| 10 | `std::sort(vec.begin(), vec.end())` sobre o teu vector | Confirma que os teus iteradores cumprem os requisitos mínimos de random access iterator |

## 1.6 Milestones (Vector)

- [ ] Construtor por omissão, destrutor, `push_back` sem move semantics (só cópia) — versão "burra" mas funcional
- [ ] Growth strategy com realloc genérico (aloca + move/copy + destrói)
- [ ] Move constructor e move assignment `noexcept`
- [ ] Iteradores básicos (`begin`/`end`, `operator++`, `operator*`)
- [ ] `insert`/`erase` com invalidação correcta
- [ ] Suite de testes completa (tabela acima) a passar com sanitizers activos
- [ ] Benchmark contra `std::vector`

---

# Parte 2 — HashMap com Robin Hood Hashing

## 2.1 Porque Robin Hood Hashing (e não chaining)

`std::unordered_map` da maioria das implementações usa **separate chaining** (cada bucket é uma lista ligada). Isso é simples mas mau para cache — cada acesso a um elemento numa lista ligada é potencialmente um cache miss.

**Robin Hood hashing** usa **open addressing** (todos os elementos vivem directamente no array, sem listas ligadas), o que significa melhor localidade de cache. A ideia central: quando insere um elemento e encontra um slot ocupado, compara a "distância à posição ideal" (o quão longe cada elemento está do seu bucket calculado pelo hash) de ambos — o que estiver **mais longe de casa** fica no slot, e o outro continua a procura ("rouba-se aos ricos para dar aos pobres", daí o nome). Isto mantém a variância de distâncias baixa, o que é o que torna o Robin Hood hashing rápido na prática.

## 2.2 API mínima a implementar

```cpp
template <typename K, typename V, typename Hash = std::hash<K>>
class HashMap {
public:
    HashMap();
    explicit HashMap(size_t initial_capacity);

    // Modificadores
    std::pair<iterator, bool> insert(const K& key, const V& value);
    bool erase(const K& key);
    void clear();

    // Acesso
    V& operator[](const K& key);           // insere se não existir
    iterator find(const K& key);
    bool contains(const K& key) const;

    // Capacidade
    size_t size() const noexcept;
    bool empty() const noexcept;
    float load_factor() const noexcept;
    void reserve(size_t count);

    // Iteradores
    iterator begin();
    iterator end();
};
```

## 2.3 Estrutura interna

Cada slot do array interno deve guardar, além de `K` e `V`:

- Um campo **`probe_distance`** (também chamado PSL — Probe Sequence Length): quantos slots de distância este elemento está da sua posição "ideal" (`hash(key) % capacity`).
- Um marcador de **slot vazio** (pode ser um valor sentinela em `probe_distance`, ex: `-1`, ou um `bool` separado — evita usar `std::optional<K>` por simplicidade e overhead).

```cpp
struct Slot {
    K key;
    V value;
    int32_t probe_distance = -1;  // -1 = slot vazio
};
```

## 2.4 Algoritmo de inserção (passo a passo)

1. Calcula `index = hash(key) % capacity`.
2. `distance = 0`.
3. Enquanto o slot em `index` não estiver vazio:
   - Se a key já existir nesse slot, actualiza o valor e termina.
   - Se `distance > slot[index].probe_distance` (o elemento que estás a inserir está **mais longe de casa** do que o que já lá está): troca o par (key, value, distance) que estás a inserir pelo que está no slot — o que estava lá continua a "andar à procura" de um novo lugar com os valores que acabaste de tirar.
   - Avança: `index = (index + 1) % capacity`, `distance += 1`.
4. Quando encontrares um slot vazio, coloca lá o elemento actual (seja o original ou o que foi "deslocado" durante as trocas).
5. Se `load_factor()` ultrapassar o limiar (tipicamente **0.9**, mais alto que o típico de chaining porque Robin Hood tolera load factors mais altos), faz **resize** (ver 2.6) antes ou depois da inserção.

## 2.5 Algoritmo de remoção — backward shift deletion

**Não** marques o slot como "tombstone" (comum noutras implementações open addressing) — isso degrada performance ao longo do tempo. Em Robin Hood hashing, a técnica correcta é **backward shift deletion**:

1. Encontra o slot da key a remover.
2. Marca-o como vazio.
3. Olha para o próximo slot: se estiver ocupado **e** `probe_distance > 0`, desloca-o uma posição para trás (para o slot que acabaste de esvaziar) e decrementa o seu `probe_distance` em 1.
4. Repete o passo 3 avançando, até encontrares um slot vazio ou um elemento com `probe_distance == 0` (já está na posição ideal, não pode ser deslocado para trás).

Isto mantém a invariante de Robin Hood (distâncias sempre correctas) sem nunca precisar de tombstones.

## 2.6 Resize

- Quando o load factor ultrapassa o limiar, cria um novo array (tipicamente **2x** a capacidade actual).
- Reinsere **todos** os elementos existentes no novo array, do zero — os hashes módulo a nova capacidade vão dar índices diferentes, por isso não há forma de "copiar" directamente.
- Isto é O(n), mas amortizado O(1) por inserção pela mesma razão do Vector.

## 2.7 Função de hash

- Usa `std::hash<K>` por omissão (como no template acima), mas implementa também a tua própria função de hash para pelo menos um tipo (ex: `std::string`) — experimenta **FNV-1a** ou **MurmurHash3**, e compara a distribuição de colisões contra `std::hash` no teu benchmark.
- Cuidado com um erro comum: se a tua função de módulo for `hash % capacity` e a capacidade for sempre potência de 2, considera usar `hash & (capacity - 1)` (mais rápido, mas exige capacidades sempre potência de 2 — decide isso conscientemente, não por acaso).

## 2.8 Casos de teste específicos (HashMap)

| # | Cenário | O que verificar |
|---|---------|------------------|
| 1 | Inserir 10.000 pares (key, value) com keys únicas | `size()` correcto; todos encontráveis via `find` |
| 2 | Inserir a mesma key duas vezes com valores diferentes | Segunda inserção actualiza o valor, `size()` não aumenta |
| 3 | Provocar colisões deliberadas (função de hash trivial tipo `key % 4` num mapa pequeno) | Confirma visualmente (print do array interno) que o Robin Hood swapping está a acontecer correctamente |
| 4 | Remover uma key a meio de uma cadeia de colisões, depois procurar as restantes | Todas as outras keys da cadeia continuam encontráveis (backward shift não partiu nada) |
| 5 | Inserir elementos suficientes para forçar resize | Todos os elementos continuam encontráveis depois do resize; `load_factor()` volta a estar abaixo do limiar |
| 6 | Medir e comparar a distância média de probe (PSL médio) do teu HashMap vs. uma implementação naive de linear probing (sem Robin Hood) | PSL médio deve ser visivelmente menor com Robin Hood sob a mesma carga |
| 7 | `HashMap<std::string, T>` com muitas keys que colidem no `std::hash` por omissão vs. a tua função de hash customizada | Compara distribuição de colisões entre as duas |
| 8 | `operator[]` sobre uma key inexistente | Insere um valor por omissão (comportamento igual a `std::unordered_map`) |

## 2.9 Milestones (HashMap)

- [ ] Estrutura `Slot` com `probe_distance`, array interno com capacidade fixa
- [ ] `insert` com o algoritmo de swapping Robin Hood
- [ ] `find` / `contains` / `operator[]`
- [ ] `erase` com backward shift deletion
- [ ] Resize automático ao ultrapassar load factor
- [ ] Função de hash customizada para pelo menos um tipo, comparada com `std::hash`
- [ ] Suite de testes completa (tabela acima)
- [ ] Benchmark contra `std::unordered_map`

---

# Parte 3 — Benchmark e Comparação Final

## 3.1 Metodologia

- Usa **Google Benchmark** em vez de medir manualmente com `std::chrono` — ele já trata de warm-up, múltiplas iterações e reporta desvio padrão, o que dá números credíveis em vez de ruído.
- Compila sempre em **modo Release** (`-O2` ou `-O3`) para os benchmarks — em modo Debug os números não significam nada.
- Desliga os sanitizers para os benchmarks (eles têm overhead significativo) — usa-os só na suite de correcção.

## 3.2 Datasets a testar

Testa em pelo menos **3 tamanhos** para veres como o comportamento muda com a escala:

| Tamanho | O que revela |
|---|---|
| ~1.000 elementos | Overhead de setup domina; diferenças pequenas entre implementações |
| ~100.000 elementos | Regime "normal" — é onde a maioria das diferenças de design aparece |
| ~10.000.000 elementos | Efeitos de cache e memória tornam-se dominantes; é aqui que Robin Hood normalmente mostra vantagem clara sobre chaining |

## 3.3 Operações a medir

Para o Vector:
- `push_back` sequencial (N elementos, do zero)
- `push_back` com `reserve` prévio (isola o custo de crescimento do custo de inserção)
- Acesso aleatório via `operator[]`
- Iteração completa (`for (auto& x : vec)`)
- Inserção/remoção no meio

Para o HashMap:
- `insert` sequencial de N pares
- `find` de chaves existentes (hit) e inexistentes (miss) — mede os dois separadamente, o comportamento é diferente
- `erase` seguido de `find` (confirma que backward shift não degradou performance)
- Iteração completa

## 3.4 Tabela de resultados (template a preencher)

| Operação | Dataset | `Vector` próprio | `std::vector` | Diferença |
|---|---|---|---|---|
| push_back sequencial | 100k | ... ns/op | ... ns/op | ...% |
| push_back com reserve | 100k | | | |
| acesso aleatório | 100k | | | |
| iteração completa | 100k | | | |

| Operação | Dataset | `HashMap` próprio | `std::unordered_map` | Diferença |
|---|---|---|---|---|
| insert sequencial | 100k | | | |
| find (hit) | 100k | | | |
| find (miss) | 100k | | | |
| erase + find | 100k | | | |

## 3.5 Análise a escrever (não só números)

O benchmark sozinho não é o entregável — a análise é. No README final, responde por escrito a:

- Onde é que a tua implementação ficou mais lenta que a da standard library, e porquê (na maioria dos casos vai ficar — as implementações da STL têm décadas de micro-optimizações). O objectivo não é vencer, é perceberes exactamente **de onde vem** a diferença.
- Em que operações o Robin Hood hashing mostrou vantagem clara sobre uma implementação naive de linear probing (se implementaste essa comparação no teste 6 da secção 2.8)?
- Como é que o growth factor do Vector (1.5x vs. 2x) afectou o número total de reallocs e o desperdício de memória final (`capacity() - size()`)?

---

# Critérios de Conclusão do Projecto

O projecto está "feito o suficiente" quando:

1. Ambos os containers passam a suite de testes completa **com AddressSanitizer e UndefinedBehaviorSanitizer activos, sem nenhum warning ou erro**.
2. O benchmark cobre os 3 tamanhos de dataset e as operações listadas em 3.3, com resultados reais preenchidos na tabela (não estimados).
3. Existe um README com: a análise escrita da secção 3.5, uma explicação por palavras próprias de como funciona o Robin Hood hashing (sem copiar de nenhuma fonte), e a justificação da escolha de growth factor do Vector.
4. Pelo menos um teste demonstra explicitamente a invalidação de iteradores (secção 1.4) e um teste demonstra o backward shift deletion a funcionar correctamente sem partir outras keys (secção 2.8, teste 4).

## Extensões opcionais (se quiseres ir mais além)

- Implementar um **allocator customizado** simples para o Vector (liga com o projecto seguinte da trilha, o Memory Allocator Framework).
- Adicionar suporte a **custom allocators** via parâmetro de template, como a STL real faz (`std::vector<T, Allocator>`).
- Implementar **small buffer optimization** no HashMap (evitar heap allocation para mapas muito pequenos).
- Tornar o HashMap **thread-safe** com fine-grained locking ou lock-free (projecto avançado à parte, não subestimes a complexidade disto).

## Recursos de estudo

- **Robin Hood Hashing**: o paper original é de Pedro Celis (1986); para uma explicação mais moderna e acessível, procura o artigo de blog "Robin Hood Hashing should be your default Hash Table implementation" (Sebastian Sylvan) e a implementação de referência `robin-hood-hashing` no GitHub (útil para comparar depois de teres a tua própria versão feita — não antes).
- **Move semantics**: a talk "Everything You Ever Wanted to Know About Move Semantics" (CppCon, Nicolai Josuttis) é uma boa referência para consolidar depois de implementares, não como tutorial passo-a-passo.
- **cppreference.com**: para confirmar comportamento exacto de `std::vector` e `std::unordered_map` (complexidade garantida de cada operação, regras de invalidação de iteradores) sempre que tiveres dúvida sobre o que a standard exige.
