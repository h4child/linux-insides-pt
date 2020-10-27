Processo bootng do kernel. Parte 4
================================================================================

A transição para o modo 64-bit
--------------------------------------------------------------------------------

Esse é o quarto parte do `Processo booting fo kernel`. Aqui, nós aprenderemos sobre as primeiras etapas ocupadas no [modo protegido](http://en.wikipedia.org/wiki/Protected_mode), como checar se CPU suporta [modo longo](http://en.wikipedia.org/wiki/Long_mode) e [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions). Iniciaremos a tabela de páginas com [paging](http://en.wikipedia.org/wiki/Paging) e, no final, transição do CPU para [modo longo](https://en.wikipedia.org/wiki/Long_mode).

**NOTE: Será vários códigos de assembly nesta parte, então se você não está familiar com isso, poderá querer consultar um livro sobre isso**

Na [parte anterior](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md) paramos no pulo para o ponto de entrada  no [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S):

```assembly
jmpl	*%eax
```

Recorda que o registro `eax` contém o endereço do ponto de entrada de 32-bit. Ler sobre isso no [protocolo de boot Linux kernel x86](https://www.kernel.org/doc/Documentation/x86/boot.txt):

```
Quando usamos bzImage, o kernel em modo protegido foi realocado para 0x100000
```

Vamos certificar que é assim, observando os valores do registro no ponto de entrada 32-bit:

```
eax            0x100000	1048576
ecx            0x0	    0
edx            0x0	    0
ebx            0x0	    0
esp            0x1ff5c	0x1ff5c
ebp            0x0	    0x0
esi            0x14470	83056
edi            0x0	    0
eip            0x100000	0x100000
eflags         0x46	    [ PF ZF ]
cs             0x10	16
ss             0x18	24
ds             0x18	24
es             0x18	24
fs             0x18	24
gs             0x18	24
```

Vemos aqui que o registro `cs` contém um valor de `0x10` (você vai recordar da [parte anterior](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md), esse é o segundo índice no `Global Descriptor Table`), o registro `eip` possui o valor `0x100000` e o endereço básico de todos os segmentos incluindo o segmento de código são zero.

Então, o endereço físico onde o kernel é carregado seria `0:0x100000` ou `0x100000`, como especificado pelo protocolo boot. Agora vamos começar com o ponto de entrada 32-bit.


O ponto de entrada 32-bit
--------------------------------------------------------------------------------

O ponto de entrada 32-bit é definido no [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) código assembly:

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
....
....
....
ENDPROC(startup_32)
```
Primeiro, por que o diretório está `compactado`? A resposta para isso é que `bzimage` é um pacote gzipped consistente do `vmlinux`, `header` e `código de inicialização do kernel`, Vimos no código de inicialização do kernel em todo parte anteriores. O principal objetivo do código no `head_64.S` é para preparar para entrar em modo longo, entra e então descompacta o kernel. Nós iremos acompanhar todos essas etapas conduzindo descompressão do kernel nesta parte:

Você encontrará 2 arquivos no diretório `arch/x86/boot/compressed`:

* [head_32.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_32.S)
* [head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S)

Mas nós iremos considerar apenas o codigo fonte `head_64.S`, porquê como você pode lembrar, esse livro é apenas relacionada: vemos no [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/Makefile). Nós podemos encontrar seguindo o alvo `make` aqui:

```Makefile
vmlinux-objs-y := $(obj)/vmlinux.lds $(obj)/head_$(BITS).o $(obj)/misc.o \
	$(obj)/string.o $(obj)/cmdline.o \
	$(obj)/piggy.o $(obj)/cpuflags.o
```

A primeira linha contém esse- `$(obj)/head_$(BITS).o`.

Isso significa que nós iremos selecionar qual arquivo linkar baseado no `$(BITS)` é definido para `head_32.o` ou `head_64.o`. A variável `$(BITS)` é definida em outro lugar no [arch/x86/Makefile](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/Makefile) baseado na configuração de kernel:

```Makefile
ifeq ($(CONFIG_X86_32),y)
        BITS := 32
        ...
        ...
else
        BITS := 64
        ...
        ...
endif
```

Agora que conhecemos onde começar, let's go.

Recarregue os segmentos se necessário
--------------------------------------------------------------------------------

Como indicado acima, começamos no [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S) arquivo código fonte assembly, Primeiro veos a deinição de um atributo de seção especial antes da definição da função `startup_32`:

```assembly
    __HEAD
    .code32
ENTRY(startup_32)
```

`__HEAD` é um macro definido em arquivo header [include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) e expande para a definição da seção seguinte:

```C
#define __HEAD		.section	".head.text","ax"
```

Aqui, `.head.text` é o nome da seção e `ax` é um conjunto de flags. Em nosso caso, essas flags mostra nos que essa seção é [executável](https://en.wikipedia.org/wiki/Executable) ou em outras palavras contém códigos. Nós podemos encontrar a definição desta seção [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/vmlinux.lds.S) script do linker:

```
SECTIONS
{
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
     }
     ...
     ...
     ...
}
```

Se você não está familiar com a sintaxe do script da linguagem do linker `GNU LD`, você pode encontrar mais informações nessa [documentação](https://sourceware.org/binutils/docs/ld/Scripts.html#Scripts). O símbolo `.` é uma variável linker especial, o contador de localização. O valor atribuído a ele é um deslocamento em relação ao segmento. Em nosso caso, nós definimos a contagem de localização para zero. Isso significa que nosso código é linkado para rodar de um deslocamento de `0` na memória. Isso é também afirma no comentários:

```
Seja cuidadoso as parte do head_64.S presuma startup_32 é no endereço 0. 
```

Agora que temos nossa orientações, vamos ver no conteúdo da função `startup_32`.

No começo da função `startup_32`, nós podemos ver a instrução `cld` que limpa o bit `DF` no registro da [flags](https://en.wikipedia.org/wiki/FLAGS_register). Quando a flag de direção é limpada, todos as operações da string como [stos](http://x86.renejeschke.de/html/file_module_x86_id_306.html), [scas](http://x86.renejeschke.de/html/file_module_x86_id_287.html) e outras irão incrementar os registros de índice `esi` ou `edi`. Nós precisamos limpar a direção da fllag, porquê mais tarde usaremos operação string para realizar várias operações como limpar espaços para tabelas de páginas.

Depois limpamos o bit `DF`, o próximo passo para checar a flag `KEEP_SEGMENTS` no campos de cabeçalho de inicialização `loadflags`. Se você lembra, nós já falamos sobre `loadflags` logo na primeira [parte](https://0xax.gitbooks.io/linux-insides/content/Booting/linux-bootstrap-1.html) desde livro. Checamos a flag `CAN_USE_HEAP` para consultar a capacidade de usar o heap. Agora precisamos verificar a flag `KEEP_SEGMENTS`. Essa flag está descrito na documentação do [protocolo Boot] do linux(https://www.kernel.org/doc/Documentation/x86/boot.txt): 

```
Bit 6 (write): KEEP_SEGMENTS
  Protocol: 2.07+
  - If 0, reload the segment registers in the 32bit entry point.
  - If 1, do not reload the segment registers in the 32bit entry point.
    Assume that %cs %ds %ss %es are all set to flat segments with
		a base of 0 (or the equivalent for their environment).
```

Então, se o bit `KEEP_SEGMENTS` não está definido no `loadflags`, precisamos definir nos registros de segmentos `ds`, `ss` e `es` para o índice do segmento de dados com uma base `0`. Vamos fazer:

```C
	testb $KEEP_SEGMENTS, BP_loadflags(%esi)
	jnz 1f

	cli
	movl	$(__BOOT_DS), %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
```

Lembre que `__BOOT_DS` é `0x18` (o índice do segmentos de dados no [Global Descriptor Table]https://en.wikipedia.org/wiki/Global_Descriptor_Table)). Se `KEEP_SEGMENTS` é definido, nós pulamos para o próximo `1f` ou registros de segmento atualizado com `__BOOT_DS` se eles não são definidos. Isso tudo é bem fácil, mas alguma coisa aqui para considerar. Se você lê a [parte anterior](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md), você pode lembrar que nós já atualizamos esses registros de segmentos logo depois de mudarmos para [modo protegido](https://en.wikipedia.org/wiki/Protected_mode) no [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S). Então, por que nós necessitamos cuidar sobre os valores nos registros de segmento de novo? A resposta é fácil. O kernel Linux também tem protocolo boot de 32-bit e se um bootloader usar *isso* para carregar o kernel Linux, todos os códigos antes da função `startup_32` será perdida. Neste caso, o `startup_32` seria a primeiro ponto de entrada para kernel Linux depois do bootloader e nenhuma garantia que os registros de segmentos será estados desconhecidos.

Depois de ter verificado a flag `KEEP_SEGMENTS` e definido o segmento de registros para um valor correto, o próximo passo é para calcular  a diferença entre onde o kernel é compilado para executar, e onde carregamos isso. Lembre que `setup.ld.S` contém o seguintes definições: `. = 0` no começo da seção `.head.text`. Isso significa que o código nesta seção é compilado para executar no endereço `0`. Nós podemos ver essa saída do `objdump`:

```
arch/x86/boot/compressed/vmlinux:     file format elf64-x86-64


Disassembly of section .head.text:

0000000000000000 <startup_32>:
   0:   fc                      cld
   1:   f6 86 11 02 00 00 40    testb  $0x40,0x211(%rsi)
```

O `objdump` util contar nos que o endereço da função `startup_32` é `0`, mas que não é isso. Nós agora precisamos para conhecer onde na verdade são. Isso é muito simples de fazer no [modo longo](https://en.wikipedia.org/wiki/Long_mode) porquê ele suporta o endereço relativo `rip`, mas atualmente nós estamos em [modo protegido](https://en.wikipedia.org/wiki/Protected_mode). 
Nós iremos usar o padrão comum para encontrar o endereço da função `startup_32`. Nós precisamos definir um label, fazer uma chamado para isso e pop o topo da stack para o registro:

```assembly
call label
label: pop %reg
```

Depois diso, o registro indicado pelo `%reg` conterá o endereço do `label`. Olhemos no código que use esse padrão para pesquisar para a função `startup_32` no kernel Linux:

```assembly
        leal	(BP_scratch+4)(%esi), %esp
        call	1f
1:      popl	%ebp
        subl	$1b, %ebp
```

Como você lembra da parte anterior, o registro `eso` contém o endereço da estrutura [boot_params](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h#L113) que foi preenchido do campo antes de nós movemos para o [modo protegido]. A estrutura `boot_params` contém um campo especial `scratch` com um deslocamento do `0x1e4`. Esses 4 campos de byte é uma stack temporária para a instrução `call`. Nós definimos `esp` para o endereço quatro bytes depois o campos `BP_scratch`, porquê como descrito, seriamos uma stack temporária e stack cresce de cima para baixo na arquitetura `x86_64`. Então nosso poneiro de stack apontará para o topo da stack temporária. Próximo, nós podemos ver o padrão que eu descrevi acima. Faremos uma chamada para label `1f` e pop o topo da stack no `ebp`. Esse trabalho porquê `call` armazena o endereço do retorno da função atual no topo da stack. Nós agora temos o endereço do label `1f` e podemos facilmente obter o endereço da função `startup_32`. Nós precisamos subtrair o endereço do label do endereço nós obtivemos na stack:

```
startup_32 (0x0)     +-----------------------+
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
                     |                       |
1f (0x0 + 1f offset) +-----------------------+ %ebp - endereço físico normal
                     |                       |
                     |                       |
                     +-----------------------+
```

A função `startup_32` é lincado para exectar no endereço `0x0` e isso significa que `1f` tem o endereço `0x0 + offset to 1f`, que é aproximadamente bytes `0x21`. O registro `ebp` contém o endereço físico real do label `1f`. Então, se nós subtrair `if` do registro `ebp`, nós obteremos o endereço físico real da função `startup_32`. O [protocolo boot]((https://www.kernel.org/doc/Documentation/x86/boot.txt)) do modo protegido diz a base do kernel em modo protegido é `0x100000`. Nós podemos verificar isso com [gdb](https://en.wikipedia.org/wiki/GNU_Debugger). Vamos começar no debugger e adicionar um breakpoint  no endereço do `1f`, que é `0x100021`. Se isso está correto, nós veremos o valor `0x100021` no registro `ebp`:


```
$ gdb
(gdb)$ target remote :1234
Remote debugging using :1234
0x0000fff0 in ?? ()
(gdb)$ br *0x100022
Breakpoint 1 at 0x100022
(gdb)$ c
Continuing.

Breakpoint 1, 0x00100022 in ?? ()
(gdb)$ i r
eax            0x18	0x18
ecx            0x0	0x0
edx            0x0	0x0
ebx            0x0	0x0
esp            0x144a8	0x144a8
ebp            0x100021	0x100021
esi            0x142c0	0x142c0
edi            0x0	0x0
eip            0x100022	0x100022
eflags         0x46	[ PF ZF ]
cs             0x10	0x10
ss             0x18	0x18
ds             0x18	0x18
es             0x18	0x18
fs             0x18	0x18
gs             0x18	0x18
```

Se nós executamos a instrução seguinte, `subl $1b, %ebp`, veremos:

```
(gdb) nexti
...
...
...
ebp            0x100000	0x100000
...
...
...
```

Ok, nós verificamos que o endereço da função `startup_32` é `0x100000`. Depois nós conhecemos o endereço do label `startup_32`, nós podemos preparar para a transição para o [modo longo](https://en.wikipedia.org/wiki/Long_mode). Nosso próximo objetivo é configurar a stack e verificar que o CPU suporta o modo longo e [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions).

Configurar stack e verifica a CPU
--------------------------------------------------------------------------------

Nós não podemos configurar a stack até nós conhecermos onde na memoria label `startup_32` está. Se imaginarmos a stack como um array, o registro de ponteiro `esp` deve apontar para o final disso. Claro, nós podemos definir um array em nosso código, mas necessitamos conhecer o atual endereço para configurar a ponteiro do stack corretamente. Vamos enxergar no código:


```assembly
	movl	$boot_stack_end, %eax
	addl	%ebp, %eax
	movl	%eax, %esp
```

O label `boot_stack_end` é também definido no arquivo de codigo do assembly [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) e é localizado na seção [.bss](https://en.wikipedia.org/wiki/.bss)

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

Primeiro de tudo, nos colocamos o endereço do `boot_stack_end` no registro `eax`, então o registro `eax` contém o endereço do `boot_stack_end` como foi lincado, qual é `0x0 + boot_stack_end`. Obter o endereço real do `boot_stack_end`, precisamos adicionar o endereço da função `startup_32`. Nós já encontramosesse endereço e colocamos no registro `ebp`. No final, o registro `eax` conterá o endereço real do `boot_stack_end` e precisamos definir o ponteiro da pilha para ele.

Depois que configuramos a stack, o próxima etapa é verifica a CPU. Desde que estamos a transição para o `modo longo`, nós precisamo checar que o CPU suporta `modo longo` e `SSE`. Nós iremos fazer isso com uma chamada para a função `verify_cpu`:

```assembly
	call	verify_cpu
	testl	%eax, %eax
	jnz	no_longmode
```

Essa função é definida no arquivo assembly [arch/x86/kernel/verify_cpu.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/kernel/verify_cpu.S) e contém uma dupla de chamada para a instrução [cpuid](https://en.wikipedia.org/wiki/CPUID). Essa instrução é usado para obter informação sobre o processador. No nosso caso, ele checa se tem suporte para `modo longo` e `SSE` e defini o registro `eax` para 0 se suporta e `1` não suporta.

Se o valor do `eax` não é zero, nós pulamos para o label `no_longmode` que apenas para o CPU com a instrução `hlt` enquanto nenhuma interrupção hardware pode acontecer:

```assembly
no_longmode:
1:
	hlt
	jmp     1b
```

Se o valor do registro `eax` é zero, tudo está ok, então podemos continuar


Calcular endereço de realocação
--------------------------------------------------------------------------------

O próxima etapaé para calcular o endereço de realocação para descompadração se preciso. Primeiro, nós precismo conhecer o que significa para um kernel ser `realocável`. Nós já conhecemos que o endereço base do ponto de entrada 32-bitdo kernel Linux é `0x100000`, mas que é um ponto de entrada 32-bit. O padrão endereço base do kernel do Linux é determinado pelo valor da opção de configuração do kernel `CONFIG_PHYSICAL_START`. Esse valor padrão é `0x1000000` ou `16 MB`. O principal problema aqui é que se o kernel do Linux crashes, um desenvolvedor kernel deve ter um `rescue kernel` para [kdump](https://www.kernel.org/doc/Documentation/kdump/kdump.txt)que é configurado para carregar de um diferente valor. O kernel Linux fornece uma opção de configuração especial para resolver esse problema: `CONFIG_RELOCATABLE`. Como nós podemos ler na documentação do kernel Linux:


```
This builds a kernel image that retains relocation information
so it can be loaded someplace besides the default 1MB.

Note: If CONFIG_RELOCATABLE=y, then the kernel runs from the address
it has been loaded at and the compile time physical address
(CONFIG_PHYSICAL_START) is used as the minimum location.
```

Agora que nós conhecemos onde começar, let's go.

Carregue os segmentos se necessário
--------------------------------------------------------------------------------

Como indicado acima, começamos no código assembly [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/arch/x86/boot/compressed/head_64.S). Nós primeiros vemos a definição do atributo da definição especial antes a definição da função `startup_32`:

```assembly
    __HEAD
    .code32
ENTRY(startup_32)
```

`__HEAD` é um macro definido no arquivo header [include/linux/init.h](https://github.com/torvalds/linux/blob/16f73eb02d7e1765ccab3d2018e0bd98eb93d973/include/linux/init.h) e expande para a definição da seguinte seção:

```C
#define __HEAD		.section	".head.text","ax"
```

Aqui, `.head.text` é o nome da seção e `ax` é um conjunto de flags. Em nosso caso, essas flags mostra nos essa seção é [executável](https://en.wikipedia.org/wiki/Executable). Em termos simples, isso significa que um kernel Linux com essa opção definida pode ser inicializada de diferentes endereços. Tecnicamente, isso é feito compilando o descompressor como [código independente de posição](https://en.wikipedia.org/wiki/Position-independent_code). Se nós vemos no [arch/x86/boot/compressed/Makefile](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/Makefile), nós podemos ver que o descompressor é compilado com a flag `-fPIC`:

```Makefile
KBUILD_CFLAGS += -fno-strict-aliasing -fPIC
```

Quando nós estamos vendo o código independente da posição, um endereço é obtido adicionando o campo de endereço da instrução para o valor do contador de programa. Podemos carregar o código que usa endereçamento de qualquer endereço. Que é porque nós tínhamos que obter o endereço físico do `startup_32`. Agora vamos voltar para o código do kernel Linux. Nosso atual objetivo é calcular um endereço onde nós podemos realocar o kernel para descompressão. O cálculo desse endereço depende da opção de configuração do kernel `CONFIG_RELOCATABLE`, vemos no código:

```assembly
#ifdef CONFIG_RELOCATABLE
	movl	%ebp, %ebx
	movl	BP_kernel_alignment(%esi), %eax
	decl	%eax
	addl	%eax, %ebx
	notl	%eax
	andl	%eax, %ebx
	cmpl	$LOAD_PHYSICAL_ADDR, %ebx
	jge	1f
#endif
	movl	$LOAD_PHYSICAL_ADDR, %ebx
```

Lembra que o valor do registro `ebp` é o endereço físico do label `startup_32`. Se a opção de configuração kernel `CONFIG_RELOCATABLE` é ativado durante a configuraçã do kernel, colocamos esse endereço no registro `ebx`, alinha para um multiplo de `2MiB` e compara com o resultado do macro `LOAD_PHYSICAL_ADDR`. `LOAD_PHYSICAL_ADDR` é definido no header [arch/x86/include/asm/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/boot.h) e vejamos isso:

```C
#define LOAD_PHYSICAL_ADDR ((CONFIG_PHYSICAL_START \
				+ (CONFIG_PHYSICAL_ALIGN - 1)) \
				& ~(CONFIG_PHYSICAL_ALIGN - 1))
```

Como nós podemos ver apenas expande para o valor alinhado `CONFIG_PHYSICAL_ALIGN` que representa o endereço físico onde o kernel vi ser carregado. Depois de comparar `LOAD_PHYSICAL_ADDR` e o valor do registro `ebx`, adicionamos o deslocamento do `startup_32` onde descomprimiremos a imagem do kernel comprimido. Se a opção `CONFIG_RELOCATABLE` não é ativado durante configuração do kernel, nós apenas adicionamos `z_extract_offset` para o endereço padrão onde o kernel é carregado

Depois de todos esses cálculos, `ebp` conterá o endereço onde ncarregamos o kernel e `ebx` conterá o endereço onde o kernel descompactado será realocado. Mas que não está terminado. A imagem do kernel descomprimido deveria ser movido para o final do buffer da descompressão para simplificar cálculos a respeito onde o kernel será localizado mais tarde. Para isso: 

```assembly
1:
    movl	BP_init_size(%esi), %eax
    subl	$_end, %eax
    addl	%eax, %ebx
```

Nós colocamos o valor do campo `boot_params.BP_init_size` (ou o valor do header de inicialização do kernel do `hdr.init_size`) no registro `eax`. O campo `BP_init_size` contém o maior tamanho [vmlinux]((https://en.wikipedia.org/wiki/Vmlinux)) compactados e descompactado.

Preparação antes de entrar no modo longo
--------------------------------------------------------------------------------

Depois adquiriro endereço realocado a imagem do kernel compactado, necessitamos fazer um último etapa antes da transição para modo 64-bit. Primeiro,deve atualizar o [Global Descriptor Table](https://en.wikipedia.org/wiki/Global_Descriptor_Table) com segmento 64-bit porque um kernel relocável é executável em qualquer endereço abaixo 512GB:

```assembly
	addl	%ebp, gdt+2(%ebp)
	lgdt	gdt(%ebp)
```

Aqui nós ajustamos a base do endereço do GDT para o endereço onde na realidade carregamos o kernel e carrega o GDT com a instrução `lgdt`.

Entender a mágica com deslocamento `gdt` nós precisamos olhar na definição do `Global Descriptor Table`. Nós encontramos a definição no mesmo [arquivo](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S) de codigo fonte:

```assembly
	.data
gdt64:
	.word	gdt_end - gdt
	.long	0
	.word	0
	.quad   0
gdt:
	.word	gdt_end - gdt
	.long	gdt
	.word	0
	.quad	0x00cf9a000000ffff	/* __KERNEL32_CS */
	.quad	0x00af9a000000ffff	/* __KERNEL_CS */
	.quad	0x00cf92000000ffff	/* __KERNEL_DS */
	.quad	0x0080890000000000	/* TS descriptor */
	.quad   0x0000000000000000	/* TS continued */
gdt_end:
```

Nós podemos ver que é localizado na seção `.data` e contém 5 descritor: a primeira é um descritor `32-bit` para o segmento de código do kernel, um segmento de código `64-bit`, um segmento de código do kernel e dois descritor de tarefa.

Nós já carregamos o `Global Descriptor Table` na [parte anterior](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md), e agora nós estamos fazendo quase o mesmo aqui, mas nós definimos descritores para usar `CS.L = 1` e `CS.D = 0` para execução no modo `64` bit. Como vemos, a definição do `gdt` começa com dois valor de byte: `gdt_end - gdt` qual representa o endereço do último byte na tabela `gdt` ou o limite da tabela. O próximos 4 bytes contém a base de endereços do `gdt`.

Depois de carregarmos o `Global Descriptor Table` com a instrução `lgdt`, deve habilitar [PAE](http://en.wikipedia.org/wiki/Physical_Address_Extension) colocando o valor do registro `cr4` no `eax`, configurando o 5th bit e carrega-lo de volta em `cr4`:

```assembly
	movl	%cr4, %eax
	orl	$X86_CR4_PAE, %eax
	movl	%eax, %cr4
```

Agora quase finalizado com a preparação necessária para mover no modo 64-bit. O última etapa é para construir tabela de página, mas antes, aqui é algumas informações sobre modo longo.

Modo longo
--------------------------------------------------------------------------------

[modo longo](https://en.wikipedia.org/wiki/Long_mode) é modo nativo para processadores [x86_64](https://en.wikipedia.org/wiki/X86-64). Primeiro, vamos olhar em algumas diferenças entre `x86_64` e `x86`.

modo `64-bit` fornece as seguintes recursos:

* 8 novos registros de propósitos gerais `r8` até `r15` 
* Todos os registros de propósito gerais são 64-bit agora
* Um ponteiro de instrução 64-bit - `RIP`
* Uma novo modo de operação - modo longo
* endereços 64-bit e operando
* Endereços relativos RIP (nós vemos um exemplo disso na parte chegando)

Modo longo é uma extensão do modo protegido legado. Consiste de dois sub modos:

* modo 64-bit
* modo compatível

Alternar para o modo `64-bit` devemos fazer as seguintes coisas:

* [PAE](https://en.wikipedia.org/wiki/Physical_Address_Extension) habilitada;
* Construir tabelas de páginas e carregar o endereço da tabela da página de nível superior no registro `cr3`;
* `EFER.LME` habilitada;
* paginação habilitada;

Nós já habilitamos `PAE` por configurando o bit `PAE` no registro de controle `cr4`. Nosso próximo objetivo é construir a estrutura para [paginação](https://en.wikipedia.org/wiki/Paging). Nós discutimos isso no próximo parágrafo.


Inicialização de tabela de página
--------------------------------------------------------------------------------

Nós já conhecemos queantes podemos mover para modo 64-bit, nós devemos construir tabelas de páginas. Vamos observar como as primeiras tabelas de página de boot `4G` são construídos.

**Nota: Eu não irei descrever a teoria da memória virtual aqui. Se você quer ser mais sobre memória virtual, checa os links no final desta parte.**

O kernel linux usa paginação `4-níveis` e nós geralmente criamos tabelas de 6 páginas:

* Um `PML4` ou tabela `Page Map Level 4` com uma entrada: 
* Um `PDP` ou tabela `Page Directory Pointer` com quatro entradas;
* Quatro tabelas de diretório da página com um otal de entrada `2048`.


Vamos olhar como isso é implementado. Primeiro, nós limpamos o buffer para a tabela de páginas em memória. Toda tabela tem `4096` bytes, então devemos limpar um buffer de `24` kilobyte:

```assembly
	leal	pgtable(%ebx), %edi
	xorl	%eax, %eax
	movl	$(BOOT_INIT_PGT_SIZE/4), %ecx
	rep	stosl
```

Colocamo o endereço do `pgtable` com um deslocamento do `ebx` (lembra que `ebx` aponta para a locação em memória onde o kernel será descompactado depois) no registro `edi`, limpa o registro `eax` e defini o registro `ecx` para `6144`.

A instrução `rep stosl` escreve o valor do `eax` para `edi`, adiciona `4` para `edi` e decrementa `ecx` por `1`. Essa operação será repetida enquanto o valor do registro `ecx` é maior que zero. Que porque nós colocamos `6144` ou `BOOT_INIT_PGT_SIZE/4` no `ecx`.

`pgtable` é definido no final do arquivo assembly [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S):

```assembly
	.section ".pgtable","a",@nobits
	.balign 4096
pgtable:
	.fill BOOT_PGT_SIZE, 1, 0
```

Como podemos ver, está localizado na seção `.pgtable` e o tamanho depende da opção da configuração kernel `CONFIG_X86_VERBOSE_BOOTUP`: 

```C
#  ifdef CONFIG_X86_VERBOSE_BOOTUP
#   define BOOT_PGT_SIZE	(19*4096)
#  else /* !CONFIG_X86_VERBOSE_BOOTUP */
#   define BOOT_PGT_SIZE	(17*4096)
#  endif
# else /* !CONFIG_RANDOMIZE_BASE */
#  define BOOT_PGT_SIZE		BOOT_INIT_PGT_SIZE
# endif
```

Depois temos um buffer para a estrutura `pgtable`, nós podemos começar construir a tabela da página no nível superior `PML4` com:

```assembly
	leal	pgtable + 0(%ebx), %edi
	leal	0x1007 (%edi), %eax
	movl	%eax, 0(%edi)
```

Aqui de novo, colocamos o endereço relativo `pgtable` para `ebx` ou em outro endereço relativo words do `startup_32` no registro `edi`. Próximo, nós colocamos esse endereço com um deslocamento do `0x1007` no registro `eax`. `0x1007` é o resultado do adicionando o tamanho da tabela `PML4` que é `4096` ou `0x1000`bytes com `7`. O `7` aqui representa a flags associada com a entrada `PML4`. Em nosso caso, essas flags são `PRESENT+RW+USER`. No final, nós escrevemos o endereço do primeira entrada `PDP` para a tabela `PML4`.

No próximo passo nós iremos construir 4 entrada `Page Directory` na tabela `Page Directory Pointer` com o mesmas flags`PRESENT+RW+USE`:

```assembly
	leal	pgtable + 0x1000(%ebx), %edi
	leal	0x1007(%edi), %eax
	movl	$4, %ecx
1:  movl	%eax, 0x00(%edi)
	addl	$0x00001000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

Nós definimos `edi` para a endereço base da page directory pointer que é um deslocamento do `4096` ou `0x1000` bytes da tabela `pgtable` e `eax` para o endereço da primeira entrada do page directory pointer. Nós também definimos `ecx` para `4` agir como um ponteiro no seguinte loop e escreve o endereço da primeira entrada da tabela page directory pointer para o registro `edi`. Depois disso, `edi` vai conter o endereço da primeira tabela page directory pointer com flags `0x7`. Em seguida calculamos o endereço da seguinte entrada page directory pointer - cada entrada é `8` bytes - e escreve seus endereços para `eax`. O último passo é construir a estrutura de página para construir tabelas páginas de `2048` com `2MiB`: 

```assembly
	leal	pgtable + 0x2000(%ebx), %edi
	movl	$0x00000183, %eax
	movl	$2048, %ecx
1:  movl	%eax, 0(%edi)
	addl	$0x00200000, %eax
	addl	$8, %edi
	decl	%ecx
	jnz	1b
```

Aqui fazemos quase o mesma coisa que fizemos no exemplo anterior, todas as entradas são associadas com essas flags  - `$0x00000183` - `PRESENT + WRITE + MBZ`. No final, nós iremos ter uma tabela de páginas com páginas `2048` `2MiB`, que representa blocos de memória de 4 GiB:

```python
>>> 2048 * 0x00200000
4294967296
```

Só finalizamos construção da nossa estrutura de tabela inicial que mapeia `4` gibibytes de memória, colocamos o endereço da tabela de páginas no nível superior - `PML4` - no registro de controle `cr3`:

```assembly
	leal	pgtable(%ebx), %eax
	movl	%eax, %cr3
```

Isso é tudo. Nós preparamos para transição para modo longo


A transição para modo 64-bit
--------------------------------------------------------------------------------

Primeiro de tudo nós precisamos definir a flag `EFER.LME` no [MSR](http://en.wikipedia.org/wiki/Model-specific_register) para `0xC0000080`:

```assembly
	movl	$MSR_EFER, %ecx
	rdmsr
	btsl	$_EFER_LME, %eax
	wrmsr
```

Aqui nós colocamos a flag `MSR_EFER` (que é definido em )[arch/x86/include/asm/msr-index.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/asm/msr-index.h)) no registro `ecx` e executa a instrução `rdmsr` que lê o registro [MSR](http://en.wikipedia.org/wiki/Model-specific_register). Depois de executar `rdmsr`, o resultado é guardado em `edx:eax` para o registro `MSR` especificado em `ecx`. Nós verificamos o bit `EFER_LME` com a instrução `btsl` e escreve dados do `edx:eax` voltar para o registro `MSR` com a instrução.

Na próxima etapa, nós empurramos o endereço do código segmento do kernel para a stack (nós definimos no GDT) e coloca o endereço da rotina `startup_64` no `eax`.

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
```

Depois disso empurramos `eax` na stack e habilitamos paginação definido os bits `PG` e `PE` no registro `cr0`: 

```assembly
	pushl	%eax
    movl	$(X86_CR0_PG | X86_CR0_PE), %eax
	movl	%eax, %cr0
```

Em seguida executamos a instrução `lret`:

```assembly
lret
```

Lembre que nós empurramos o endereço da função `startup_64` para a stack na etapa anterior. O CPU extraí o endereço do `startup_64` da stack e pula.

Depois de todos essas stapas nós estamos finalmente no modo 64-bit:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
....
....
....
```

Isso é tudo!


Conclusão
--------------------------------------------------------------------------------

Este é o fm da quarta parte do processo de booting kernel Linux. Se você ter qualquer questão ou sugestão, me ping twitter

This is the end of the fourth part of the linux kernel booting process. If you have any questions or suggestions, ping me on twitter [h4child](https://twitter.com/h4child), mande-me um [email](h4child@protonmail.ch) ou crie uma [issue](https://github.com/h4child/linux-insides-pt/issues/new).

Nesta próxima parte, nós iremos aprender sobre muitas coisas, incluindo a descompressão do trabalho do kernel.

Links
--------------------------------------------------------------------------------

* [Protected mode](http://en.wikipedia.org/wiki/Protected_mode)
* [Intel® 64 and IA-32 Architectures Software Developer’s Manual 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html)
* [GNU linker](http://www.eecs.umich.edu/courses/eecs373/readings/Linker.pdf)
* [SSE](http://en.wikipedia.org/wiki/Streaming_SIMD_Extensions)
* [Paging](http://en.wikipedia.org/wiki/Paging)
* [Model specific register](http://en.wikipedia.org/wiki/Model-specific_register)
* [.fill instruction](http://www.chemie.fu-berlin.de/chemnet/use/info/gas/gas_7.html)
* [Previous part](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-3.md)
* [Paging on osdev.org](http://wiki.osdev.org/Paging)
* [Paging Systems](https://www.cs.rutgers.edu/~pxk/416/notes/09a-paging.html)
* [x86 Paging Tutorial](http://www.cirosantilli.com/x86-paging/)
