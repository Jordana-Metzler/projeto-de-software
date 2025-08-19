# projeto-de-software
Aplicando o conceito solid em código
## Solid

# Classe Banco: #
**Antes:**  
A classe criava e fechava contas e também imprimia coisas na tela.

public class Banco {
    private String nome;
    private List<Conta> contas = new ArrayList<>();

    public Banco(String nome) {
        this.nome = nome;
    }

    public void criarConta(Conta conta) {
        contas.add(conta);
        System.out.println("Conta criada: " + conta);
    }

    public void fecharConta(String numero) {
        // ...
        System.out.println("Conta fechada: " + numero);
    }

    public void listarContas() {
        for (Conta c : contas) System.out.println(c);
    }

    public void listarClientes() {
        Set<String> infos = new LinkedHashSet<>();
        for (Conta c : contas) {
            infos.add(c.getCliente().getInfo());
        }
        for (String info : infos) System.out.println(info);
    }

    public Conta encontrarContaPorNumero(String numero) {
        for (Conta c : contas) {
            if (c.getNumero().equals(numero)) return c;
        }
        return null;
    }
}

**Depois:**  
Agora ela só guarda e organiza as contas. Quem imprime é outra parte do programa.

public class Banco {
    private final String nome;
    private final List<Conta> contas = new ArrayList<>();

    public Banco(String nome) { this.nome = Objects.requireNonNull(nome); }

    public void adicionarConta(Conta conta) { contas.add(conta); }

    public boolean removerConta(String numero) {
        return contas.removeIf(c -> c.getNumero().equals(numero));
    }

    public List<Conta> getContas() { return Collections.unmodifiableList(contas); }

    public Optional<Conta> encontrarContaPorNumero(String numero) {
        return contas.stream().filter(c -> c.getNumero().equals(numero)).findFirst();
    }

    public Set<String> infosClientes() {
        Set<String> infos = new LinkedHashSet<>();
        for (Conta c : contas) infos.add(c.getCliente().getInfo());
        return infos;
    }
}

**Conceito:**
SRP (Responsabilidade Única). A classe Banco só cuida das contas. A parte de imprimir ficou fora. Assim o código fica mais limpo e organizado.

## Cliente
**Antes:** 
Tinha os dados e também uma forma de mostrar as informações do cliente.

public class Cliente {
    private String nome;
    private String cpf;
    private String endereco;
    private String telefone;

    public Cliente(String nome, String cpf, String endereco, String telefone) {
        this.nome = nome;
        this.cpf = cpf;
        this.endereco = endereco;
        this.telefone = telefone;
    }

    public String getInfo() {
        return String.format("Cliente: %s | CPF: %s | Endereço: %s | Telefone: %s",
                nome, cpf, endereco, telefone);
    }

    public String getNome() { return nome; }
}

**Depois:** 
Continua com os dados, mas não mistura outras coisas (como salvar ou imprimir).

public class Cliente {
    private final String nome;
    private final String cpf;
    private final String endereco;
    private final String telefone;

    public Cliente(String nome, String cpf, String endereco, String telefone) {
        this.nome = Objects.requireNonNull(nome);
        this.cpf = Objects.requireNonNull(cpf);
        this.endereco = Objects.requireNonNull(endereco);
        this.telefone = Objects.requireNonNull(telefone);
    }

    public String getInfo() {
        return String.format("Cliente: %s | CPF: %s | Endereço: %s | Telefone: %s",
                nome, cpf, endereco, telefone);
    }

    public String getNome() { return nome; }
}

**Conceito:** 
SRP. O cliente é só o cliente, não faz mais nada além de guardar suas informações.


## Conta
**Antes:** 
A conta tinha depósitos, saques, transferências e histórico, mas podia ter comportamentos diferentes dependendo da conta filha, às vezes sem deixar claro.

public abstract class Conta {
    protected String numero;
    protected double saldo;
    protected Cliente cliente;
    protected String tipo;
    private List<Transacao> historico = new ArrayList<>();

    public Conta(String numero, Cliente cliente, String tipo) {
        this.numero = numero;
        this.cliente = cliente;
        this.tipo = tipo;
        this.saldo = 0.0;
    }

    // depositar/sacar/transferir + historico + podeSacar + toString com 'tipo'...
}

**Depois:**
Criei interfaces menores (depositar, sacar, transferir) e defini regras claras que todas as contas têm que seguir.

public abstract class Conta implements Depositable, Withdrawable, Transferable {
    protected final String numero;
    protected double saldo;
    protected final Cliente cliente;
    private final List<Transacao> historico = new ArrayList<>();

    protected Conta(String numero, Cliente cliente) {
        this.numero = Objects.requireNonNull(numero);
        this.cliente = Objects.requireNonNull(cliente);
    }

    @Override
    public boolean depositar(double valor, String descricao) {
        validarValorPositivo(valor);
        saldo += valor;
        registrarTransacao("DEPOSITO", valor, descricao);
        return true;
    }

    @Override
    public boolean sacar(double valor, String descricao) throws SaldoInsuficienteException {
        validarValorPositivo(valor);
        if (!podeSacar(valor)) throw new SaldoInsuficienteException();
        saldo -= valor;
        registrarTransacao("SAQUE", valor, descricao);
        return true;
    }

    @Override
    public boolean transferir(Conta destino, double valor, String descricao) throws SaldoInsuficienteException {
        validarValorPositivo(valor);
        if (!podeSacar(valor)) throw new SaldoInsuficienteException();
        this.saldo -= valor;
        destino.saldo += valor;
        registrarTransacao("TRANSFERENCIA", valor, descricao + " -> " + destino.numero);
        destino.registrarTransacao("TRANSFERENCIA", valor, descricao + " <- " + this.numero);
        return true;
    }

    protected abstract boolean podeSacar(double valor);

    protected void registrarTransacao(String tipo, double valor, String descricao) {
        historico.add(new Transacao(valor, tipo, descricao));
    }

    protected void validarValorPositivo(double v) {
        if (v <= 0) throw new IllegalArgumentException("valor inválido");
    }

    public String getNumero() { return numero; }
    public Cliente getCliente() { return cliente; }
    public double getSaldo() { return saldo; }

    @Override
    public String toString() {
        return String.format("%s %s | Titular: %s | Saldo: %.2f",
            getClass().getSimpleName(), numero, cliente.getNome(), saldo);
    }
}

**Conceito:** 
LSP e ISP. Agora todas as contas seguem as mesmas regras de contrato (LSP) e cada parte do sistema pode usar só as funções que precisa (ISP).

## Conta Corrente

**Antes:**
Permitida sacar além do saldo, mas estava misturado no mesmo método.

public class ContaCorrente extends Conta {
    private double limite;

    public ContaCorrente(String numero, Cliente cliente, double limite) {
        super(numero, cliente, "ContaCorrente");
        this.limite = limite;
    }

    @Override
    protected boolean podeSacar(double valor) {
        return (saldo + limite) >= valor;
    }
}

**Depois:**
Continua igual, mas agora segue certinho o contrato da classe mãe, só mudando a regra de cheque especial.

public class ContaCorrente extends Conta {
    private final double limite;

    public ContaCorrente(String numero, Cliente cliente, double limite) {
        super(numero, cliente);
        this.limite = limite;
    }

    @Override
    protected boolean podeSacar(double valor) {
        return (saldo + limite) >= valor;
    }
}

**Conceito:** 
LSP. A conta corrente funciona como qualquer outra conta, só com um limite a mais.

## Conta Poupança

**Antes:**
Tinha a regra de saque normal e aplicava rendimento.

public class ContaPoupanca extends Conta {
    private double rendimento; // ex: 0.005 = 0,5% ao mês

    public ContaPoupanca(String numero, Cliente cliente, double rendimento) {
        super(numero, cliente, "ContaPoupanca");
        this.rendimento = rendimento;
    }

    @Override
    protected boolean podeSacar(double valor) {
        return saldo >= valor;
    }

    public void aplicarRendimentoMensal() {
        double ganho = saldo * rendimento;
        if (ganho > 0) {
            saldo += ganho;
            registrarTransacao("RENDIMENTO", ganho, "Rendimento mensal poup.");
        }
    }
}


**Depois:**
Mantive a mesma ideia, mas deixei mais clara a função de aplicar juros mensais.

public class ContaPoupanca extends Conta {
    private final double rendimentoMensal;

    public ContaPoupanca(String numero, Cliente cliente, double rendimentoMensal) {
        super(numero, cliente);
        this.rendimentoMensal = rendimentoMensal;
    }

    @Override
    protected boolean podeSacar(double valor) { return saldo >= valor; }

    public void aplicarRendimentoMensal() {
        double ganho = saldo * rendimentoMensal;
        if (ganho > 0) {
            saldo += ganho;
            registrarTransacao("RENDIMENTO", ganho, "Rendimento mensal poup.");
        }
    }
}

**Conceito:**
LSP + SRP. Ela continua respeitando o contrato da conta e cuida só das suas próprias regras.

## Transação
**Antes:**
Criava transações, mas tinha método “registrar” que não fazia nada.

public class Transacao {
    private static int SEQ = 1;

    private int id;
    private LocalDateTime data;
    private double valor;
    private String tipo;
    private String descricao;

    public Transacao(double valor, String tipo, String descricao) {
        this.id = SEQ++;
        this.data = LocalDateTime.now();
        this.valor = valor;
        this.tipo = tipo;
        this.descricao = descricao;
    }

    public void registrar() {
        // neste exemplo, existir no histórico já é "registrar"
    }

    @Override
    public String toString() {
        return String.format("#%d | %s | %s | Valor: %.2f | %s",
                id, data, tipo, valor, descricao);
    }
}


**Depois:**
Agora só guarda os dados da transação e garante que os valores sejam válidos.

public class Transacao {
    private static int SEQ = 1;

    private final int id;
    private final LocalDateTime data;
    private final double valor;
    private final String tipo;
    private final String descricao;

    public Transacao(double valor, String tipo, String descricao) {
        if (valor <= 0) throw new IllegalArgumentException("valor inválido");
        this.id = SEQ++;
        this.data = LocalDateTime.now();
        this.valor = valor;
        this.tipo = Objects.requireNonNull(tipo);
        this.descricao = Objects.requireNonNull(descricao);
    }

    @Override
    public String toString() {
        return String.format("#%d | %s | %s | Valor: %.2f | %s",
                id, data, tipo, valor, descricao);
    }
}


**Conceito:** 
SRP. A transação é só a transação, não mistura registro em banco de dados ou prints.

## App
**Antes:**  
A classe principal criava o banco, imprimia coisas na tela e também fazia operações direto nas contas.


public class App {

    public static void main(String[] args) {
        // cria banco com dados mockados (já imprime as operações iniciais)
        Banco banco = criar_dados_mock();

        System.out.println("\n=== LISTAR CLIENTES ===");
        banco.listarClientes();

        System.out.println("\n=== LISTAR CONTAS ===");
        banco.listarContas();

        // Pegar duas contas para demonstrar as operações
        Conta contaA = banco.encontrarContaPorNumero("001");
        Conta contaB = banco.encontrarContaPorNumero("002");

        contaA.depositar(300, "Depósito inicial");
        contaA.transferir(contaB, 100, "Transferência teste");

        System.out.println("Saldo final conta A: " + contaA.getSaldo());
        System.out.println("Saldo final conta B: " + contaB.getSaldo());
    }
}

**Depois:**
Agora a classe App só inicializa os serviços e chama as operações de forma organizada.

public class App {
    public static void main(String[] args) {
        AccountRepository repo = new InMemoryAccountRepository();
        Notifier notifier = new ConsoleNotifier();
        Logger logger = new ConsoleLogger();

        FeePolicy fee = new BasicFeePolicy();
        TransferService transferService = new TransferService(repo, fee, notifier, logger);

        // popular dados iniciais
        Seed.seed(repo);

        // buscar contas
        Conta origem = repo.findByNumero("001").orElseThrow();
        Conta destino = repo.findByNumero("002").orElseThrow();

        // executar operação
        transferService.transferir(origem, destino, 100.0, "PIX apresentação");

        // mostrar saldos
        System.out.println("Saldo final origem: " + origem.getSaldo());
        System.out.println("Saldo final destino: " + destino.getSaldo());
    }
}

**Conceito:**
DIP e SRP. A App agora só monta os objetos e chama os serviços. As regras de negócio ficam em classes separadas.

Com DIP, o App depende de interfaces, não de implementações fixas.



