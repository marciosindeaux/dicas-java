## As vezes a liguagem é a sua melhor amiga
Digamos que você tenha uma classe de controllers como a seguinte:

```java
@Controller
@RequestMapping("/usuario")
public class UsuarioController {

    public final UsuarioService usuarioService;

    public UsuarioController(UsuarioService usuarioService) {
        this.usuarioService = usuarioService;
    }

    @GetMapping
    public ResponseEntity buscarPessoas() {
        try {
            return ResponseEntity
                .ok()
                .body(usuarioService.buscarPessoas());
        }catch (Exception e){
            return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Não foi possival buscar");
        }
    }

    @PostMapping
    public ResponseEntity cadastrarPessoa(@RequestBody Pessoa pessoa) {
        try {
            return ResponseEntity
                .status(HttpStatus.CREATED)
                .body(usuarioService.cadastrarPessoa(pessoa));
        } catch (Exception e) {
            return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Não foi possivel cadastrar");
        }
    }
}
```

Eu sei, há muitas coisas a serem corrigidas em uma classe como essa, mas uma coisa que incomoda a todos (inclusive o Sonar) é a quantidade de repetição desnecessária de blocos try-catch. Até seus testes unitários ficam tristes com essa implementação. Agora, escale isso para projetos medios ou grandes e vamos ter centenas , talvez milhares de linhas repetidas APENAS NOS CONTROLLERS.

Todos nós sabemos que você sempre quis dizer para aquele seu chefe que esse tipo de repetição é o que acaba com as estatísticas do portão de qualidade, mas como abordar isso com ele? Além do mais, ele vai me pedir uma solução. Como implementar algo melhor, já que quase sempre estamos sem tempo?

Eis a solucao : Tenha algum conhecimento sobre as interfaces funcionais no java e como elas funcionam. Algumas interfaces podem atender de forma excepecional em varias situações como essa, como as classes `Supplier<T>` e `Runnable`

```java
public class AbstractController {

    protected <T> ResponseEntity<T> tryRequestWithStatus(Supplier<T> supplier, HttpStatus status){
        try {
            return ResponseEntity
                    .status(status)
                    .body(supplier.get());
        }catch (Exception e) {
            return tratamentoExcessao(e);
        }
    }

    protected ResponseEntity tryRequestWithStatus(Runnable runnable, HttpStatus status){
        try {
            runnable.run();
            return ResponseEntity.status(status).build();
        }catch (Exception e) {
            return tratamentoExcessao(e);
        }
    }
    private ResponseEntity tratamentoExcessao(Exception e) {
        /**
         *  Tente implementar sua solução de tratamentos de excessoes
         *  particulares aqui , como no exemplo abaixo.
         *
            if(e instanceof MinhaException) {
                return ResponseEntity
                        .status(HttpStatus.I_AM_A_TEAPOT)
                        .body("Talvez eu nao deva usar esse status");
            }
        */
        return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body("Falha critica não planejada");
    }

}
```
Seja criativo na implementação dessa classe para atender as suas necessidades, esse é só um exemplo. Sinta-se vontade para usa-lo caso necessario.

Com essa pequena classe sendo herdada, a classe `UsuarioController` passa a ficar da seguinte maneira : 

```java
@Controller
@RequestMapping("/usuario")
public class UsuarioController extends AbstractController{

    public final UsuarioService usuarioService;

    public UsuarioController(UsuarioService usuarioService) {
        this.usuarioService = usuarioService;
    }

    @GetMapping
    public ResponseEntity buscarPessoas() {
        return tryRequestWithStatus(usuarioService::buscarPessoas, HttpStatus.OK);
    }

    @PostMapping
    public ResponseEntity cadastrarPessoa(@RequestBody Pessoa pessoa) {
        return tryRequestWithStatus(() -> usuarioService.cadastrarPessoa(pessoa), HttpStatus.CREATED);
    }
}
```

Intrigante, não é? Imagine agora a quantidade de linhas que serão reduzidas nos projetos médios ou grandes. Essa abordagem simplifica o principalmente o tratamento de exceções repetitivas. Com a classe `AbstractController`, você pode centralizar o tratamento de exceções comuns do seu projeto.

