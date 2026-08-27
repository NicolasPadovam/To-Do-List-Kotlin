# To-Do List — Kotlin + Jetpack Compose

Aplicativo Android de lista de tarefas construído com Jetpack Compose e persistência local em banco de dados. O objetivo é permitir que o usuário cadastre, visualize, edite, marque como concluída e exclua tarefas, com os dados salvos no próprio dispositivo — ou seja, a lista continua igual mesmo depois de fechar e reabrir o app.

A aplicação foi organizada seguindo a arquitetura **MVVM (Model - View - ViewModel)** com uma camada de repositório, de modo que cada parte do código tenha uma responsabilidade única e bem definida:

```
UI (Compose)  →  ViewModel  →  Repository  →  DAO / Room  →  SQLite
     ↑                                                          |
     └────────────────── Flow (fluxo reativo) ──────────────────┘
```

---

## Tecnologias utilizadas

| Tecnologia | Para que foi usada no projeto |
|---|---|
| **Kotlin** | Linguagem base de toda a aplicação. |
| **Jetpack Compose** | Construção da interface de forma declarativa, sem XML de layout. |
| **Room** | Camada de persistência local (abstração sobre o SQLite) com a entidade `Tarefa` e o `TarefaDao`. |
| **Coroutines / Flow** | Execução das operações de banco fora da thread principal e entrega reativa da lista de tarefas para a UI. |
| **ViewModel** | Guarda e expõe o estado da tela, sobrevivendo a mudanças de configuração (ex.: girar a tela). |
| **Navigation Compose** | Controle das rotas entre a tela de listagem e a tela de formulário, incluindo a passagem de argumentos. |

---

## Arquitetura e responsabilidades

### `TarefaRepository`

É a **única porta de entrada para os dados** da aplicação. O repositório recebe o `TarefaDao` e expõe métodos de alto nível (listar, buscar por ID, inserir, atualizar e excluir), escondendo do restante do app o detalhe de que os dados vêm do Room.

Responsabilidades:

- Centralizar o acesso ao banco de dados, evitando que a ViewModel converse diretamente com o DAO.
- Expor a lista de tarefas como um `Flow<List<Tarefa>>`, de forma que qualquer alteração no banco seja propagada automaticamente.
- Isolar a origem dos dados: se um dia a fonte deixar de ser o Room e passar a ser uma API, apenas o repositório muda — ViewModel e telas continuam iguais.

### `TarefaViewModel`

É a ponte entre o repositório e a interface. Ela **não conhece Compose** e **não conhece Room**: apenas mantém o estado e executa as ações.

Responsabilidades:

- Expor o estado da lista para a UI (o `Flow` do repositório é convertido em `StateFlow` com `stateIn`, usando `viewModelScope` e um valor inicial de lista vazia).
- Disparar as operações de escrita (`salvar`, `atualizar`, `excluir`, `alternarConclusao`) dentro do `viewModelScope`, em coroutines, para não travar a thread principal.
- Buscar uma tarefa específica pelo ID quando a tela de formulário é aberta em modo de edição.
- Sobreviver a recriações da Activity, mantendo o estado carregado.

### `ListaTarefasScreen`

Tela principal, responsável por **observar o estado e disparar ações** — nunca por manipular dados diretamente.

Como isso funciona na prática:

- A tela coleta o `StateFlow` da ViewModel com `collectAsState()` (ou `collectAsStateWithLifecycle()`), o que transforma o fluxo em um `State` observável pelo Compose.
- Sempre que o banco muda, o `Flow` emite uma nova lista, o `State` é atualizado e o Compose **recompõe automaticamente** apenas o que precisa ser redesenhado. Não existe nenhuma chamada manual de "atualizar lista".
- As interações do usuário são repassadas como eventos para a ViewModel:
  - tocar no checkbox → `viewModel.alternarConclusao(tarefa)`;
  - tocar no item → navega para o formulário passando o ID da tarefa;
  - tocar no ícone de lixeira → `viewModel.excluir(tarefa)`;
  - tocar no `FloatingActionButton` → navega para o formulário sem ID (modo cadastro).

### `FormularioTarefaScreen`

Tela única que atende os dois cenários — **cadastro e edição** — e diferencia um do outro pelo **ID recebido como argumento de navegação**.

A lógica é a seguinte:

- Se o ID recebido for nulo ou igual a `-1` (valor padrão usado na rota de cadastro), a tela entende que é uma **nova tarefa**: os campos iniciam vazios, o título vira "Nova Tarefa" e o botão de salvar chama a inserção no repositório.
- Se vier um ID válido, um `LaunchedEffect(id)` busca a tarefa correspondente na ViewModel e preenche os campos de título e descrição com os valores existentes. O título vira "Editar Tarefa" e o botão de salvar chama a atualização, mantendo o mesmo ID.
- Os campos são controlados por `remember { mutableStateOf(...) }`, e ao final da ação a tela chama `navController.popBackStack()` para voltar à listagem — que já estará atualizada, graças ao `Flow`.

### `AppNavigation` e a passagem do ID

O `AppNavigation` concentra o `NavHost` e a definição das rotas:

| Rota | Finalidade |
|---|---|
| `lista` | Tela inicial (`startDestination`) com a listagem das tarefas. |
| `formulario` | Abre o formulário em modo **cadastro**. |
| `formulario/{tarefaId}` | Abre o formulário em modo **edição**, com o ID da tarefa na URL da rota. |

A passagem do ID acontece assim:

1. Na rota com argumento, o parâmetro é declarado com `navArgument("tarefaId") { type = NavType.IntType }`.
2. Na listagem, ao tocar em um item, a navegação é feita com `navController.navigate("formulario/${tarefa.id}")`.
3. Dentro do `composable`, o valor é recuperado com `backStackEntry.arguments?.getInt("tarefaId")` e repassado para a `FormularioTarefaScreen`, que decide entre cadastro e edição.

### `MainActivity`

É o ponto de entrada do app. Suas responsabilidades:

- Chamar `setContent { }` e aplicar o tema do Compose.
- Montar as dependências: obter a instância do banco Room, pegar o `TarefaDao` e criar o `TarefaRepository`.
- Criar a `TarefaViewModel` já com o repositório injetado, usando uma `ViewModelProvider.Factory` (necessária porque a ViewModel tem parâmetro no construtor).
- Criar o `NavController` com `rememberNavController()` e chamar o `AppNavigation`, passando o controller e a ViewModel — iniciando assim a navegação do app.

---

## Como executar o projeto

**Pré-requisitos**

- Android Studio (versão Ladybug ou mais recente)
- JDK 17
- Um emulador Android ou dispositivo físico com **API 24 (Android 7.0)** ou superior

**Passo a passo**

1. Clone o repositório:
   ```bash
   git clone https://github.com/NicolasPadovam/To-Do-List-Kotlin.git
   ```
2. Abra o Android Studio e selecione **File → Open**, apontando para a pasta do projeto.
3. Aguarde o **Gradle Sync** terminar (o download das dependências acontece automaticamente).
4. Selecione um emulador ou conecte um dispositivo com a depuração USB ativada.
5. Clique em **Run ▶** (ou pressione `Shift + F10`).

Também é possível gerar o APK pelo terminal:

```bash
./gradlew assembleDebug
```

O arquivo será gerado em `app/build/outputs/apk/debug/`.

---

## Estrutura de pastas

```
To-Do-List-Kotlin/
├── app/
│   └── src/main/java/.../
│       ├── data/
│       │   ├── Tarefa.kt                 # Entidade do Room
│       │   ├── TarefaDao.kt              # Consultas ao banco
│       │   ├── AppDatabase.kt            # Configuração do Room
│       │   └── TarefaRepository.kt       # Camada de acesso a dados
│       ├── ui/
│       │   ├── ListaTarefasScreen.kt     # Tela de listagem
│       │   └── FormularioTarefaScreen.kt # Tela de cadastro/edição
│       ├── viewmodel/
│       │   └── TarefaViewModel.kt        # Estado e ações da UI
│       ├── navigation/
│       │   └── AppNavigation.kt          # NavHost e rotas
│       └── MainActivity.kt               # Ponto de entrada
├── docs/
│   └── evidencias/                       # Prints da aplicação em execução
├── gradle/
├── build.gradle.kts
├── settings.gradle.kts
└── README.md
```

---

**Autor:** Nicolas Varella Barros Padovam
