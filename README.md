# Checkpoint 4 - Bug Hunt StreamFIAP

## Identificação

**Grupo:** Entrega individual

| Integrante | RM | Turma |
|---|---|---|
| Gabriel | 565462 | Não informada |

| Campo | Resultado |
|---|---|
| **Total de bugs corrigidos** | **12 / 12** |
| **Total de ajustes de Clean Code** | **6 / 6** |

---

## Parte 1 - Bugs encontrados

| # | Sintoma observado (o que fiz/vi) | Causa raiz (arquivo e linha aproximada) | Correção aplicada | Conceito da disciplina |
|---|---|---|---|---|
| bug01 | `GET /api/conteudos/999` devolvia resposta vazia, sem explicar que o item não existia. | `ConteudoController.java`, método `buscarPorId` (linhas 32-34): um `catch (Exception)` vazio engolia `ConteudoNaoEncontradoException` e o método retornava `null`. | Removi o `try/catch` genérico e deixei a exceção customizada chegar ao `GlobalExceptionHandler`, que responde 404 com a mensagem. | Exceções, propagação e responsabilidade do tratamento global. |
| bug02 | `GET /api/conteudos/categoria/FICCAO` não encontrava conteúdos que estavam gravados nessa categoria. | `ConteudoController.java`, método `listarPorCategoria` (linha 40): a implementação anterior comparava `String` com `==` e varria todos os registros em memória. | Passei a usar `conteudoRepository.findByCategoria(categoria)`. | Igualdade de objetos e query derivada do Spring Data JPA. |
| bug03 | O cadastro de usuário não tinha um ID gerado de forma automática. | `Usuario.java` (linhas 12-14): o campo tinha apenas `@Id`, sem estratégia de geração. | Adicionei `@GeneratedValue(strategy = GenerationType.IDENTITY)`. | Mapeamento ORM e geração de chave primária. |
| bug04 | O usuário era salvo com `nome: null`, mesmo enviando um nome válido no JSON. | `Usuario.java`, construtor (linhas 23-26): havia autoatribuição `nome = nome`, que não alterava o atributo. | Corrigi para `this.nome = nome`. | Escopo, sombreamento de variáveis e estado do objeto. |
| bug05 | Usuário sem saldo conseguia alugar e ficava negativo; em outros casos, saldo suficiente era recusado. | `Usuario.java`, `temCreditosSuficientes` (linhas 29-31): os operandos da comparação estavam invertidos. | A regra agora retorna `this.creditos >= preco`, inclusive aceitando saldo exatamente igual ao preço. | Regra de negócio, operadores relacionais e invariantes. |
| bug06 | Um conteúdo com `disponivel=false` ainda podia ser alugado. | `Usuario.java`, início de `alugar` (linhas 37-40): a disponibilidade nunca era verificada. | Incluí a guarda `if (!conteudo.isDisponivel())` e lancei `ConteudoIndisponivelException` com mensagem clara. | Encapsulamento de regra de negócio e exceções customizadas. |
| bug07 | A tentativa de aluguel por menor de idade virava erro genérico 500, sem a explicação da classificação. | `ClassificacaoIndicativaException.java` (linha 3) era checked e não havia handler correspondente; hoje o tratamento está em `GlobalExceptionHandler.java` (linhas 29-31). | A exceção passou a estender `RuntimeException` e ganhou um `@ExceptionHandler` que responde 422 com a mensagem da regra. | Exceções checked/unchecked e tratamento HTTP centralizado. |
| bug08 | A promoção de filme aumentava o preço em 20%: um filme de R$ 14,90 passava a R$ 17,88. | `Filme.java`, `aplicarPromocao` (linhas 27-29): o preço era multiplicado por `1.2`. | O fator foi corrigido para 0,80; o mesmo exemplo agora retorna R$ 11,92. | Polimorfismo por interface e regra percentual. |
| bug09 | Série cadastrada perdia título, categoria, duração e classificação, além de ficar indisponível. | `Serie.java`, construtor (linhas 17-19): não havia chamada ao construtor de `Conteudo`. | Adicionei `super(titulo, categoria, duracaoMinutos, classificacaoEtaria, true)`. | Herança, encadeamento de construtores e inicialização de estado. |
| bug10 | Série com 5 temporadas custava R$ 9,90 em vez de R$ 24,50. | `Serie.java`, `calcularPrecoAluguel` (linhas 23-25): a versão anterior recebia um parâmetro extra e apenas sobrecarregava o método da superclasse. | Removi o parâmetro e adicionei `@Override`, aplicando R$ 4,90 por temporada. | Sobrescrita, sobrecarga e polimorfismo. |
| bug11 | Documentário herdava o preço padrão de R$ 9,90, embora o contrato diga que ele é gratuito. | `Documentario.java` (linhas 18-20): não existia implementação própria de `calcularPrecoAluguel`. | Sobrescrevi o método para retornar `0.0`. | Classe abstrata, especialização e despacho dinâmico. |
| bug12 | Era possível cadastrar conteúdo com `duracaoMinutos=0` ou negativo. | `Conteudo.java`, construtor e setter (linhas 24-29 e 55-60): o valor era aceito sem validação. | Centralizei a guarda no setter, reutilizei-a no construtor e tratei entrada inválida como HTTP 400 com a mensagem "A duração deve ser maior que zero". | Validação de estado, invariantes e fail-fast. |

## Parte 2 - Ajustes de Clean Code

| # | Onde estava | Qual princípio/boas práticas era violado | O que eu mudei |
|---|---|---|---|
| clean01 | `Conteudo.duracaoMinutos` e os três métodos de cadastro em `ConteudoController`. | Encapsulamento: o atributo era público e podia ignorar a validação do setter. | Tornei o atributo `private` e substituí acessos diretos por `getDuracaoMinutos()`. |
| clean02 | `Usuario.alugar`, com parâmetro `c` e variável `p`. | Nomes curtos não comunicavam intenção. | Renomeei para `conteudo` e `precoAluguel` em todo o método. |
| clean03 | Campos de repositories nos três controllers. | Field injection escondia dependências e dificultava teste/imutabilidade. | Adotei injeção por construtor e campos `final` em `ConteudoController`, `UsuarioController` e `AluguelController`. |
| clean04 | Números `9.90`, `5.00`, `4.90` e `0.8` espalhados em `Filme` e `Serie`. | Números mágicos não explicavam a regra. | Criei constantes nomeadas como `PRECO_BASE`, `ADICIONAL_ESTREIA`, `PRECO_POR_TEMPORADA` e `FATOR_PROMOCIONAL`. |
| clean05 | `Usuario.debitarCreditos`. | O comentário afirmava que o método adicionava créditos, mas o código debitava; comentário incorreto é pior que ausência de comentário. | Removi o comentário enganoso e mantive o código autoexplicativo. |
| clean06 | Final de `ConteudoController`. | Código morto e grande bloco comentado aumentavam ruído e manutenção sem participar do comportamento atual. | Removi `calcularDescontoAntigo` e o protótipo comentado de cupons; o histórico do Git preserva versões antigas. |

---

## Parte 3 - Perguntas de reflexão

### 1. Injeção de dependência (Aula 13)

Os controllers precisam de objetos que realmente implementem `ConteudoRepository` e `UsuarioRepository`, que são apenas interfaces no código.  
O Spring lê essas interfaces, cria proxies em tempo de execução, configura JPA, transações e conexão e registra essas instâncias como beans.  
Em seguida, ele encontra o único construtor de cada controller e fornece o bean correto para os parâmetros pedidos.  
Por isso `ConteudoController` pode manter `private final ConteudoRepository conteudoRepository` sem conhecer a classe concreta.  
Um `new ConteudoRepository()` nem compilaria por ser interface; criar uma implementação manual também perderia proxy, `EntityManager` e transações.  
A injeção por construtor ainda deixa a dependência explícita, imutável e substituível em um teste isolado.

### 2. JDBC vs Spring Data JPA (Aulas 12 e 13)

No DAO JDBC é nossa responsabilidade abrir `Connection`, preparar SQL, preencher parâmetros, percorrer `ResultSet` e fechar recursos.  
O Spring Data JPA automatiza esse trabalho comum, além do mapeamento entre linhas e as entidades `Conteudo` e `Usuario`.  
Assim, `ConteudoRepository extends JpaRepository<Conteudo, Long>` já recebe `save`, `findById`, `findAll` e outras operações CRUD.  
`findByCategoria` funciona sem implementação porque o Spring interpreta o nome do método, identifica o atributo `categoria` e gera a consulta.  
JDBC/DAO ainda pode ser melhor quando a consulta é muito específica, usa recursos próprios do banco ou exige controle fino de SQL e desempenho.  
Neste projeto, JPA reduz bastante o código repetitivo e deixa controller e model concentrados nas regras da StreamFIAP.

### 3. Exceções checked vs unchecked (Aula 11)

Uma checked exception que estende `Exception` precisa ser capturada ou declarada com `throws`, o que espalhava a decisão pela chamada de aluguel.  
Já uma unchecked exception estende `RuntimeException` e pode atravessar model e controller até um ponto central de tratamento.  
`ClassificacaoIndicativaException` representa violação de regra de negócio, e não uma falha externa da qual o controller consiga se recuperar.  
Por isso ela passou a estender `RuntimeException`, como as outras exceções de aluguel do projeto.  
No `GlobalExceptionHandler`, o método anotado para essa classe converte a falha em HTTP 422 e no JSON `{"erro": mensagem}`.  
O cliente agora recebe a idade, o título e a classificação que impediram a operação, em vez de um 500 genérico.

### 4. Sobrescrita vs sobrecarga (Aula 7)

Sobrescrita ocorre quando a subclasse mantém a mesma assinatura do método herdado e fornece um comportamento especializado.  
Sobrecarga cria outro método com o mesmo nome, mas parâmetros diferentes, sem substituir o original.  
`Conteudo` declara `calcularPrecoAluguel()` sem argumentos, enquanto `Serie` tinha `calcularPrecoAluguel(double desconto)`.  
Ao chamar o objeto pelo tipo `Conteudo`, Java encontrava a versão sem argumentos da superclasse e devolvia o preço padrão de R$ 9,90.  
A correção removeu o parâmetro e marcou o método de `Serie` com `@Override`, fazendo 5 temporadas custarem R$ 24,50.  
Se `@Override` estivesse presente desde o começo, o compilador teria recusado a assinatura errada e impedido o bug silencioso.

### 5. Onde blindar o objeto? (Aulas 3, 4 e 13)

Uma regra estrutural, como duração maior que zero, deve proteger toda entrada do objeto; por isso o construtor chama o setter validado de `Conteudo`.  
O setter também precisa validar porque frameworks como Jackson e JPA podem criar a instância pelo construtor vazio e preencher os campos depois.  
O encapsulamento de `duracaoMinutos` como `private` impede que outro código contorne essa guarda com uma atribuição direta.  
Créditos não podem ficar negativos por uma operação de aluguel: `temCreditosSuficientes` valida antes e `debitarCreditos` só roda depois da guarda.  
Os campos nulos observados não vinham do JSON, mas de propagação quebrada: `this.nome` não era atribuído e `Serie` não chamava `super`; ambos foram corrigidos na construção.  
Se a API também precisar proibir nome/título nulo enviado pelo cliente, a mesma ideia deve ser aplicada nos respectivos construtores e setters, com mensagem HTTP clara.  
Validar em apenas um ponto deixa caminhos alternativos de criação abertos; a regra deve morar no model e ser alcançada por todos esses caminhos.

### 6. Abstração e interface (Aulas 8 e 9)

`Conteudo` é uma classe abstrata porque reúne identidade, estado e comportamento comum a filme, série e documentário.  
`Promocionavel` é uma interface de capacidade: indica quais tipos aceitam a operação `aplicarPromocao`, sem obrigá-los a ter a mesma hierarquia além disso.  
Hoje `Filme` e `Serie` implementam a interface; `Conteudo.calcularPrecoPromocional` detecta essa capacidade e delega o cálculo.  
Se documentário passasse a ter promoções, `Documentario` declararia `implements Promocionavel` e implementaria `aplicarPromocao`.  
`ConteudoController`, `ConteudoRepository` e o endpoint de preço promocional permaneceriam intactos, pois já trabalham com a abstração.  
Isso mostra baixo acoplamento e abertura para extensão: acrescentamos uma capacidade a um tipo sem reescrever os consumidores existentes.

---

## Parte 4 - Espaço livre

O projeto foi compilado e empacotado com JDK 26 usando `release 17`, conforme o `pom.xml`.  
Também subi a API com H2 em memória e executei 15 verificações manuais do contrato: **15 passaram e 0 falharam**.  
Foram conferidos cadastro, geração de ID, filtro, preços, promoções, duração inválida, item inexistente, indisponibilidade, classificação, créditos e aluguel válido.  
As credenciais reais do Oracle não fazem parte do histórico Git; o arquivo versionado continua com `SEU_RM` e `SUA_SENHA`.
