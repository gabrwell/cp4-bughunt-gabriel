# Checkpoint 4 - Bug Hunt Stream

## Identificação

**Grupo:** StreamF

| Integrante | RM | Turma |
|---|---|---|
| Pedro Henrique dos Santos Cardoso | 563268 | 2CCPG |
| Gabriel Gibin Leoncio | 565462 | 2CCPG |
| Rafael do Nascimento Silva | 566263 | 2CCPG |
| Rai Augusto Ribeiro | 562870 | 2CCPG |
| Guilherme Morais de Assis | 564198 | 2CCPG |
| Lucas Werpp Franco | 556044 | 2CCPG |

| Campo | Resultado |
|---|---|
| **Total de bugs corrigidos** | **12 / 12** |
| **Total de ajustes de Clean Code** | **6 / 6** |

> **Instrução para o professor:** antes de executar o projeto, substitua `SEU_RM` e `SUA_SENHA` no arquivo `src/main/resources/application.properties` por credenciais válidas do banco Oracle da FIAP.

## Parte 1 - Bugs encontrados

| # | Sintoma observado (o que fiz/vi) | Causa raiz (arquivo e linha aproximada) | Correção aplicada | Conceito da disciplina |
|---|---|---|---|---|
| bug01 | Ao buscar `GET /api/conteudos/999`, a resposta vinha vazia e não dizia que o conteúdo não existia. | `ConteudoController.java`, método `buscarPorId` (linhas 32-34): o `catch (Exception)` pegava a exceção e depois retornava `null`. | Tiramos o `try/catch` e deixamos a `ConteudoNaoEncontradoException` chegar ao tratamento global, que devolve 404. | Exceções e tratamento global. |
| bug02 | A busca por `/categoria/FICCAO` voltava vazia mesmo com filmes cadastrados. | `ConteudoController.java`, método `listarPorCategoria` (linha 40): o código comparava duas `String` com `==`. | Usamos o método `findByCategoria` que já existia no repository. | Comparação de objetos e Spring Data JPA. |
| bug03 | O usuário era cadastrado sem o banco gerar seu ID. | `Usuario.java` (linhas 12-14): faltava a anotação de geração no campo `id`. | Colocamos `@GeneratedValue(strategy = GenerationType.IDENTITY)`. | JPA e chave primária. |
| bug04 | O nome chegava no POST, mas era salvo como `null`. | `Usuario.java`, construtor (linhas 23-26): estava escrito `nome = nome`. | Mudamos para `this.nome = nome` para preencher o atributo do objeto. | Escopo e uso do `this`. |
| bug05 | Um usuário com zero créditos conseguia alugar e ficava com saldo negativo. | `Usuario.java`, `temCreditosSuficientes` (linhas 29-31): a comparação estava ao contrário. | Corrigimos para `this.creditos >= preco` e testamos também quando o saldo é exatamente igual ao preço. | Regra de negócio e operadores relacionais. |
| bug06 | Conteúdo marcado como indisponível ainda podia ser alugado. | `Usuario.java`, método `alugar` (linhas 37-40): não existia nenhuma conferência de disponibilidade. | Adicionamos a verificação e lançamos `ConteudoIndisponivelException`. | Validação de regra de negócio. |
| bug07 | O aluguel por usuário menor de idade retornava erro 500 sem uma mensagem útil. | `ClassificacaoIndicativaException.java` era checked e não tinha tratamento no `GlobalExceptionHandler`. | Ela passou a estender `RuntimeException` e agora o handler devolve HTTP 422 com a mensagem da classificação. | Exceções checked e unchecked. |
| bug08 | A promoção aumentava o valor do filme em vez de dar desconto. | `Filme.java`, `aplicarPromocao` (linhas 27-29): multiplicava por `1.2`. | Trocamos o fator para 0,80. Um filme de R$ 14,90 passou a ficar R$ 11,92 na promoção. | Interface e cálculo percentual. |
| bug09 | A série era salva sem título, categoria, duração e classificação. | `Serie.java`, construtor (linhas 17-19): faltava chamar o construtor de `Conteudo`. | Chamamos `super(...)` com os dados recebidos. | Herança e construtores. |
| bug10 | Uma série de 5 temporadas custava R$ 9,90 em vez de R$ 24,50. | `Serie.java` tinha `calcularPrecoAluguel(double desconto)`, que era uma sobrecarga e não substituía o método de `Conteudo`. | Retiramos o parâmetro e colocamos `@Override`. | Sobrescrita e sobrecarga. |
| bug11 | O documentário custava R$ 9,90, mas deveria ser gratuito. | `Documentario.java` não sobrescrevia `calcularPrecoAluguel` e usava o preço padrão de `Conteudo`. | Criamos o método na classe e retornamos `0.0`. | Herança e polimorfismo. |
| bug12 | O cadastro aceitava duração igual a zero ou negativa. | `Conteudo.java`, construtor e setter (linhas 24-29 e 55-60): não havia validação. | Validamos no setter, chamamos esse setter no construtor e devolvemos HTTP 400 quando o valor é inválido. | Encapsulamento e validação. |

## Parte 2 - Ajustes de Clean Code

| # | Onde estava | Qual princípio/boas práticas era violado | O que eu mudei |
|---|---|---|---|
| clean01 | `Conteudo.duracaoMinutos` e os métodos de cadastro. | O atributo era público e qualquer classe podia mudar o valor sem passar pela validação. | Deixamos o campo `private` e passamos a usar `getDuracaoMinutos()`. |
| clean02 | `Usuario.alugar`, com as variáveis `c` e `p`. | Os nomes não explicavam o que cada valor representava. | Trocamos por `conteudo` e `precoAluguel`. |
| clean03 | Repositories dos três controllers. | As dependências eram injetadas diretamente nos campos com `@Autowired`. | Mudamos para injeção pelo construtor e deixamos os campos como `final`. |
| clean04 | Valores `9.90`, `5.00`, `4.90` e `0.8` em `Filme` e `Serie`. | Eram números soltos no meio do código. | Criamos constantes com nomes que explicam cada preço e desconto. |
| clean05 | `Usuario.debitarCreditos`. | O comentário dizia que o método adicionava créditos, mas o código fazia o contrário. | Apagamos o comentário incorreto porque o nome do método já explica o que ele faz. |
| clean06 | Final de `ConteudoController`. | Existia um método antigo sem uso e um bloco grande de código comentado. | Removemos o código morto; se ele for necessário novamente, ainda está no histórico do Git. |

---

## Parte 3 - Perguntas de reflexão

### 1. Injeção de dependência (Aula 13)

Os repositories do projeto são interfaces, então não daria para fazer simplesmente `new ConteudoRepository()`.
Quando a aplicação inicia, o Spring encontra essas interfaces e cria uma implementação delas para trabalhar com o JPA.
Essa implementação é registrada como um bean e depois entregue ao controller pelo construtor.
Por isso o `ConteudoController` consegue chamar `save` e `findById` sem saber qual é a classe criada pelo Spring.
Se o objeto fosse criado manualmente, ele não teria a configuração do banco, o `EntityManager` e o controle de transação.
Preferimos a injeção pelo construtor porque fica fácil ver o que cada controller precisa para funcionar.

### 2. JDBC vs Spring Data JPA (Aulas 12 e 13)

No JDBC nós teríamos que abrir a conexão, escrever o SQL, preencher o `PreparedStatement` e ler o `ResultSet`.
Também seria nossa responsabilidade fechar os recursos e transformar cada linha retornada em um objeto.
Com `JpaRepository`, métodos como `save`, `findAll` e `findById` já vêm prontos.
O `findByCategoria` funciona porque o Spring lê o nome do método e monta a consulta usando o atributo `categoria`.
Mesmo assim, JDBC ainda pode ser útil em uma consulta muito específica ou quando queremos controlar exatamente o SQL executado.
Para o CRUD deste projeto, o JPA deixou os repositories pequenos e evitou bastante código repetido.

### 3. Exceções checked vs unchecked (Aula 11)

Quando uma exceção estende `Exception`, o Java obriga a capturar ou colocar `throws` no método.
Isso estava acontecendo com `ClassificacaoIndicativaException` e mesmo assim a API terminava em um erro 500 genérico.
Como esse caso é uma quebra de regra de negócio, mudamos a classe para estender `RuntimeException`.
Assim ela pode sair do model e chegar ao `GlobalExceptionHandler` sem espalhar `throws` pelos métodos.
No handler associamos essa exceção ao status 422 e retornamos a mensagem dentro do campo `erro`.
No teste com um usuário de 12 anos, a resposta passou a explicar qual classificação impediu o aluguel.

### 4. Sobrescrita vs sobrecarga (Aula 7)

Na sobrescrita, a classe filha usa a mesma assinatura do método da classe pai e troca seu comportamento.
Na sobrecarga, o nome é igual, mas os parâmetros são diferentes, então os dois métodos continuam existindo.
`Conteudo` tinha `calcularPrecoAluguel()`, mas `Serie` tinha `calcularPrecoAluguel(double desconto)`.
Por causa desse parâmetro extra, o Java continuava chamando o método de `Conteudo` e retornava R$ 9,90.
Retiramos o parâmetro e adicionamos `@Override`; depois disso 5 temporadas passaram a custar R$ 24,50.
Se a anotação estivesse no código original, o compilador teria apontado o erro antes da aplicação rodar.

### 5. Onde blindar o objeto? (Aulas 3, 4 e 13)

A duração precisa ser validada sempre, porque não faz sentido existir um conteúdo com zero minutos ou duração negativa.
Colocamos essa regra no setter e o construtor usa o próprio setter, evitando escrever a mesma validação duas vezes.
O setter é importante porque o Jackson pode criar o objeto pelo construtor vazio e preencher os campos depois.
Também deixamos `duracaoMinutos` como `private` para ninguém alterar o campo diretamente e pular essa regra.
No aluguel, o saldo é conferido antes de chamar `debitarCreditos`, evitando que o usuário fique com créditos negativos.
Os campos nulos que encontramos também vieram de construtores: faltava `this.nome` em `Usuario` e faltava chamar `super` em `Serie`.
Por isso não basta validar só no controller; o próprio model precisa impedir que um objeto inválido seja montado.

### 6. Abstração e interface (Aulas 8 e 9)

`Conteudo` é abstrata porque guarda o que filme, série e documentário têm em comum, como título e classificação.
Ela também define comportamentos que cada tipo pode sobrescrever, como o cálculo do preço do aluguel.
Já `Promocionavel` só representa a capacidade de participar de uma promoção.
Hoje quem implementa essa interface é `Filme` e `Serie`, enquanto `Documentario` fica de fora.
Se documentário entrasse em uma promoção, seria necessário implementar `Promocionavel` nessa classe e criar `aplicarPromocao`.
O controller, o repository e o endpoint de preço não precisariam mudar, porque `calcularPrecoPromocional` já verifica a interface.

---

## Parte 4 - Espaço livre

A parte que mais deu trabalho foi perceber que alguns erros compilavam normalmente, principalmente o método da série e a comparação de `String`.
Também tivemos que acertar a configuração do Java, porque o Windows estava encontrando o Java 8 antes do JDK mais novo.
Para conferir as correções, rodamos a API com H2 e fizemos 15 testes de cadastro, consulta, preço e aluguel.
Todos os 15 cenários passaram. As credenciais reais do Oracle ficaram somente na configuração local e não foram enviadas para o GitHub.
