Processo booting no kernel. Part 1.
================================================================================

Do bootloader ao kernel
--------------------------------------------------------------------------------

Se você leu as minhas [postagens anteriores](https://0xax.github.io/categories/assembler/), você pode ter notado que eu estive envolvido com programação de baixo nível por algum tempo. Eu escrevi algumas postagem sobre programando assembly para Linux `x86_64` e no mesmo tempo, comecei a mergulhar no código fonte do kernel Linux.

Eu tenho um grande interesse em entender como as coisas funcionam em baixo nível, como os programas executam no computador, como são localizados na memória, como o kernel administra os processos e memória, como a pilha de rede funcionam em baixo nível e muitas outras coisas. Então, eu decidi escrever uma série de postagens sobre kernel Linux para arquitetura **x86_64**.

Note que e não sou um profisional kernel hacker e eu não escrevo código para o kernel no trabalho. É apenas um hobby. Eu apenas gosto das coisas em baixo nível e é interessante para eu ver como essas coisas funcionam. Então se você perceber algo confuso ou se você ter qualquer perguntas/observações, me avise no twitter [autor 0xAX](https://twitter.com/0xAX) [tradutor rodgger](https://twitter.com/rodgger1), 

Se você encontrar alguma coisa errada com meu português ou sobre o conteúdo, sinta-se livre para enviar um pull request.

*Note que esse não é uma documentção oficial, apenas aprendendo e compartilhanddo conhecimento.*

**Conhecimento requeridos**

* Entender código C
* Entender código assembly (AT&T syntax)

De qualquer forma, se você esta apenas começando aprender ferramentas, eu tentarei explicar algumas partes durante este e as sequintes postagens. Bem, esse é o final da introdução simples. Deixe começar mergulhar no kernel Linux e coisas de baixo nível!

Eu comecei a escrever estas postagens na versão `3.18` do kernel Linux, e muitas coisas mudaram deste aquele tempo. Se houver mudanças, eu atualizarei a postagens adequadamente.


O Mágico Botão Power, o que acontece depois?
--------------------------------------------------------------------------------

Embora isso é uma série de postagens sobre kernel linux, não comearemos diretamente no código do kernel. Assim que pressionou o botão mágico power no laptop ou computador desktop, começa funcionar. A placa-mãe envia um sinal para [power suply](https://en.wikipedia.org/wiki/Power_supply)[(similar pt)](https://pt.wikipedia.org/wiki/Fonte_de_alimentação) e tenta iniciar a CPU.A CPU reseta todos os dados no registro e valor predefinido para cada um deles

O [80386](https://en.wikipedia.org/wiki/Intel_80386)[(similar pt)](https://pt.wikipedia.org/wiki/Intel_80386) e os CPUs porteriores definem os seguintes dados predefinidos nos registros da CPU após resetar.

```
IP          0xFFF0
CS selector 0xF000
CS base     0xFFFF0000
```
O processador começa funcionando em [modo real](https://en.wikipedia.org/wiki/Real_mode)[(similar pt)](https://pt.wikipedia.org/wiki/Modo_real). Irei voltar um pouco e tentar entender [segmentação da memória](https://en.wikipedia.org/wiki/Memory_segmentation)[(similar pt)](https://pt.wikipedia.org/wiki/Segmentação_%28memória%29) neste modo. No Modo real é suportado em todos os processadores compatível x86, do CPU [8086](https://en.wikipedia.org/wiki/Intel_8086)[(similar pt)](https://pt.wikipedia.org/wiki/Intel_8086) até as mais modernas como CPU 64-bit da Intel. O processador `8086` tem 20-bit de endereço bus, qual significa que deveria funcionar com um espaço de endereço `0-0xFFFFF` ou `1 mebibyte`. Mas apenas tem registros `16-bit`, qual tem um endereço máximo de `2^16 - 1` ou `0xFFFF` (64 kibibyte).

A [Segmentação de memória](https://en.wikipedia.org/wiki/Memory_segmentation)[(similar pt)](https://pt.wikipedia.org/wiki/Segmentação_%28memória%29) é usado para utilizar todo o espaço de endereço disponível. Toda memória é dividida em paquenos segmentos de tamanhos fixo de `65536` bytes (64KiB). Como não podemos acessar a memória acima de `64KiB`com registros de 16-bit, um método alternativa foi inventada.

Um endereço consiste em duas partes: um seletor de segmento , qual tem uma base de endereço e um offset (deslocamento) da base de endereço. No modo real, o endereço da base asociada de um seletor de segmento é `seletor de segmento * 16`. Então, para obter um endereço físico na memória, nós precisamos multiplicar o seletor do segmento por `16` e adcionar o deslocamento  para isso:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

Por exemplo, se `CS:IP` é `0x2000:0x0010`, então o endereço físico correspondente será:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

Mas, pegarmos o maior seletor de segmentos e offset, `0xFFFF:0xFFFF`, então o endereço será:


```python
>>> hex((0xFFFF << 4) + 0xFFFF)
'0x10ffef'
```

O qual é `65520` bytes a mais que 1 mebibyte `(0x10ffef = 1 mebibyte + 65520)`. Como apenas um mebibyte é acessível em modo real, `0x10FFEF` torna-se `0x00FFEF` com a [linha A20(A20 line)](https://en.wikipedia.org/wiki/A20_line) desativada.

Ok, agora nós conhecemos um pouco sobre modo real e endereçamento de memória. Deixe voltar para a conversa dos valores do registradores depois do resetar.

O registro `CS` consiste de duas partes: o seletor de segmento visível e o endereço base oculto. Enquanto o endereço base é normalmente formato multiplicando o valor do seletor de segmento por 16, durante o reset no hardware o selector de segmento no registro CS é carregado com `0xF000` e o endereço base é carregado com `0xFFFF0000`. O processador usa essa base de endereço especial até  `CS` mudar.

O endereço inicial é formato adicionando o endereço base ao valor no registro IP:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

Obtemos `0xFFFFFFF0`, que é 16 bytes abaixo dos 4GiB. Esse ponteiro é chamado de [redefinir vetor(reset vector)](https://en.wikipedia.org/wiki/Reset_vector). É a localização de memória no qual a CPT esperar encontrar o primeira instrução para executar depois de resetar. Contém uma instrução [jump](https://en.wikipedia.org/wiki/JMP_%28x86_instruction%29)[(similar pt)](http://marco.uminho.pt/~joao/Computacao2/node47.html) (`jmp`) que usualmente aponta para a entrada da [Bios](https://en.wikipedia.org/wiki/BIOS)[(similar pt)](https://pt.wikipedia.org/wiki/BIOS)(Basic Input/Output System). Por exemplo, se olharmos no código fonte(`src/cpu/x86/16bit/reset16.inc` [github](https://github.com/coreboot/coreboot/blob/master/src/cpu/x86/16bit/reset16.inc)) [coreboot](https://www.coreboot.org/), veremos:

```assembly
    .section ".reset", "ax", %progbits
    .code16
.globl	_start
_start:
    .byte  0xe9
    .int   _start16bit - ( . + 2 )
    ...
```

Aqui podemos ver [opcode](http://ref.x86asm.net/coder32.html#xE9) da instrução `jmp`, qual é `0xe9` e endereço de destino em `_start16bit - ( . + 2)`.

Podemos ver que a seção `reset (resetar)` é 16 bytes e é compilado para começar do endereço `0xfffffff0` (`src/cpu/x86/16bit/reset16.ld` [github](https://github.com/coreboot/coreboot/blob/master/src/cpu/x86/16bit/reset16.ld))):


```
SECTIONS {
	/* Trigger an error if I have an unuseable start address */
	_TOO_LOW = CONFIG_X86_RESET_VECTOR - 0xfff0;
	_bogus = ASSERT(_start16bit >= _TOO_LOW, "_start16bit too low. Please report.");

	. = CONFIG_X86_RESET_VECTOR;
	.reset . : {
		*(.reset);
		. = 15;
		BYTE(0x00);
	}
}
```

Agora inicia a BIOS. Após inicializar e checar o hardware, a BIOS precisa encontrar um dispositivo iniciável. A ordem do boot é armazenado na configuração da BIOS, controlando a partir de qual dispositivo a BIOS usar. Iniciar de um hard disk, a BIOS tenta encontrar um setor boot. Na partição hard disk com um [layout da partição MBR](https://en.wikipedia.org/wiki/Master_boot_record)[(similar pt)](https://pt.wikipedia.org/wiki/Master_Boot_Record), o setor de boot é armazenado no primeiro `446` bytes do primeiro setor, onde cada setor é `512` bytes. O dois bytes finais do primeiro setor são `0x55` e `0xaa`, que indica a BIOS que o dispositivo é iniciável.

Por exemplo:

```assembly
;
; Note: this example is written in Intel Assembly syntax
;
[BITS 16]

boot:
    mov al, '!'
    mov ah, 0x0e
    mov bh, 0x00
    mov bl, 0x07

    int 0x10
    jmp $

times 510-($-$$) db 0

db 0x55
db 0xaa
```

Construir e executar isso com:

```
nasm -f bin boot.nasm && qemu-system-x86_64 boot
```

Isso instruirá o [QEMU](https://www.qemu.org/) usar o binário `boot` que nós contruimos a imagem de disco. Como o binário gerado pelo código do assembly acima cumpre o requerimento do setor boot (a origem é definido para `0x7c00` e nós finalizamos com a sequencia mágica), QEMU vai atuar binário como o Master Boot Record (MBR) de uma imagem de disco. 

Você irá ver:

![Simples Bootloader que mostra apenas `!`](images/simple_bootloader.png)

Neste exemplo, nós podemos ver que o código vai ser executado em modo real `16-bit` e começar no na memória `0x7c00`. Após começar, chamar a interrupção [0x10](http://www.ctyme.com/intr/rb-0106.htm), que mostra o símbolo `!`. Preenche o resto `510` bytes com zeros e finaliza com os dois bytes `0xaa` e `0x55`.

Você pode ver o dump(despejo) do binário disso usando o `objdump`:

```
nasm -f bin boot.nasm
objdump -D -b binary -mi386 -Maddr16,data16,intel boot
```

O setor inicial, em um mundo real, tem código para continuar o processo de boot e uma tabela de partição em vez de um monte de 0's e um ponto de exclamação. :) Deste passo em diante, a BIOS passa a controlar sobre o bootloader.

**NOTE**: Como explicado acima, o CPU esta em modo real. Em modo real, calcular o endereço físico na memória é feita assim:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

Apenas explicado acima. Temos apenas um registro geral de 16-bit, que tem um valor máximo do `0xFFFF`, então se pegarmos os maiores valores o resultado será:

```python
>>> hex((0xffff * 16) + 0xffff)
'0x10ffef'
```

Onde `0x10ffef` é o maior endereço `0 até 0x10ffef`, com total `1MiB + 64KiB - 16B`. O processador [8086](https://en.wikipedia.org/wiki/Intel_8086)[(similar pt)](https://pt.wikipedia.org/wiki/Intel_8086)  (qual foi o primeiro processador com modo real), em contraste, tem 20-bit de linha de endereço. Como `2^20 = 1048576 (0 até 2^20-1)` é 1MiB, isso significa que a memória disponivel atual é 1MiB. 

Em geral, memória em modo real é mapeada como seguinte:

```
0x00000000 - 0x000003FF - Real Mode Interrupt Vector Table (Tabelas de vetores de interrupções)
0x00000400 - 0x000004FF - BIOS Data Area (Área de dados da BIOS)
0x00000500 - 0x00007BFF - Unused (não utilizado)
0x00007C00 - 0x00007DFF - Our Bootloader (nosso bootloader)
0x00007E00 - 0x0009FFFF - Unused (não utilizado)
0x000A0000 - 0x000BFFFF - Video RAM (VRAM) Memory (memória RAM de vídeo)
0x000B0000 - 0x000B7777 - Monochrome Video Memory (mémória de vídeo monocromática)
0x000B8000 - 0x000BFFFF - Color Video Memory (memória de vídeo em cores)
0x000C0000 - 0x000C7FFF - Video ROM BIOS (ROM de vídeo da BIOS)
0x000C8000 - 0x000EFFFF - BIOS Shadow Area (Área de sombra da BIOS)
0x000F0000 - 0x000FFFFF - System BIOS (sistema da BIOS)
```

No começo deste post, eu escrevi que a primeira instrução executada pelo CPU é localizada no endereço `0xFFFFFFF0`, qual é muito maior que `0xFFFFF` (1MB). Como a CPU pode acessar esse endereço no modo real? A resposta esta na documentação do [coreboot](https://www.coreboot.org/Developer_Manual/Memory_map):

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

No início da execução, a BIOS não esta na RAM, mas em ROM.


Bootloader (carregador de inicialização)
--------------------------------------------------------------------------------

Existe um número de bootloader que podem inicializar o Linux, tal como [GRUB 2](https://www.gnu.org/software/grub/) e [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). O kernel Linux tem um [protocolo de boot](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt) que especifica os requerimentos para um bootloader implementar suporte ao Linux. Esse exemplo descreve GRUB 2.

Agora que a BIOS tem que escolher um dispositivo para inicializar e transferir o controle para o setor do boot, começa execução do [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD). Código muito simples, devido a limitação de espaço disponível. Contém um ponteiro que é usado para pular para a localização da imagem principal do GRUB 2. A imagem começa com [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), que é geralmente armazenado logo depois do primeiro setor no espaço não utilizado (unused), antes da primeira partição. O código acima carrega o resto da imagem principal, que contém o kernel e os drivers do GRUB 2 para lidar com sistemas de arquivos (filesystems), na memória. Depois carrega o resto da imagem, executa a função [grub_main].

A função `grub_main` inicializa o console, obtém o endereço base para módulos, defini o dispositivo principal, carrega/analisa o arquivos de configuração grub e etc. No final da execução, a função `grub_main` move o grub para o modo normal. A função `grub_normal_execute` (do `grub-core/normal/main.c` no [código fonte](https://github.com/rhboot/grub2/blob/master/grub-core/normal/main.c)) completa a preparação final e mostra um menu para selecionar o sistema operacional. Quando selecionamos a entrada do menu grub, executa a função `grub_menu_execute_entry`, executando o comando grub `boot` e inicializando o sistema operacional selecionado.

Como nós lemos no protocolo boot do kernel, o bootloader deve ler e preencher alguns campos do cabeçalho de configuração do kernel, que começa no ofsset (deslocamento) `0x01f1` do código de configuração do Kernel. Você pode olhar no [linker do script](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) do boot para confirmar o valor do offset. O cabeçalho do kernel começa em [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S):

```assembly
    .globl hdr
hdr:
    setup_sects: .byte 0
    root_flags:  .word ROOT_RDONLY
    syssize:     .long 0
    ram_size:    .word 0
    vid_mode:    .word SVGA_MODE
    root_dev:    .word 0
    boot_flag:   .word 0xAA55
```

O bootloader deve preencher e o resto do cabeçalho (que são apenas marcados como sendo tipo `write` (escrever) em protocolo boot, tal como neste [exemplo](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L354))) com valores recebidos da linha de comando ou calculado durante a inicialização. (Nós não iremos descrever ou explicações para todos os campos do cabeçalho de configuração do kernel por agora, mas devmos fazer quando discutiremos como o kernel usa eles. Você pode encontrar uma descrição de todos os campos no [protocolo boot](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156).)

Como nós podemos ver no protocolo boot do kernel, a memória será mapeada da seguinte maneira após o carregamento do kernel:

```shell
         | Protected-mode kernel  |
100000   +------------------------+
         | I/O memory hole        |
0A0000   +------------------------+
         | Reserved for BIOS      | Deixe o máximo sem uso
         ~                        ~
         | Command line           | (Pode estar abaixo da marca x+10000)
X+10000  +------------------------+
         | Stack/heap             | uso do código em modo real do kernel
X+08000  +------------------------+
         | Kernel setup           | código do modo real do kernel.
         | Kernel boot sector     | O serto boot do kernel legado.
       X +------------------------+
         | Boot loader            | <- ponto de entrada do setor boot 0x7C00
001000   +------------------------+
         | Reserved for MBR/BIOS  |
000800   +------------------------+
         | Typically used by MBR  |
000600   +------------------------+
         | BIOS use only          |
000000   +------------------------+

```

##traduzido até aqui

When the bootloader transfers control to the kernel, it starts at:

```
X + sizeof(KernelBootSector) + 1
```

where `X` is the address of the kernel boot sector being loaded. In my case, `X` is `0x10000`, as we can see in a memory dump:

![kernel first address](images/kernel_first_address.png)

The bootloader has now loaded the Linux kernel into memory, filled the header fields, and then jumped to the corresponding memory address. We now move directly to the kernel setup code.

The Beginning of the Kernel Setup Stage
--------------------------------------------------------------------------------

Finally, we are in the kernel! Technically, the kernel hasn't run yet. First, the kernel setup part must configure stuff such as the decompressor and some memory management related things, to name a few. After all these things are done, the kernel setup part will decompress the actual kernel and jump to it. Execution of the setup part starts from [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) at the [_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L292) symbol.

It may look a bit strange at first sight, as there are several instructions before it. A long time ago, the Linux kernel had its own bootloader. Now, however, if you run, for example,

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

then you will see:

![Try vmlinuz in qemu](images/try_vmlinuz_in_qemu.png)

Actually, the file `header.S` starts with the magic number [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (see image above), the error message that displays and, following that, the [PE](https://en.wikipedia.org/wiki/Portable_Executable) header:

```assembly
#ifdef CONFIG_EFI_STUB
# "MZ", MS-DOS header
.byte 0x4d
.byte 0x5a
#endif
...
...
...
pe_header:
    .ascii "PE"
    .word 0
```

It needs this to load an operating system with [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface) support. We won't be looking into its inner workings right now but will cover it in upcoming chapters.

The actual kernel setup entry point is:

```assembly
// header.S line 292
.globl _start
_start:
```

The bootloader (GRUB 2 and others) knows about this point (at an offset of `0x200` from `MZ`) and jumps directly to it, despite the fact that `header.S` starts from the `.bstext` section, which prints an error message:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

The kernel setup entry point is:

```assembly
    .globl _start
_start:
    .byte  0xeb
    .byte  start_of_setup-1f
1:
    //
    // rest of the header
    //
```

Here we can see a `jmp` instruction opcode (`0xeb`) that jumps to the `start_of_setup-1f` point. In `Nf` notation, `2f`, for example, refers to the local label `2:`. In our case, it's label `1:` that is present right after the jump, and contains the rest of the setup [header](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156). Right after the setup header, we see the `.entrytext` section, which starts at the `start_of_setup` label.

This is the first code that actually runs (aside from the previous jump instructions, of course). After the kernel setup part receives control from the bootloader, the first `jmp` instruction is located at the `0x200` offset from the start of the kernel real mode, i.e., after the first 512 bytes. This can be seen in both the Linux kernel boot protocol and the GRUB 2 source code:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

In my case, the kernel is loaded at the physical address `0x10000`. This means that segment registers have the following values after kernel setup starts:

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

After the jump to `start_of_setup`, the kernel needs to do the following:

* Make sure that all segment register values are equal
* Set up a correct stack, if needed
* Set up [bss](https://en.wikipedia.org/wiki/.bss)
* Jump to the C code in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)

Let's look at the implementation.

Aligning the Segment Registers 
--------------------------------------------------------------------------------

First of all, the kernel ensures that the `ds` and `es` segment registers point to the same address. Next, it clears the direction flag using the `cld` instruction:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

As I wrote earlier, `grub2` loads kernel setup code at address `0x10000` by default and `cs` at `0x1020` because execution doesn't start from the start of the file, but from the jump here:

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

which is at a `512` byte offset from [4d 5a](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L46). We also need to align `cs` from `0x1020` to `0x1000`, as well as all other segment registers. After that, we set up the stack:

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

which pushes the value of `ds` to the stack, followed by the address of the [6](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L602) label and executes the `lretw` instruction. When the `lretw` instruction is called, it loads the address of label `6` into the [instruction pointer](https://en.wikipedia.org/wiki/Program_counter) register and loads `cs` with the value of `ds`. Afterward, `ds` and `cs` will have the same values.

Stack Setup
--------------------------------------------------------------------------------

Almost all of the setup code is for preparing the C language environment in real mode. The next [step](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L575) is checking the `ss` register's value and setting up a correct stack if `ss` is wrong:

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

This can lead to 3 different scenarios:

* `ss` has a valid value `0x1000` (as do all the other segment registers besides `cs`)
* `ss` is invalid and the `CAN_USE_HEAP` flag is set     (see below)
* `ss` is invalid and the `CAN_USE_HEAP` flag is not set (see below)

Let's look at all three of these scenarios in turn:

* `ss` has a correct address (`0x1000`). In this case, we go to label [2](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L589):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Here we set the alignment of `dx` (which contains the value of `sp` as given by the bootloader) to `4` bytes and check if it is zero. If it is, we set `dx` to `0xfffc` (The last 4-byte aligned address in a 64KB segment). If it is not zero, we continue to use the value of `sp` given by the bootloader (`0xf7f4` in my case). Afterwards, we put the value of `ax` (`0x1000`) into `ss`. We now have a correct stack:

![stack](images/stack1.png)

* The second scenario, (`ss` != `ds`). First, we put the value of [_end](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) (the address of the end of the setup code) into `dx` and check the `loadflags` header field using the `testb` instruction to see whether we can use the heap. [loadflags](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) is a bitmask header defined as:

```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

and as we can read in the boot protocol:

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

If the `CAN_USE_HEAP` bit is set, we put `heap_end_ptr` into `dx` (which points to `_end`) and add `STACK_SIZE` (the minimum stack size, `1024` bytes) to it. After this, if `dx` is not carried (it will not be carried, `dx = _end + 1024`), jump to label `2` (as in the previous case) and make a correct stack.

![stack](images/stack2.png)

* When `CAN_USE_HEAP` is not set, we just use a minimal stack from `_end` to `_end + STACK_SIZE`:

![minimal stack](images/minimal_stack.png)

BSS Setup
--------------------------------------------------------------------------------

The last two steps that need to happen before we can jump to the main C code are setting up the [BSS](https://en.wikipedia.org/wiki/.bss) area and checking the "magic" signature. First, signature checking:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

This simply compares the [setup_sig](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) with the magic number `0x5a5aaa55`. If they are not equal, a fatal error is reported.

If the magic number matches, knowing we have a set of correct segment registers and a stack, we only need to set up the BSS section before jumping into the C code.

The BSS section is used to store statically allocated, uninitialized data. Linux carefully ensures this area of memory is first zeroed using the following code:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

First, the [__bss_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) address is moved into `di`. Next, the `_end + 3` address (+3 - aligns to 4 bytes) is moved into `cx`. The `eax` register is cleared (using the `xor` instruction), and the bss section size (`cx - di`) is calculated and put into `cx`. Then, `cx` is divided by four (the size of a 'word'), and the `stosl` instruction is used repeatedly, storing the value of `eax` (zero) into the address pointed to by `di`, automatically increasing `di` by four, repeating until `cx` reaches zero. The net effect of this code is that zeros are written through all words in memory from `__bss_start` to `_end`:

![bss](images/bss.png)

Jump to main
--------------------------------------------------------------------------------

That's all! We have the stack and BSS, so we can jump to the `main()` C function:

```assembly
    calll main
```

The `main()` function is located in [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). You can read about what this does in the next part.

Conclusion
--------------------------------------------------------------------------------

This is the end of the first part about Linux kernel insides. If you have questions or suggestions, ping me on Twitter [0xAX](https://twitter.com/0xAX), drop me an [email](anotherworldofworld@gmail.com), or just create an [issue](https://github.com/0xAX/linux-internals/issues/new). In the next part, we will see the first C code that executes in the Linux kernel setup, the implementation of memory routines such as `memset`, `memcpy`, `earlyprintk`, early console implementation and initialization, and much more.

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

  * [Intel 80386 programmer's reference manual 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Minimal Boot Loader for Intel® Architecture](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [Minimal Boot Loader in Assembler with comments](https://github.com/Stefan20162016/linux-insides-code/blob/master/bootloader.asm)
  * [8086](https://en.wikipedia.org/wiki/Intel_8086)
  * [80386](https://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](https://en.wikipedia.org/wiki/Reset_vector)
  * [Real mode](https://en.wikipedia.org/wiki/Real_mode)
  * [Linux kernel boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [coreboot developer manual](https://www.coreboot.org/Developer_Manual)
  * [Ralf Brown's Interrupt List](http://www.ctyme.com/intr/int.htm)
  * [Power supply](https://en.wikipedia.org/wiki/Power_supply)
  * [Power good signal](https://en.wikipedia.org/wiki/Power_good_signal)
