<img width="415" height="876" alt="image" src="https://github.com/user-attachments/assets/08472e76-b4cb-4c47-90c4-817c7fc26b21" />
<img width="308" height="566" alt="image" src="https://github.com/user-attachments/assets/d2eab9e1-e5f8-4b7d-8f48-b26a43069124" />
<img width="282" height="562" alt="image" src="https://github.com/user-attachments/assets/fd681e26-9f36-4d24-a15f-c7ef9ff0556f" />
<img width="295" height="567" alt="image" src="https://github.com/user-attachments/assets/43ff9ed7-301e-4639-906b-df5c5db9bce6" />
<img width="282" height="559" alt="image" src="https://github.com/user-attachments/assets/629a72e3-4cd1-47cb-bec6-5ec84608d560" />
<img width="416" height="866" alt="image" src="https://github.com/user-attachments/assets/36ea13f8-a9b8-4cd1-a356-e41990418b00" />
<img width="1600" height="864" alt="image" src="https://github.com/user-attachments/assets/fde622da-8a9f-4264-9797-53b93942cb19" />


# To-Do List (Android)

## Descrição do projeto e objetivo da aplicação

Aplicativo Android de lista de tarefas (to-do list) desenvolvido em Kotlin com Jetpack Compose. O objetivo é permitir que o usuário cadastre, visualize, edite, marque como concluída e exclua tarefas, com os dados persistidos localmente no dispositivo por meio do Room. O projeto foi construído seguindo uma arquitetura em camadas (dados, repositório, ViewModel e UI), separando responsabilidades entre persistência, regras de acesso aos dados, estado da tela e apresentação.

## Tecnologias utilizadas

- **Kotlin**: linguagem principal do projeto.
- **Jetpack Compose**: construção declarativa da interface (telas de lista e formulário).
- **Room**: biblioteca de persistência que abstrai o acesso ao banco de dados SQLite (entidade `Tarefa`, DAO `TarefaDao` e banco `TarefaDatabase`).
- **Coroutines/Flow**: operações assíncronas (inserir, atualizar, deletar) executadas em coroutines, e leitura reativa da lista de tarefas via `Flow`.
- **ViewModel**: mantém o estado da tela sobrevivendo a mudanças de configuração e expõe os dados para a UI via `StateFlow`.
- **Navigation Compose**: gerencia a navegação entre a tela de lista e a tela de formulário, incluindo a passagem do ID da tarefa como argumento de rota.

## Responsabilidade de `TarefaRepository`

O `TarefaRepository` é a camada intermediária entre o `TarefaViewModel` e o `TarefaDao`. Ele expõe a lista de tarefas como um `Flow<List<Tarefa>>` (obtido diretamente do DAO) e disponibiliza funções de suspensão (`inserir`, `atualizar`, `deletar`) que apenas repassam a chamada ao DAO. Sua função é isolar o restante da aplicação dos detalhes de acesso ao banco de dados: o ViewModel não conhece o Room nem o DAO diretamente, apenas conversa com o repositório. Isso facilita manutenção e testes, já que a fonte de dados poderia ser trocada sem alterar o ViewModel ou a UI.

## Responsabilidade de `TarefaViewModel`

O `TarefaViewModel` mantém e expõe o estado da tela de tarefas. Ele transforma o `Flow` de tarefas do repositório em um `StateFlow` (usando `stateIn`, com `SharingStarted.WhileSubscribed(5_000)`), garantindo que a lista permaneça observável de forma eficiente enquanto há uma tela coletando o estado, mesmo durante recomposições ou rotações de tela. Além disso, oferece as funções `inserir`, `atualizar` e `deletar`, cada uma disparando uma coroutine em `viewModelScope` que delega a operação ao repositório. O ViewModel também define uma `factory` estática, responsável por construir a instância do `TarefaDatabase`, criar o `TarefaRepository` a partir do DAO e, por fim, instanciar o próprio `TarefaViewModel` — isso permite que o Android construa o ViewModel com suas dependências sem a necessidade de um framework de injeção de dependências.

## Como `ListaTarefasScreen` observa o estado e dispara ações

A `ListaTarefasScreen` coleta o `StateFlow` de tarefas do `TarefaViewModel` usando `collectAsStateWithLifecycle()`, o que garante que a coleta respeite o ciclo de vida da tela (pausando quando ela não está visível). O valor coletado (`tarefas`) é repassado como parâmetro para o `ListaTarefasContent`, um composable "burro" (stateless) que apenas recebe dados e callbacks — essa separação facilita a visualização em `@Preview` sem depender do ViewModel real. As ações do usuário são disparadas por callbacks:

- Marcar/desmarcar conclusão dispara `onCheckedChange`, que chama `viewModel.atualizar(tarefa.copy(concluida = ...))`.
- Tocar no botão de exclusão dispara `onDeletar`, que chama `viewModel.deletar(tarefa)`.
- Tocar em um item da lista chama `onEditarTarefa`, navegando para o formulário com o ID da tarefa.
- Tocar no botão flutuante (FAB) chama `onNovaTarefa`, navegando para o formulário sem ID (nova tarefa).

Como a lista é um `StateFlow` observado pela UI, qualquer alteração no banco (feita pelo repositório) propaga automaticamente uma nova lista, recompondo a tela sem necessidade de atualização manual.

## Como `FormularioTarefaScreen` diferencia cadastro e edição

A tela recebe um parâmetro `tarefaId` vindo da navegação. Se `tarefaId` for igual a `0`, o formulário está em modo de **cadastro** (nova tarefa); qualquer outro valor indica **edição** de uma tarefa existente. A tela busca a tarefa correspondente dentro da lista atual (`tarefas.find { it.id == tarefaId }`) e usa seus valores (título e descrição) para pré-preencher os campos quando em modo de edição. O composable `FormularioTarefaContent` recebe a flag `isEdicao` apenas para ajustar o título da barra superior ("Nova Tarefa" ou "Editar Tarefa"), mantendo a lógica de decisão isolada do componente visual. Ao salvar, se `tarefaId == 0` uma nova `Tarefa` é inserida via `viewModel.inserir`; caso contrário, a tarefa existente é atualizada com `copy(titulo = ..., descricao = ...)` via `viewModel.atualizar`, preservando o ID e os demais campos originais (como `concluida` e `dataCriacao`).

## Rotas configuradas em `AppNavigation` e passagem do ID da tarefa

O `AppNavigation` define um `NavHost` com destino inicial `"lista"` e duas rotas:

- **`"lista"`**: exibe a `ListaTarefasScreen`. Define o que acontece ao pedir uma nova tarefa (navega para `"formulario/0"`) e ao pedir a edição de uma tarefa (navega para `"formulario/{id}"`, substituindo `{id}` pelo ID real da tarefa clicada).
- **`"formulario/{tarefaId}"`**: exibe a `FormularioTarefaScreen`. O ID é declarado como parte do caminho da rota (argumento `tarefaId`) e é recuperado a partir do `backStackEntry.arguments`, convertido de `String` para `Int` (com `0` como valor padrão de segurança). Esse ID é repassado ao formulário, que o utiliza para decidir entre os modos de cadastro e edição, como descrito acima.

A navegação de volta (tanto ao cancelar quanto ao salvar) é feita chamando `navController.popBackStack()`, retornando à tela de lista.

## Como `MainActivity` cria a ViewModel e inicia a navegação

Na `MainActivity`, dentro de `setContent`, o tema do aplicativo (`FiaptodolistTheme`) envolve todo o conteúdo. A instância do `TarefaViewModel` é criada com a função `viewModel()` do Compose, passando a `factory` estática definida no próprio `TarefaViewModel` (`TarefaViewModel.factory(applicationContext)`). Essa fábrica é responsável por obter o banco de dados (`TarefaDatabase.getDatabase`), criar o `TarefaRepository` a partir do DAO e, por fim, o `TarefaViewModel`. Com o ViewModel pronto, a `MainActivity` chama `AppNavigation(viewModel = viewModel)`, iniciando o `NavHost` que controla a navegação entre as telas de lista e formulário, com o ViewModel compartilhado entre ambas.

## Instruções básicas para executar o projeto

1. Abra o projeto no Android Studio (versão compatível com AGP 9.2.1 e Kotlin 2.2.10).
2. Aguarde a sincronização automática do Gradle (o wrapper já está incluído no projeto, via `gradlew`/`gradlew.bat`).
3. Conecte um dispositivo físico com depuração USB habilitada ou inicie um emulador Android (API mínima 24, API alvo 36).
4. Selecione o módulo `app` e clique em **Run** (ou execute `./gradlew installDebug` pelo terminal, na raiz do projeto).
5. O aplicativo abrirá na tela de lista de tarefas. Use o botão flutuante (+) para cadastrar uma nova tarefa, toque em um item para editá-lo, use a caixa de seleção para marcar conclusão e o ícone de lixeira para excluir.

Para executar os testes automatizados:

- Testes de unidade: `./gradlew test`
- Testes instrumentados (incluindo `TarefaDaoTest`): `./gradlew connectedAndroidTest` (requer emulador ou dispositivo conectado)

## Seção de evidências

Adicione nesta seção as capturas de tela produzidas durante a execução da atividade (por exemplo: tela de lista vazia, tela de lista com tarefas, formulário de cadastro, formulário de edição e tarefa marcada como concluída). Sugestão de estrutura:

```
docs/evidencias/
├── lista-vazia.png
├── lista-com-tarefas.png
├── formulario-cadastro.png
├── formulario-edicao.png
└── tarefa-concluida.png
```

E referencie as imagens aqui, por exemplo:

```markdown
### Lista de tarefas
![Lista de tarefas](docs/evidencias/lista-com-tarefas.png)

### Formulário de cadastro
![Cadastro de tarefa](docs/evidencias/formulario-cadastro.png)

### Formulário de edição
![Edição de tarefa](docs/evidencias/formulario-edicao.png)
```
