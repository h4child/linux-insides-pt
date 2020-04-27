Processo de inicialização do kernel Parte 2.
================================================================================

Primeiro passo na inicialização do kernel
--------------------------------------------------------------------------------

Nós começamos mergulhar dentro do kernel linux no [parte anterior](linux-bootstrap-1.md) e vimos a parte inicial da inicialização do código do kernel. Nós paramos na chamada da função `main` (é o primeira função escrita em C) do [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c).

Nesta parte, nós continuaremos inspecionar o código de inicialização do kernel e revisando
* O que o `modo protegido` é
* a transição para ele
* a inicialização do heap e o console
* detectação de memória, validação CPU e a inicialização do teclado
* e muito, muito mais.

Então, vamos adiante.

Modo protegido
--------------------------------------------------------------------------------

Antes podemos mover para o [modo longo](http://en.wikipedia.org/wiki/Long_mode) no Intel64 nativo, o kernel deve interromper CPU no modo protegido.

O que é [modo protegido](https://en.wikipedia.org/wiki/Protected_mode)?
Modo protegido foi o adicionado primeiro na arquitetura x86 em 1982 e foi o principal modo do processador [80286](http://en.wikipedia.org/wiki/Intel_80286) da Intel até Intel 64 e modo longo

O principal motivo para afastar do [modo real](http://wiki.osdev.org/Real_Mode)[(similar pt)](https://pt.wikipedia.org/wiki/Modo_real) é que tem aceso muito limitado para RAM. Como você pode lembrar parte anterior, há apenas 2<sup>20</sup> bytes ou 1 mebibytes, algumas vezes até 640 kibibytes de RAM disponível em modo real.

Modo protegido trouxe muitas mudanças, mas a principal diferença é no gerenciamento de memória. O endereço bus de 20-bit foi substituído com endereços bus de 32-bit. Permitiu acessar 4 gibibytes de memória vs 1 mebibytes em modo real. Também, suporte a [paginação](http://en.wikipedia.org/wiki/Paging)[(similar pt)](https://pt.wikipedia.org/wiki/Pagina%C3%A7%C3%A3o_de_mem%C3%B3ria) foi adicionado, qual você pode ler sobre na próxima seção.

Gerenciamento de memória no modo protegido é divido em duas partes quase independentes:

* segmentação
* paginação

Aqui nós iremos apenas falar sobre segmentação, Paginação será discutido na próxima seção.

Como você leu anteriormente, endereços consistem de duas partes em modo real:

* Endereço base do segmento
* Offset do segmento base

Podemos obter o endereço físico se nós conhecermos essas duas partes por:

```
EndereçoFísico = Base * 16 + Offset
```

Gerenciamento de memória foi completamente refeito em modo protegido. Não há segmento de tamanho fixo de 64 kibibytes. Em vez disso, o tamanho e o local de cada segmento são descrito por uma estrutura de dados associada chamada _Descritor de Segmento_. Esses descritores de segmentos são armazenados em uma estrutura de dados chamado de `Global Descriptor Table` (GDT - tabela global de descritores).

O GDT é uma estrutura o qual reside em memória. Não tem lugar fixo na memória, então o endereço é armazenado no registro especial `GDTR`. Mais tarde iremos ver como o GDT é carregado no código no Kernel do Linux. Haverá uma operação para carrega-lo da memória, alguma coisa como:

```assembly
lgdt gdt
```

Onde a instrução `lgdt` carrega o endereço base e o limite (tamanho) da tabela de descritores globais no registro `GDTR`. `GDTR` é um registro de 48-bit e consiste de duas partes:

 * o tamanho(16-bit) do GDT;
 * o endereço(32-bit) do GDT.

Como mencionado acima, o GDT contém o `descritores de segmentos` que descreve segmentos de memória. Cada descritor tem 64-bits de tamanho. O esquema geral de um descritor é:

```
 63         56         51    48    45           39        32 
-------------------------------------------------------------
|             | |B| |A|        | |   | |0|E|W|A|            |
| BASE 31:24  |G|/|L|V| LIMITE |P|DPL|S| tipo  | BASE 23:16 |
|             | |D| |L|  19:16 | |   | |1|C|R|A|            |
-------------------------------------------------------------

 31                         16 15                         0 
------------------------------------------------------------
|                             |                            |
|        BASE 15:0            |      LIMITE 15:0           |
|                             |                            |
------------------------------------------------------------
```
Não preocupe, eu sei que parece sombrio depois do modo real, mas é fácil. Por exemplo LIMITE 15:0 significa que bits 0-15 do limite do segmento são localizado no começo de descritor. O resto está no LIMITE 19:16, qual é localizado nos bits 48-51 de descritor. Então, o tamanho do limite é 0-19 intos é, 20-bits. Vamos ver mais de perto isso:

1. Limite[20-bits] é dividido entre bits 0-15 e 48-51. Define o `length_of_segment - 1`. Depende no `G` (Granularity) bit.

  * Se `G` (bit 55) é 0 e o limite do segmento é 0, o tamanho do segmento é 1 Byte
  * Se `G` é 1 e o limite de segmento é 0, o tamanho do segmento é 4096 Bytes
  * Se `G` é 0 e o limite do segmento é 0xfffff, o tamanho do segmento é 1 Mebibyte
  * Se `G` é 1 e o limite do segmento é 0xfffff, o tamanho do segmento é 4 Gibibytes

  Então, o que isso significa
  * Se G é 0, Limite é interpretado em termos de 1 Byte e o tamanho máximo do segmento pode ser 1 Mebibyte.
  * Se G é 1, Limite é interpretado em termos de 4096 Bytes = 4KBytes = 1 página e o tamanho máximo do segmento pode ser 4 Gibibytes. Na verdade, quando G é 1, o valor do Limite é deslocado para a esquerda por 12 bits. Então, 20 bits + 12 bits = 32 bits e 2<sup>32</sup> = 4 Gibibytes.

2. Base[32-bits] é dividido entre bits 16-31, 32-39 e 56-63. Define o endereço físico da localização inicial do segmento.

3. Tipo/atributo[5-bits] é representado por bits 40-44. Define o tipo do segmento e como pode ser acessado.
  * flag(bandeira) `S` no bit 44 especifica o tipo do descritor. Se `S` é 0 então esse segmento é um segmento de sistema, enquanto se `S` é 1 então esse é um código ou segmento de dados (segmento de pilha são segmentos de dados qual devem ser segmentos lido/escrever).

Determinar se o segmento é um código ou segmento de dados, nós podemos checar seu atributo EX(bit 43) (marcado como 0 no diagrama acima). Se é 0, então o segmento, de outra forma, é um segmento de código.

Um segmento pode ser de um dos seguintes tipos:

```
--------------------------------------------------------------------------------------
|         Campo Tipo          |   Tipo de   |            Descrição                   |
|                             |  Descritor  |                                        |
|-----------------------------|-------------|----------------------------------------|
| Decimal                     |             |                                        |
|             0    E    W   A |             |                                        |
| 0           0    0    0   0 | Dados       | ler-apenas                             |
| 1           0    0    0   1 | Dados       | ler-apenas, acessado                   |
| 2           0    0    1   0 | Dados       | ler/escrever                           |
| 3           0    0    1   1 | Dados       | ler/escrever, acessado                 |
| 4           0    1    0   0 | Dados       | ler-apenas, expandir-baixo             |
| 5           0    1    0   1 | Dados       | ler-apenas, expandir-baixo, acessado   |
| 6           0    1    1   0 | Dados       | ler/escrever, expandir-baixo           |
| 7           0    1    1   1 | Dados       | ler/escrever, expandir-baixo, acessado |
|                  C    R   A |             |                                        |
| 8           1    0    0   0 | Código      | executar-apenas                        |
| 9           1    0    0   1 | Código      | executar-apenas, acessado              |
| 10          1    0    1   0 | Código      | executar/ler                           |
| 11          1    0    1   1 | Código      | executar/ler, acessado                 |
| 12          1    1    0   0 | Código      | executar-apenas, conforme              |
| 14          1    1    0   1 | Código      | executar-apenas, conforme, acessado    |
| 13          1    1    1   0 | Código      | executar/ler, conforme                 |
| 15          1    1    1   1 | Código      | executar/ler, conforme, acessado       |
--------------------------------------------------------------------------------------
```

Como nós vemos o primeiro bit(bit 43) é `0` para um segmento de _data_ (dados) e `1` para um segmento de _code_ (código). O próximos 3 bits (40, 41, 42) são `EWA` (*E*xpansion[expansão] *W*ritable[gravável] *A*ccessible[acessível]) ou CRA(*C*onforming[conforme] *R*eadable[legível] *A*ccessible[acessível]).

  * Se E(bit 42) é 0, expande para cima, de outra forma, expande para baixo. Leia mais [aqui](http://www.sudleyplace.com/dpmione/expanddown.html).
  * Se W(bit 41)(para segmentos de dados) é 1, acesso escrita é permitido no segmento de dados e se é 0, o segmento é apenas leitura. Note que o acesso a leitura é sempre permitido nos segmentos da dados.
  * A(bit 40) controla se o segmento pode ser acessado pelo processador ou não.
  * C(bit 43) é o bit conforme(para seletor de códigos). Se C é 1, o segmento de código pode ser executado do nível de privilégio baixo (exemplo usuário). Se C é 0, pode apenas ser executado do mesmo nível de privilégio.
  * R(bit 41) controla o acesso a leitura para segmento de código; quando é 1, o segmento pode ser lido. O acesso a escrita nunca é concedida pelo segmento de código.

4. DPL[2-bits] (Nível de privilégio do descritor) consta o bits 45-46. Define o nível de privilégio do segmento. Pode ser 0-3 onde 0 é o nível de privilégio maior.

5. A flag(bandeira) P(bit 47) indica se o segmento está presente  na memória ou não. Se P é 0, o segmento vai ser declarada como _invalid_ (inválido) e o processador recusa ler o segmento.

6. A flag(bandeira) AVL(bit 52) - bits disponível e reservado. É ignorado em linux.

7. A flag(bandeira) L(bit 53) indica se o segmento de código contém código 64-bit nativo. Se está definido, então o segmento de código executa em modo 64-bit.

8. A flag(bandeira) D/B(bit 54) (default/Big flag) representa o tamanho do operando, ou seja 16/32 bits. Se definido, tamanho do operando é 32 bits. Senão é 16 bits.

Registro de segmentos contém seletores de segmentos como em modo real. Contudo, em modo protegido, um seletor de segmento está lidando diferentemente. Cada descritor de segmento tem um seletor de segmento associado que é uma estrutura de 16-bits:

```
 15                3 2  1     0
-------------------------------
|  índice (index)  | TI | RPL |
-------------------------------
```

Onde,
* **índice** armazena o número do índice de descritor no GDT.
* **TI**(indicador de tabela [table indicator]) indica onde procurar para o descritor. Se for 0, então o descritor será pesquisado na tabela global de descritores (GDT).
* **RPL** contém o nível de privilégio do requisitante.

Todo registro de segmento tem uma parte visível e uma parte oculto.
* Visível - o seletor de segmento é armazenado aqui.
* oculto  - O descritor de segmento (o qual contém a base, limite, atributos e flags) é armazenado aqui.

O passos seguintes são necessário obter um endereço físico em modo protegido:

* O seletor de segmento deve ser carregado em um dos registros de segmentos.
* A CPU tenta encontrar um descritor de segmento no deslocamento `Endereço GDT + Índice` do seletor e então carrega o descritor na parte *oculta* do registro de segmento.
* Se a paginação é desativada, o endereço linear do segmento ou endereço físico, é dado pela fórmula:

 Endereço básico (encontra no descritor obtido nos passos anteriores) + Offset.

Esquema será assim:

![Endereço Linear](images/linear_address.png)

O algoritmo para a transição do modo real para modo protegido é:

* Desativar interruptores
* descreve e carrega o GDT com a intrução `lgdt`
* Defini o bit PE (ativar proteção - Protection Enable) em CR0 (Controle registro 0 - Control Register 0)
* Pula para o código do modo protegido

Nós iremos ver a transição completa para o modoprotegido no Kernel do Linux na próxima parte, mas antes nós mover para o modo protegido, nós iremos fazer algumas outras preparações.

Deixe-nos olhar no [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). Nós vemos algumas rotinas o qual realiza inicialização de teclados, inicialização heap, etc... Vamos dar uma olhada.

Copiando parâmetros boot para o "zeropage"
--------------------------------------------------------------------------------

Nós começaremos da rotina `main` em "main.c". A primeira função que é chamada em `main` é [`copy_boot_params(void)`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). Copia o cabeçalho do kernel no campos correspondente da estrutura `boot_params` que é definido no arquivo de cabeçalho [arch/x86/include/uapi/asm/bootparam.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h).

A estrutura `boot_params` contém o campo `struct setup_header hdr`. Essa estrutura contém o mesmo campo definido no [linux boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) e é preenchido por um boot loader e também no tempo de compilação do kernel. `copy_boot_params` faz duas coisas:

1. Copia `hdr` do [header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L280) para o campo `setup_header` na estrutura `boot_params`.

2. Atualiza o ponteiro para o linha de comando do kernel, se o kernel foi carregado com o antigo protocolo de linha de comando.

Note que copia `hdr` com o função `memcpy`, definido no arquivo [copy.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S). Vamos dar uma olhada dentro:

```assembly
GLOBAL(memcpy)
    pushw   %si
    pushw   %di
    movw    %ax, %di
    movw    %dx, %si
    pushw   %cx
    shrw    $2, %cx
    rep; movsl
    popw    %cx
    andw    $3, %cx
    rep; movsb
    popw    %di
    popw    %si
    retl
ENDPROC(memcpy)
```
Yeah, nós movemos para código C e agora assembly de novo :) primeiro de tudo, nós podemos ver que `memcpy` e outras rotinas que são definido aqui, começa e termina com 2 macros: `GLOBAL` e `ENDPROC`. `GLOBAL` é descrito em [arch/x86/include/asm/linkage.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/linkage.h) que define a diretiva `globl` e label. `ENDPROC` é descrito em [include/linux/linkage.h](https://github.com/torvalds/linux/blob/v4.16/include/linux/linkage.h) e marca o símbolo `name` como um nome de uma função e termina com o tamanho do simbolo `name`.

A implementação do `memcpy` é simples. Em primeiro, envia valores do registros `si` e `di` para a stack preservar seus valores por que eles vão mudar durante o `memcpy`. Como nós podemos ver no `REALMODE_CFLAGS` no `arch/x86/Makefile`, o sistema de compilação do kernel usa a opção `-mregparm=3` do GCC, então a função obtém os três primeiro parâmetro dos registros `ax`, `dx` e `cx`. Chamar  `memcpy`  parece com isso:

```c
memcpy(&boot_params.hdr, &hdr, sizeof hdr);
```
Então,
* `ax` contém o endereço do `boot_params.hdr`
* `dx` contém o endereço do `hdr`
* `cx` contém o tamanho do `hdr` em bytes.

`memcpy` coloca o endereço do `boot_params.hdr` no `di` e salva `cx` na stack. Depois disso, ele altera o valor para a direita 2 vezes (ou o divide por 4) e copia quatro bytes do endereço em `si` para o endereço em `di`. Depois disso, nós restauramos o tamanho do `hdr` de novo, alinha por 4 bytes e copia o resto dos bytes do endereço no `si` para o endereço no `di` bytes por byte (se houver mais). Agora o valor do `si` e `di` são restaurado  da stack e a operação de cópia concluída.

Inicialização do Console
--------------------------------------------------------------------------------

Depois `hdr` é copiado no `boot_params.hdr`, o próximo passo é para inicializar o console chamando a função `console_init`, definido em [arch/x86/boot/early_serial_console.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/early_serial_console.c).
 
Tenta encontrar a opção `earlyprintk` na linha de comando e se a busca foi bem-sucedida, analise o endereço da porta e a taxa de transmissão da porta serial e inicializa a porta serial. O valor da opção da linha de comando `earlyprintk` pode ser um desses:

* serial,0x3f8,115200
* serial,ttyS0,115200
* ttyS0,115200

Depois inicializa a porta serial, nós podemos ver o primeira saída:

```C
if (cmdline_find_option_bool("debug"))
    puts("early console in setup code\n");
```

A definição de `puts` é em [tty.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c). Como nós vermos imprimir caracteres por caracteres em um loop chamando a função `putchar`. Deixe-nos ver a implementação do  `putchar`.

```C
void __attribute__((section(".inittext"))) putchar(int ch)
{
    if (ch == '\n')
        putchar('\r');

    bios_putchar(ch);

    if (early_serial_base != 0)
        serial_putchar(ch);
}
```
`__attribute__((section(".inittext")))` significa que esse código será na seção `.inittext`. Nós podemos encontrar no arquivo linker [setup.ld](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld).

Primeiro de tudo, checa `putchar` para o símbolo `\n` e se é encontrado, imprimi `\r` depois. Depois que imprimi os caracteres na tela VGA chamando a BIOS com a chamada interrupt `0x10`:

```C
static void __attribute__((section(".inittext"))) bios_putchar(int ch)
{
    struct biosregs ireg;

    initregs(&ireg);
    ireg.bx = 0x0007;
    ireg.cx = 0x0001;
    ireg.ah = 0x0e;
    ireg.al = ch;
    intcall(0x10, &ireg, NULL);
}
```
Aqui o `initregs` pega a estrutura `biosregs` e preenche o primeiro `biosregs` com zeros usando a função `memset` e então preenche com valores de registros.

```C
    memset(reg, 0, sizeof *reg);
    reg->eflags |= X86_EFLAGS_CF;
    reg->ds = ds();
    reg->es = ds();
    reg->fs = fs();
    reg->gs = gs();
```
Vamos observar a implementação do [memset](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/copy.S#L36):

```assembly
GLOBAL(memset)
    pushw   %di
    movw    %ax, %di
    movzbl  %dl, %eax
    imull   $0x01010101,%eax
    pushw   %cx
    shrw    $2, %cx
    rep; stosl
    popw    %cx
    andw    $3, %cx
    rep; stosb
    popw    %di
    retl
ENDPROC(memset)
```
Como você pode ler acima, usa a mesma convenção de chamada como a função `memcpy`, que significa que a função tem parâmetro dos registros `ax`, `dx` e `cx`.

A implementação do `memset` é semelhante à do memcpy. Salva o valor do registro `di` na stack e coloca o valor do `ax`, que armazena o endereço da estrutura `biosregs` em  `di`. Próximo é a instrução `movzbl`, que copia o valor do `dl` para o byte mais baixo do registro `eax`. Os 3 bytes altos restantes de `eax` serão preenchidos com zeros.

A próxima instrução multiplica `eax` com `0x01010101`. Precisa por que  `memset` vai copiar 4 bytes no mesmo tempo. Por exemplo, se nós necessitar preencher uma estrutura cujo tamanho seja 4 bytes com o valor `0x7` com memset, `eax` vao conter o `0x00000007`. Então se nós multiplicarmos `eax` com `0x01010101`, nós iremos obter `0x07070707` e agora nós podemos copiar esses 4 bytes na estrutura. `memset` usa a instrução `rep; stosl` para copiar `eax` em `es:di`.

O resto da função `memset` quase a mesma coisa que `memcpy`.

Depois a estrutura `biosregs` é preenchido com `memset`, `bios_putchar` chama o interrupt [0x10](http://www.ctyme.com/intr/rb-0106.htm) que imprimi um caracteres. Mais tarde checa se a porta serial foi inicializada ou não e grava um caracteres com [serial_putchar](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/tty.c) e a instrução `inb/outb` se foi definida.

Inicialização do Heap
--------------------------------------------------------------------------------

Após a seção stack e bss ter preparado em [header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) (veja [parte anterior](linux-bootstrap-1.md)), o kernel precisa inicializar o [heap](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c) com a função [`init_heap`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c).

Primeiro de tudo `init_heap` checa a flag [`CAN_USE_HEAP`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h#L24) da estrutura [`loadflags`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) no cabeçalho de inicialização do kernel e calcula o final da stack se essa flag foi definida:

```C
    char *stack_end;

    if (boot_params.hdr.loadflags & CAN_USE_HEAP) {
        asm("leal %P1(%%esp),%0"
            : "=r" (stack_end) : "i" (-STACK_SIZE));
```

ou em outras palavras `stack_end = esp - STACK_SIZE`.

Então tem o cálculo `heap_end`:

```C
     heap_end = (char *)((size_t)boot_params.hdr.heap_end_ptr + 0x200);
```

que significa `heap_end_ptr` ou `_end`  + `512` (`0x200h`). A última verificação é se `heap_end` é maior que  `stack_end`. Se estiver, então `stack_end` é atribuído para `heap_end` para faze-los iguais.

Agora o heap está inicializado e podemos usar o método `GET_HEAP`. Veremos para que é usado, como usá-lo e como é implementado nos próximos posts.

Validação da CPU
--------------------------------------------------------------------------------

O próximo passo, como podemos ver, é a validação da CPU através da função `validate_cpu` do arquivo [arch/x86/boot/cpu.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpu.c).

Chama a função [`check_cpu`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/cpucheck.c) e passa o nível da CPU e requer o nível da CPU e verifica se o kernel é iniciado no nível correto da CPU.

```C
check_cpu(&cpu_level, &req_level, &err_flags);
if (cpu_level < req_level) {
    ...
    return -1;
}
```

A função `check_cpu` checa a flag da CPU, a presença do [long mode (modo longo)](http://en.wikipedia.org/wiki/Long_mode) no caso da CPU x86_64 (64 bits), verifica o fornecedor do processador e faz os preparativos para determinados fornecedores, como desativar o SSE + SSE2 para AMD se estiverem ausentes, etc.

No próximo passo chamamos a função `set_bios_mode` depois que o código de inicialização descobriu que a CPU é adequada. A função está implementada apenas para o modo `x86_64`:

```C
static void set_bios_mode(void)
{
#ifdef CONFIG_X86_64
	struct biosregs ireg;

	initregs(&ireg);
	ireg.ax = 0xec00;
	ireg.bx = 2;
	intcall(0x15, &ireg, NULL);
#endif
}
```

A função `set_bios_mode` executa o interrupt BIOS `0x15` informar a BIOS que [long mode](https://en.wikipedia.org/wiki/Long_mode) (se `bx == 2`) será usado. 

Detecção de memória
--------------------------------------------------------------------------------

O próximo passo é detecção de memória através da função `detect_memory`. `detect_memory` fornece um mapa de RAM disponível para a CPU. Usa diferentes interfaces de programação para detecção de memória como `0xe820`, `0xe801` e `0x88`. Abordaremo apenas a implementação da interface **0xE820** aqui.

Aqui está a implementação da função `detect_memory_e820` da fonte [arch/x86/boot/memory.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/memory.c). Primeiro de tudo, a função `detect_memory_e820` inicializa estrutura `biosregs` como visto acima e registros preenchidos com valores especais da chamada `0xe820`:

```assembly
    initregs(&ireg);
    ireg.ax  = 0xe820;
    ireg.cx  = sizeof buf;
    ireg.edx = SMAP;
    ireg.di  = (size_t)&buf;
```
* `ax` contém o número da função (0xe820 em nosso caso)
* `cx` contém o tamanho do buffer que vai conter dados sobre a memória
* `edx` deve conter o numero mágico `SMAP`
* `es:di` deve conter o endereço do buffer que vai conter dados de memória
* `ebx` tem que ser zero.

Próximo é um loop aonde dados sobre a memória ser colecionado. Começa com uma chamada para o interrupt BIOS `0x15`, que escreve uma linha de tabela de alocação de endereços. Para obter o próxima linha precisamos chamar o interrupt de novo (que nós fizemos no loop). Antes da próxima chamada  `ebx` contém o valor de retorno anteriormente:

```C
    intcall(0x15, &ireg, &oreg);
    ireg.ebx = oreg.ebx;
```

Ultimamente, essa coleção de dados da função da tabela de alocação de endereços e escreve esses dados em array `e820_entry`:

* Começa do segmento de memória
* Tamanho do segmento de memóra
* Tipo de segmento de memória (se o segmento particular é usable[utilizável] ou reserved[reservado])

Você pode ver o resultado disso na saída `dmesg`, alguma coisa como:

```
[    0.000000] e820: BIOS-provided physical RAM map:
[    0.000000] BIOS-e820: [mem 0x0000000000000000-0x000000000009fbff] usable
[    0.000000] BIOS-e820: [mem 0x000000000009fc00-0x000000000009ffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000000f0000-0x00000000000fffff] reserved
[    0.000000] BIOS-e820: [mem 0x0000000000100000-0x000000003ffdffff] usable
[    0.000000] BIOS-e820: [mem 0x000000003ffe0000-0x000000003fffffff] reserved
[    0.000000] BIOS-e820: [mem 0x00000000fffc0000-0x00000000ffffffff] reserved
```

Inicialização do teclado
--------------------------------------------------------------------------------

O próximo passo é a inicialização do teclado com a chamada da função [`keyboard_init`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). Primeiro `keyboard_init` inicializa registros usando a função `initregs`. Então chamamos o interruptor [0x16](http://www.ctyme.com/intr/rb-1756.htm) para consultar o status do teclado.

```c
    initregs(&ireg);
    ireg.ah = 0x02;     /* Get keyboard status */
    intcall(0x16, &ireg, &oreg);
    boot_params.kbd_status = oreg.al;
```

Depois disso chama [0x16](http://www.ctyme.com/intr/rb-1757.htm) novamente definir
a taxa de repetição e atraso (daley).

```c
    ireg.ax = 0x0305;   /* Set keyboard repeat rate */
    intcall(0x16, &ireg, NULL);
```

Consulta
--------------------------------------------------------------------------------

Os próximos passos são consultas para parametros diferentes. Não entraremos em detalhes sobre essas consulas, mas voltaremos a eles em partes posteriores. Vamos dar uma rápida olhada nessas funções:

O primeiro passo é obter a informação [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep) chmando a função `query_ist`. Checa o nível do CPU e se é correto, chama `0x15` para obter a info e salva o resultado para `boot_params`.

Próximo, a função [query_apm_bios](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/apm.c#L21) recebe a informação [Advanced Power Management](http://en.wikipedia.org/wiki/Advanced_Power_Management) da BIOS. `query_apm_bios` chama interruption da BIOS `0x15` também,  mas com `ah` = `0x53` para checar instalação do `APM`. Depois o `0x15` dinaliza a execução, a função `query_apm_bios` checa a assinatura `PM` (deve ser `0x504d`), o carry flag (deve ser 0 se suportar `APM`) e o valor do registro `cx` (se 0x02, a interface modo protegido é suportado).

Próximo, chama `0x15` novamente, mas com `ax = 0x5304` desconecta a interface `APM` e conecta a interface no modo protegido de 32-bit. No final, preenche `boot_params.apm_bios_info` com o valor obtido da BIOS.

Note que `query_apm_bios` será executado apenas se a compilação for definido as flags `CONFIG_APM` ou `CONFIG_APM_MODULE` no arquivo de configuração:

```C
#if defined(CONFIG_APM) || defined(CONFIG_APM_MODULE)
    query_apm_bios();
#endif
```
O último e a função [`query_edd`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/edd.c#L122) que consulta a informação `Enhanced Disk Drive` da BIOS. Iremos detalhar Como `query_edd` é implementado.

Primeiro de tudo, lê a opção [edd](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst) da linha de comando do kernel e se foi definida para `off então ` `query_edd` apenas retorna.

Se o EDD estiver ativado, o `query_edd` percorre os discos rígidos suportados pelo BIOS e consulta as informações do EDD no seguinte loop:

```C
for (devno = 0x80; devno < 0x80 + EDD_MBR_SIG_MAX; devno++) {
    if (!get_edd_info(devno, &ei) && boot_params.eddbuf_entries < EDDMAXNR) {
        memcpy(edp, &ei, sizeof ei);
        edp++;
        boot_params.eddbuf_entries++;
    }
    ...
    ...
    ...
    }
```
Onde `0x80` é o primeiro hd e o valor do macro `EDD_MBR_SIG_MAX` é 16. Coleção dados em um array de estrutura [edd_info](https://github.com/torvalds/linux/blob/v4.16/include/uapi/linux/edd.h). `get_edd_info` checa que EDD é presente por invocar o inerrupt `0x13` com `ah` como `0x41` e se EDD é presente `get_edd_info` novamente chama o interrupt `0x13`, com `ah` `0x41` e se EDD é presente `get_edd_info` chama de novo o interrupt `0x13`, mas com `ah` como `0x48` e `si` contendo o enereço do buffer aonde informação EDD vai ser armazenado.


Conclusão
--------------------------------------------------------------------------------

Esse é o final do segunda parte sobre interior do kernel do Linux. Na próxima parte, veremos a configuração do modo de vídeo e o restante dos preparativos antes da transição para o modo protegido e a transição direta para ele.

Se você ter qualquer erro de tradução me escreva um comentário ou um ping no [twitter](https://twitter.com/rodgger1).

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Protected mode](http://wiki.osdev.org/Protected_Mode)
* [Long mode](http://en.wikipedia.org/wiki/Long_mode)
* [Nice explanation of CPU Modes with code](http://www.codeproject.com/Articles/45788/The-Real-Protected-Long-mode-assembly-tutorial-for)
* [How to Use Expand Down Segments on Intel 386 and Later CPUs](http://www.sudleyplace.com/dpmione/expanddown.html)
* [documentação earlyprintk](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/earlyprintk.txt)
* [Parâmetro do  Kernel](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/kernel-parameters.rst)
* [Serial console](https://github.com/torvalds/linux/blob/v4.16/Documentation/admin-guide/serial-console.rst)
* [Intel SpeedStep](http://en.wikipedia.org/wiki/SpeedStep)
* [APM](https://en.wikipedia.org/wiki/Advanced_Power_Management)
* [Especificação EDD](http://www.t13.org/documents/UploadedDocuments/docs2004/d1572r3-EDD3.pdf)
* [Documentação TLDP para Linux Boot Process](http://www.tldp.org/HOWTO/Linux-i386-Boot-Code-HOWTO/setup.html) (velho)
* [Parte anterior](linux-bootstrap-1.md)
