Processo booting no kernel. Part 1.
================================================================================

Do bootloader ao kernel
--------------------------------------------------------------------------------

Se você leu as minhas [postagens anteriores](https://0xax.github.io/categories/assembler/), você pode ter notado que eu estive envolvido com programação de baixo nível por algum tempo. Eu escrevi algumas postagem sobre programando assembly para Linux `x86_64` e no mesmo tempo, comecei a mergulhar no código fonte do kernel Linux.

Eu tenho um grande interesse em entender como as coisas funcionam em baixo nível, como os programas executam no computador, como são localizados na memória, como o kernel administra os processos e memória, como a stack (pilha) de rede funcionam em baixo nível e muitas outras coisas. Então, eu decidi escrever uma série de postagens sobre kernel Linux para arquitetura **x86_64**.

Note que e não sou um profissional kernel hacker e eu não escrevo código para o kernel no trabalho. É apenas um hobby. Eu apenas gosto das coisas em baixo nível e é interessante para eu ver como essas coisas funcionam. Então se você perceber algo confuso ou se você ter qualquer perguntas/observações, me avise no twitter [autor 0xAX](https://twitter.com/0xAX) [tradutor h4child](https://twitter.com/h4child), envie um [email](mailto:h4child@protonmail.ch) ou apenas crie uma [issue](https://github.com/h4child/linux-insides-pt/issues/new). Eu lhe agradeço.

Se você encontrar alguma coisa errada com meu português ou sobre o conteúdo, sinta-se livre para enviar um pull request.

*Note que esse não é uma documentação oficial, apenas aprendendo e compartilhando conhecimentos.*

**Conhecimento requeridos**

* Entender código C
* Entender código assembly (AT&T syntax)

De qualquer forma, se você esta apenas começando aprender ferramentas, eu tentarei explicar algumas partes durante este e as postagens seguintes. Bem, esse é o final da introdução simples. Deixe começar mergulhar no kernel Linux e coisas de baixo nível!

Eu comecei a escrever estas postagens na versão `3.18` do kernel Linux, e muitas coisas mudaram deste aquele tempo. Se houver mudanças, eu atualizarei a postagens adequadamente.


O Mágico Botão Power(ligar), o que acontece depois?
--------------------------------------------------------------------------------

Embora isso é uma série de postagens sobre kernel linux, não começaremos diretamente no código do kernel. Assim que pressionar o mágico botão power no laptop ou computador desktop, começa funcionar. A placa-mãe recebe um sinal para [power suply](https://en.wikipedia.org/wiki/Power_supply), tenta iniciar a CPU. A CPU reseta todos os dados nos registros e predefini para cada um deles.

O [80386](https://en.wikipedia.org/wiki/Intel_80386) e os CPUs posteriores definem os seguintes dados predefinidos nos registros da CPU após reset.

```
IP          0xFFF0
CS selector 0xF000
CS base     0xFFFF0000
```
O processador começa funcionando em [modo real](https://en.wikipedia.org/wiki/Real_mode). Irei voltar um pouco e tentar entender o [segmento da memória](https://en.wikipedia.org/wiki/Memory_segmentation) neste modo. No Modo real é suportado em todos os processadores compatível x86, do CPU [8086](https://en.wikipedia.org/wiki/Intel_8086) até as mais modernas como CPU 64-bit da Intel. O processador `8086` tem 20-bit de endereço bus, que significa que deveria funcionar com um espaço de endereço `0-0xFFFFF` ou `1 mebibyte`. Mas apenas tem registros `16-bit` que tem um endereço máximo de `2^16 - 1` ou `0xFFFF` (64 kibibyte).

A [Segmentação de memória](https://en.wikipedia.org/wiki/Memory_segmentation)[(similar pt)](https://pt.wikipedia.org/wiki/Segmentação_%28memória%29) é usado para utilizar todo o espaço de endereço disponível. Toda memória é dividida em pequenos segmentos de tamanhos fixo de `65536` bytes (64KiB). Como não podemos acessar a memória acima de `64KiB`com registros de 16-bit, um método alternativo foi inventada.

Um endereço consiste em duas partes: um seletor de segmento, que tem uma base de endereço; e um deslocamento desse endereço base. No modo real, o endereço da base é associada de um seletor de segmento é `seletor de segmento * 16`. Então, para obter um endereço físico na memória, nós precisamos multiplicar o seletor do segmento por `16` e adicionar o deslocamento  para isso:

```
PhysicalAddress = Segment Selector * 16 + Offset
```

Por exemplo, se `CS:IP` é `0x2000:0x0010`, então o endereço físico correspondente será:

```python
>>> hex((0x2000 << 4) + 0x0010)
'0x20010'
```

Mas, se pegarmos o maior seletor de segmentos e offset, `0xFFFF:0xFFFF`, então o endereço será:


```python
>>> hex((0xFFFF << 4) + 0xFFFF)
'0x10ffef'
```

O qual é `65520` bytes a mais que 1 mebibyte `(0x10ffef = 1 mebibyte + 65521)`. Como apenas um mebibyte é acessível em modo real, `0x10FFEF` torna-se `0x00FFEF` com a [linha A20(A20 line)](https://en.wikipedia.org/wiki/A20_line) desativada.

Ok, agora nós conhecemos um pouco sobre modo real e endereçamento de memória. Deixe voltar para a conversa dos valores do registradores depois do resetar.

O registro `CS` consiste de duas partes: o seletor de segmento visível e o endereço base oculto. O endereço base é normalmente formato multiplicando o valor do seletor de segmento por 16, durante o reset no hardware o seletor de segmento no registro CS é carregado com `0xF000` e o endereço base é carregado com `0xFFFF0000`. O processador usa essa base de endereço especial até  `CS` mudar.

O endereço inicial é formato adicionando o endereço base ao valor no registro EIP:

```python
>>> 0xffff0000 + 0xfff0
'0xfffffff0'
```

Obtemos `0xFFFFFFF0`, que é 16 bytes abaixo dos 4GiB. Esse ponteiro é chamado de [reset vector](https://en.wikipedia.org/wiki/Reset_vector). É a localização de memória no qual a CPU esperar encontrar o primeira instrução para executar depois do reset. Contém uma instrução [jump](https://en.wikipedia.org/wiki/JMP_%28x86_instruction%29)[(similar pt)](http://marco.uminho.pt/~joao/Computacao2/node47.html) (`jmp`) que usualmente aponta para o ponto de entrada da [Bios](https://en.wikipedia.org/wiki/BIOS)[(similar pt)](https://pt.wikipedia.org/wiki/BIOS)(Basic Input/Output System). Por exemplo, se olharmos no código fonte(`src/cpu/x86/16bit/reset16.inc` [github](https://github.com/coreboot/coreboot/blob/master/src/cpu/x86/16bit/reset16.inc)) [coreboot](https://www.coreboot.org/), veremos:

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

Agora inicia a BIOS. Após inicializar e checar o hardware, a BIOS deve encontrar um dispositivo iniciável. A ordem do boot é armazenado na configuração da BIOS, controlando a partir de qual dispositivo usar. Iniciar de um hard disk, a BIOS tenta encontrar um setor boot. Na partição hard disk com um [layout da partição MBR](https://en.wikipedia.org/wiki/Master_boot_record)[(similar pt)](https://pt.wikipedia.org/wiki/Master_Boot_Record), o setor de boot é armazenado no primeiro `446` bytes do primeiro setor, onde cada setor é `512` bytes. Os dois bytes finais do primeiro setor são `0x55` e `0xaa`, que indica a BIOS que o dispositivo é iniciável. Logo que a BIOS enconrar o setor boot, copia na localização de memória fixa 0x7c00, pula para esse endereço e então começa executa-lo.

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

Isso instruirá o [QEMU](https://www.qemu.org/) usar o binário `boot` que nós contruimos a imagem de disco. Como o binário gerado pelo código do assembly acima cumpre o requerimento do setor boot (nós concluímos com a sequência mágica), QEMU vai tratar binário como o Master Boot Record (MBR) de uma imagem de disco. Observe que quando geramos uma imagem do binário boot para QEMU, definindo a origem para 0x7c00 (usando `[ORG  0x7c00]`)

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

Onde `0x10ffef` é o maior endereço `0 até 0x10ffef`, com total `(1MB + 64KB - 16B) - 1`. O processador [8086](https://en.wikipedia.org/wiki/Intel_8086)[(similar pt)](https://pt.wikipedia.org/wiki/Intel_8086)  (qual foi o primeiro processador com modo real), em contraste, tem 20-bit de linha de endereço. Como `2^20 = 1048576 (0 até 2^20-1)` é 1MiB, isso significa que a memória disponivel atual é 1MiB. 

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

No começo deste post, eu escrevi que a primeira instrução executada pelo CPU é localizada no endereço `0xFFFFFFF0`, qual é muito maior que `0xFFFFF` (1MiB). Como a CPU pode acessar esse endereço no modo real? A resposta esta na documentação do [coreboot](https://www.coreboot.org/Developer_Manual/Memory_map):

```
0xFFFE_0000 - 0xFFFF_FFFF: 128 kilobyte ROM mapped into address space
```

No início da execução, a BIOS não esta na RAM, mas em ROM.


Bootloader (carregador de inicialização)
--------------------------------------------------------------------------------

Existe um número de bootloader que podem inicializar o Linux, tal como [GRUB 2](https://www.gnu.org/software/grub/) e [syslinux](http://www.syslinux.org/wiki/index.php/The_Syslinux_Project). O kernel Linux tem um [protocolo de boot](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt) que especifica os requerimentos para um bootloader implementar suporte ao Linux. Esse exemplo descreve GRUB 2.

Agora que a BIOS tem que escolher um dispositivo para inicializar e transferir o controle para o setor do boot, começa execução do [boot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/boot.S;hb=HEAD). Código muito simples, devido a limitação de espaço disponível. Contém um ponteiro que é usado para pular para a localização da imagem principal do GRUB 2. A imagem começa com [diskboot.img](http://git.savannah.gnu.org/gitweb/?p=grub.git;a=blob;f=grub-core/boot/i386/pc/diskboot.S;hb=HEAD), que é geralmente armazenado logo depois do primeiro setor no espaço não utilizado (unused), antes da primeira partição. O código acima carrega o resto da imagem principal, que contém o kernel e os drivers do GRUB 2 para lidar com sistemas de arquivos (filesystems), na memória. Depois carrega o resto da imagem, executa a função [grub_main].

A função `grub_main` inicializa o console, obtém o endereço base para módulos, defini o dispositivo principal, carrega/analisa o arquivos de configuração grub, carrega módulos e etc. No final da execução, a função `grub_main` move o grub para o modo normal. A função `grub_normal_execute` (do `grub-core/normal/main.c` no [código fonte](https://github.com/rhboot/grub2/blob/master/grub-core/normal/main.c)) completa a preparação final e mostra um menu para selecionar o sistema operacional. Quando selecionamos a entrada do menu grub, executa a função `grub_menu_execute_entry`, executando o comando grub `boot` e inicializando o sistema operacional selecionado.

Como nós lemos no protocolo boot do kernel, o bootloader deve ler e preencher alguns campos do cabeçalho de inicialização do kernel, que começa no ofset (deslocamento) `0x01f1` do código de inicialização Kernel. Você pode olhar no [linker do script](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) do boot para confirmar o valor do offset. O cabeçalho do kernel começa em [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S):

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

O bootloader deve preencher esse e o resto do cabeçalho (que são apenas marcados como sendo tipo `write` (escrever) em protocolo boot, tal como neste [exemplo](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L354)) com valores recebidos da linha de comando ou calculado durante a inicialização. (Nós não iremos descrever ou explicações para todos os campos do cabeçalho de inicialização do kernel por agora, mas devemos fazer quando discutiremos como o kernel usa eles. Você pode encontrar uma descrição de todos os campos no [protocolo boot](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156).)

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
         | Kernel boot sector     | O setor boot do kernel legado.
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

Quando o boorloader transfere controle para o kernel, começa no:

```
X + sizeof(KernelBootSector) + 1
```

Onde `X` é o endereço do setor boot do kernel sendo carregado. Em meu caso, `X` é `0x0x10000`, como podemos ver podemos ver no dump de memória:

![Primeiro endereço do kernel](images/kernel_first_address.png)

O bootloader tem agora carregado o kernel linux na memória, preenchido os campos de cabeçalho e saltou para o endereço da memória correspondente. Agora movemos para o código de inicialização do kernel linux.

O estágio da inicialização do kernel
--------------------------------------------------------------------------------

Finalmente, nós estamos no kernel! Tecnicamente, o kernel não está executando ainda. Primeiramente, uma parte da inicialização do kernel deve configurar coisas tal como descompactador e algumas coisas relacionadas a gerenciamento de memória. Depois de todas essas coisas foram feitas, uma parte da inicialização do kernel vai descompactar o atual  kernel e pular para ele. Execução da parte da inicialização começa do [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S) no símbolo [_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L292).

Pode aparecer um pouco estranho a primeira vista, como são várias instruções antes disso. Tempos atrás, o kernel linux tinha seu próprio bootloader. Agora, tanto faz, se você executar, por exemplo:

```
qemu-system-x86_64 vmlinuz-3.18-generic
```

Então verá:

![Tentar vmlinuz no qemu](images/try_vmlinuz_in_qemu.png)

Realmente, o arquivo `header.S` começa com o número mágico [MZ](https://en.wikipedia.org/wiki/DOS_MZ_executable) (veja imagem acima), a mensagem de erro que é mostrado e seguindo o header [PE](https://en.wikipedia.org/wiki/Portable_Executable)[(similar pt)](https://pt.wikipedia.org/wiki/Portable_Executable):

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

Precisa para carregar um sistema operacional com suporte a [UEFI](https://en.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface)[(similar pt)](https://pt.wikipedia.org/wiki/Unified_Extensible_Firmware_Interface). Nós não

O ponto de entrada do kernel é:

```assembly
// header.S line 292
.globl _start
_start:
```

O bootloader (GRUB 2 e outros) conhece sobre esse ponto (em um offset do `0x200` do `MZ`) e pula diretamente para ele, apesar do fato que `header.S` começa da seção `.bstext`, o qual mostra uma mensagem de erro:

```
//
// arch/x86/boot/setup.ld
//
. = 0;                    // current position
.bstext : { *(.bstext) }  // put .bstext section to position 0
.bsdata : { *(.bsdata) }
```

O ponto de entrada do kernel é:

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

Aqui nós podemos ver um upcode da intrução `jmp` (`0xeb`) que pula para o ponto `start_of_setup-1f`. Em notação `Nf`, `2f`, por exemplo, refere-se ao local do rótulo `2:`. Nosso caso, é rótulo `1:` que é presente depois de pular e contém o resto do [cabeçalho de inicialização](https://github.com/torvalds/linux/blob/v4.16/Documentation/x86/boot.txt#L156) de inicialização. Logo depois do cabeçalho, nós vemos a seção `.entrytext`, que começa no rótulo `start_of_setup`.

Esse é o primeiro código que realmente executa (além da instrução jump anteriormente, claro). Depois a inicialização do kernel receber o controle do bootloader, a primeira instrução `jmp` é localizada no offset `0x200` do começo do modo real do kernel, ou seja, depois dos primeiros 512 bytes. Esse pode ser visto em ambos o protocolo boot do kernel linux e o código fonte do GRUB 2:

```C
segment = grub_linux_real_target >> 4;
state.gs = state.fs = state.es = state.ds = state.ss = segment;
state.cs = segment + 0x20;
```

Em meu caso, o kernel é carregado no endereço físico `0x10000`. Isso significa que o registro de segmento tem o seguinte valores depois de iniciar o kernel:

```
gs = fs = es = ds = ss = 0x1000
cs = 0x1020
```

Depois o pulo para `start_of_setup`, o kernel precisa fazer o seguinte:

* Verifique se todos os valores de registro de segmento são iguais
* Construir uma stack, se necessário
* Construir [bss](https://en.wikipedia.org/wiki/.bss)
* Pular para o código C em [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c)

Vamos reparar na implementação.

Alinhamento dos registros de segmento
--------------------------------------------------------------------------------

Primeiro de tudo, o kernel gerante que os registradores de segmento `ds` e `es` apontem para o mesmo segmento. Próximo, limpa a flag (bandeira) de direção usando a instrução `cld`:

```assembly
    movw    %ds, %ax
    movw    %ax, %es
    cld
```

Como eu escrevi mais cedo, `grub2` carrega o código de inicialização do kernel no endereço `0x10000` por padrão e `cs` no `0x1020` porquê a execução não começa do start do arquivo, mas do jump aqui:

```assembly
_start:
    .byte 0xeb
    .byte start_of_setup-1f
```

Que esta no offset `512` bytes do [4d 5a](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L46). Nós precisamos alinhar `cs` do `0x1020` para `0x1000`, como todos outros registros de segmento. Depois que, nós definimos a  (stack):

```assembly
    pushw   %ds
    pushw   $6f
    lretw
```

Empurrar ([push](https://pt.wikibooks.org/wiki/Programar_em_Assembly_com_GAS/Instru%C3%A7%C3%B5es#Instru%C3%A7%C3%B5es_para_manipula%C3%A7%C3%A3o_de_stack)) o valor de `ds` para a stack, seguido pelo endereço do rótulo [6](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L602) e executa a instrução `lretw`. Quando a instrução `lretw` é chamado, carrega o endereço do rótulo `6` no registro de [ponteiro de instrução (PI)](https://en.wikipedia.org/wiki/Program_counter) e carrega `cs` como o valor de `ds`. Depois, `ds` e `cs` vai ter os mesmos valores.

inicialização do stack (pilha)
--------------------------------------------------------------------------------

Quase todo código é para inicializar o ambiente (environment) da linguagem C em modo real. O próximo [passo](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L575) esta checando o valor do registro `ss` e configurar uma stack correta se `ss` estiver errado: 

```assembly
    movw    %ss, %dx
    cmpw    %ax, %dx
    movw    %sp, %dx
    je      2f
```

Isso pode levar a 3 cenários diferentes:

* `ss` tem um valor válido `0x1000` (como todos os registros de segmento além do `cs`)
* `ss` é inválido e a bandeira (flag) `CAN_USE_HEAP` é definido (veja abaixo)
* `ss` é inválido e a bandeira (flag) `CAN_USE_HEAP` não é definido (veja abaixo)

Vamos inspecionar todos os três cenário:

* `ss` tem um endereço correta (`0x1000`). Neste caso, nós vamos para rótulo [2](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L589):

```assembly
2:  andw    $~3, %dx
    jnz     3f
    movw    $0xfffc, %dx
3:  movw    %ax, %ss
    movzwl  %dx, %esp
    sti
```

Aqui nós definimos do `dx` (que contém o valor do `sp` fornecido pelo bootloader) para `4` bytes e checa se é zero. Se é, definimos para o endereço de alinhamento `0xfffc` (o último 4-byte alinha) em segmento 64 KiB). Se não é zero, continua usar o valor de `sp` dado pelo bootloader (`0xf7f4` no meu caso). Mais tarde, colocaremos o valor de `ax` (0x1000) no `ss`. Agora temos uma stack correta:

![stack (Pilha)](images/stack1.png)

* O segundo cenário, (`ss` != `ds`). Primeiro, colocamos o valor de [_end](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) (o endereço da finalização do código) em `dx` e checar o campo cabeçalho `loadflags` usando a instrução `testb` ver se nós podemos usar o heap. [loadflags](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S#L320) é um cabeçalho bitmask definido como:


```C
#define LOADED_HIGH     (1<<0)
#define QUIET_FLAG      (1<<5)
#define KEEP_SEGMENTS   (1<<6)
#define CAN_USE_HEAP    (1<<7)
```

e podemos ler o protocolo de boot:

```
Field name: loadflags

  This field is a bitmask.

  Bit 7 (write): CAN_USE_HEAP
    Set this bit to 1 to indicate that the value entered in the
    heap_end_ptr is valid.  If this field is clear, some setup code
    functionality will be disabled.
```

Se o bit `CAN_USE_HEAP` é definido, nós colocamos `heap_end_ptr` em `dx` (que aponta para `_end`) e adiciona `STACK_SIZE` (o mínimo tamanho de stack, `1024 bytes`). Depois disso, se `dx` não é carregado (não é carregado, `dx = _end + 1024`), pula para o rórulo `2` (como no caso anterior) e faz stack.

![stack (pilha)](images/stack2.png)

* Quando `CAN_USE_HEAP` não é definido, nós usamos stack mínimo do `_end` para `_end + STACK_SIZE`:

![stack (pilha) mínimo](images/minimal_stack.png)

Inicialização do BSS
--------------------------------------------------------------------------------

Os dois últimos passos que precisa acontecer antes de pular para a main no código C configurando a área [BSS](https://en.wikipedia.org/wiki/.bss) e checando a assinatura mágica. Primeiro, checando a assinatura:

```assembly
    cmpl    $0x5a5aaa55, setup_sig
    jne     setup_bad
```

Isso simplesmente compara o [setup_sig](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) com o número mágico `0x5a5aaa55`. Se eles não são iguais, um erro fatal é reportado.

Se o número mágico combinar, sabendo definir do registros de segmento correto e uma stack, nós apenas precisamos definir a seção BSS antes de pular no código C.

A seção BSS é usado para armazenar estaticamente, dados não inicializado. Linux cuidadosamente garante essa área da memória primeiro seja zerado usando o seguinte código:

```assembly
    movw    $__bss_start, %di
    movw    $_end+3, %cx
    xorl    %eax, %eax
    subw    %di, %cx
    shrw    $2, %cx
    rep; stosl
```

Primeiramente, o endereço [__bss_start](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/setup.ld) é movido para `di`. E sequida, o endereço `_end + 3` (+3 - alinhar to 4 bytes) é movido para `cx`. O registro `eax` é zerado (usando a instrução xor) e o tamanho da seção bss (`cx - di`) é calculado e colocar no `cx`. Então, `cx` é dividido por 4 (o tamanho de uma 'word') e a instrução `stosl` é usado repetidamente, armazenando o valor de `eax` (zero) no endereço apontado por `di`, automaticamente encrementado `di` por 4, repetidamente até `cx` atingir zero. O efeito deste código é que zeros são escrito através de todas 'word' na memória do `__bss_start` ao `_end`:

![bss](images/bss.png)

Pular para main
--------------------------------------------------------------------------------

Isso é tudo! Nós teremos a pilha e BSS, então pulomos para a função  `main()` da linguagem C:

```assembly
    calll main
```

A função `main()`  é localizado em [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). Você pode ler sobre o que isso faz na próxima parte.

Conclusão
--------------------------------------------------------------------------------

Essa é o final da primeira parte sobre Linux kernel innsides. Se você tem pergunta ou sugestão, me envie no twitter [rodgger1](https://twitter.com/rodgger1), mande um [email](rodggerbruno@gmail.com) ou crie uma [issue](https://github.com/rodggerbr/linux-insides/issues/new). Na próxima parte iremos ver o primeiro o primeiro código em C que executa na inicialização do kernel, a implementação da rotina de memória tal como `memset`, `memcpy`, `earlyprintk` e muito mais.

**Se você encontrou qualquer erro, por favor me envie um PR (pull request) para [traduçã do linux-insides](https://github.com/rodggerbr/linux-insides).**

**Please note that English is not my first language and I am really sorry for any inconvenience. If you find any mistakes please send me PR to [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

  * [Intel 80386 manual referência para programadores 1986](http://css.csail.mit.edu/6.858/2014/readings/i386.pdf)
  * [Carregador boot mínimo para arquitetura Intel®](https://www.cs.cmu.edu/~410/doc/minimal_boot.pdf)
  * [Carregador boot mínimo em assembler com comentário](https://github.com/Stefan20162016/linux-insides-code/blob/master/bootloader.asm)
  * [8086](https://en.wikipedia.org/wiki/Intel_8086)
  * [80386](https://en.wikipedia.org/wiki/Intel_80386)
  * [Reset vector](https://en.wikipedia.org/wiki/Reset_vector)
  * [modo real](https://en.wikipedia.org/wiki/Real_mode)
  * [Protocolo boot do kernel linux](https://www.kernel.org/doc/Documentation/x86/boot.txt)
  * [Manual coreboot para desenvolvedores](https://www.coreboot.org/Developer_Manual)
  * [List de interrupções de Ralf Brown](http://www.ctyme.com/intr/int.htm)
  * [fonte de energia](https://en.wikipedia.org/wiki/Power_supply)
  * [bom sinal de energia](https://en.wikipedia.org/wiki/Power_good_signal)
