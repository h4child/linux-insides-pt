Processo do booting do Kernel. Parte 5.
================================================================================

Descompressão do Kernel
--------------------------------------------------------------------------------

Esse é a quinta parte da série `processo booting do Kernel`. Nós ficamos na transição do mdo 64-bit na [parte](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-4-pt.md#transition-to-the-long-mode) anterior e iremos continuar onde paramos. Estudaremos os passos que prepara a descompressão do Kernel, realocação e o processo de descompressão do Kernel em si. Então... entramos no código do Kernel de novo.

Preparando para descompressão do Kernel
--------------------------------------------------------------------------------

Nós ficamos antes de pular para o ponto de entrada do modo `64-bìt` - `startup_64` que é localizado no arquivo fonte [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S). Nós já cobrimos a ida para `startup_64` do `startup_32` na parte anterior:

```assembly
	pushl	$__KERNEL_CS
	leal	startup_64(%ebp), %eax
	...
	...
	...
	pushl	%eax
	...
	...
	...
	lret
```

Deste que nós carregamos um novo `Global Descriptor Table` e o CPU tem transicionado para um modo (`64-bit` modo no noso caso) novo, definimos o registro de segmento de novo no começo da função `startup_64`:

```assembly
	.code64
	.org 0x200
ENTRY(startup_64)
	xorl	%eax, %eax
	movl	%eax, %ds
	movl	%eax, %es
	movl	%eax, %ss
	movl	%eax, %fs
	movl	%eax, %gs
```

Todos os registros de segmento além do registro `cs` são agora redefinidos em `long mode`.

A próxima etapa é computar a diferença entre a localização do Kernel que foi compilado para ser carregado e a localização onde está carregado atualmente:

```assembly
#ifdef CONFIG_RELOCATABLE
	leaq	startup_32(%rip), %rbp
	movl	BP_kernel_alignment(%rsi), %eax
	decl	%eax
	addq	%rax, %rbp
	notq	%rax
	andq	%rax, %rbp
	cmpq	$LOAD_PHYSICAL_ADDR, %rbp
	jge	1f
#endif
	movq	$LOAD_PHYSICAL_ADDR, %rbp
1:
	movl	BP_init_size(%rsi), %ebx
	subl	$_end, %ebx
	addq	%rbp, %rbx
```

O registro `rpb` contém o endereço introdutório do Kernel descompactado. Depois esse código executa, o registro `rbx` que vai conter o endereço onde o código do Kernel vai ser realocado para descompressão. Nós já isso antes na função `startup_32` (você pode ler sobre isso na parte anterior - [Cálculo o endereço de realocação](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-4.md#calculate-relocation-address)), mas precisamos fazer esse cálculo de novo porque o bootloader pode uso o protocolo boot 64-bit agora e o `startup_32` não está sendo executado.

No próximo passo nós definiremos o ponteiro da stack, redefinimos os registros de flags e definimos o `GDT` de novo substituindo com os valores específicos `32-bit` com esses do protocolo `64-bit`:

```assembly
    leaq	boot_stack_end(%rbx), %rsp

    leaq	gdt(%rip), %rax
    movq	%rax, gdt64+2(%rip)
    lgdt	gdt64(%rip)

    pushq	$0
    popfq
```

Se você levar um olhar no código depois da instrução, você verá que existe alguns código adicional. Esse código constrói trampolim para ativar uma [paginação level-5](https://lwn.net/Articles/708526/) se precisar. Nós consideremos paging nível 4 neste livro, então esse código vai ser omitido.

Como pode ver abaixo o registro `rbx` contém o endereço inicial do descompactador do Kernel e apenas colocamos esse endereço com um offset do `boot_stack_end` no registro `rsp` que aponta para o topo da stack. Depois dessa etapa, o stack vai ser corrigido. Você pode encontrar a definição da constante `boot_stack_end` no final do arquivo código fonte em assembly [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S):

```assembly
	.bss
	.balign 4
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
boot_stack:
	.fill BOOT_STACK_SIZE, 1, 0
boot_stack_end:
```

Localizado no final da seção `.bss`, antes `.pgtable`. Se você prestar a atenção dentro do script linker [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/vmlinux.lds.S), você encontrará a definição do `.bss` e `.pgtable` lá.

Desde a stack está correta agora, poderemos copiar o Kernel descomprimido para o endereço que obtivemos acima, quando nós calculamos o endereço de realocação do Kernel descomprimido. Antes pegamos no detalhes, vamos dar uma olhada neste código em assembly:

```assembly
	pushq	%rsi
	leaq	(_bss-8)(%rip), %rsi
	leaq	(_bss-8)(%rbx), %rdi
	movq	$_bss, %rcx
	shrq	$3, %rcx
	std
	rep	movsq
	cld
	popq	%rsi
```

Esse conjunto de instruções copia o Kernel descomprimido sobre onde será descomprimido.

Primeiro de tudo nós push `rsi` para a stack. Nós preservamos o valor do `rsi`, porque esse registro agora armazena o ponteiro para `boot_params` que é uma instruão em modo real que contém o booting de dados relacionado (lembre, essa estrutura foi preenchida no início do setup kernel).

A próximas duas instruções `leaq` calculada o endereço afetivo dos registros `rip` e `rbx` com um offset do `_bss - 8` e atribuí o resultado para `rsi` e `rdi` respectivamente. Por que nós calculamos esses endereços? A imagem Kernel comprimido é carregado entre esse código (do `startup_32` para o código atual) e o código descomprimido. Verifica isso olhando neste script linker - [arch/x86/boot/compressed/vmlinux.lds.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/vmlinux.lds.S):

```
	. = 0;
	.head.text : {
		_head = . ;
		HEAD_TEXT
		_ehead = . ;
	}
	.rodata..compressed : {
		*(.rodata..compressed)
	}
	.text :	{
		_text = .; 	/* Text */
		*(.text)
		*(.text.*)
		_etext = . ;
	}
```

Note que a seção `.head.text` contém `startup_32`. Você pode lembrar na parte anterior:

```assembly
	__HEAD
	.code32
ENTRY(startup_32)
...
...
...
```
A seção `.text` contém o código descompressão:

```assembly
	.text
relocated:
...
...
...
/*
 * Do the decompression, and jump to the new kernel..
 */
...
```

E `.rodata..compressed` contém a imagem do Kernel comprimido. Então o `rsi` conterá o endereço absoluto do `_bss - 8` e `rdi` vai conter a relocação do endereço relativo do `_bss - 8`. No mesmo caminho nós armazenaremos esses endereços no registros, colocamos o endereço do `_bss` no registro `rcx`. Como você pode ver no script linker `vmlinux.lds.S`, está localizado no final de todas as seções com o código setup/kernel. Agora copiamos dados do `rsi` para `rdi`, `8` bytes de cada vez com a instrução `movsq`.

Note que nós executamos uma instrução `std` antes de copiar os dados. Isso defini o flag `DF` que significa que `rsi` e `rdi` será decrementado. Em outras palavras, iremos copiar os bytes para trás. No final, iremos limpar a flag `DF` com a instrução  `cld` e restaurar a estrutura `boot_params` para o `rsi`.

Nos temos um ponteiro para o endereço da  seção `.text` depois da realocação e pulamos nele:

```assembly
	leaq	relocated(%rbx), %rax
	jmp	*%rax
```

Os arranjos finais antes da descompressão do Kernel
--------------------------------------------------------------------------------

No parágrafo anterior nós vimos que a seção `.text` começa com o rótulo `relocated`. A primeira coisa que fazemos é limpar a seção `bss` com:

```assembly
	xorl	%eax, %eax
	leaq    _bss(%rip), %rdi
	leaq    _ebss(%rip), %rcx
	subq	%rdi, %rcx
	shrq	$3, %rcx
	rep	stosq
```

Nós precisamos inicializar a seção `.bss`, porque logos pulamos para o código [C](https://en.wikipedia.org/wiki/C_%28programming_language%29). Aqui nós apenas limpamos `eax`, coloca o endereço do `_bss` em `rdi`, `_ebss` em `rcx` e preenche `bss` com zeros com a instrução `rep stosq`.

```assembly
	pushq	%rsi
	movq	%rsi, %rdi
	leaq	boot_heap(%rip), %rsi
	leaq	input_data(%rip), %rdx
	movl	$z_input_len, %ecx
	movq	%rbp, %r8
	movq	$z_output_len, %r9
	call	extract_kernel
	popq	%rsi
```

Como antes, instrução push `rsi` para a stack preservar o ponteiro `boot_params`. Nós também copiamos o conteúdo do `rsi` para `rdi`. Então, nós definimos `rsi` para apontar para a área onde o kernel vai ser descomprimido. o última etapa é preparar os parâmetros para a função `extract_kernel` e chamar para descomprimir o Kernel. A função `extract_kernel` é definido no arquivo do código fonte [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c) e tem seis argumentos:

* `mode` - um ponteiro para a estrutura [boot_params](https://github.com/torvalds/linux/blob/v4.16/arch/x86/include/uapi/asm/bootparam.h) que é preenchido pelo bootloader ou durante a inicialização do Kernel;
* `heap` - um ponteiro para `boot_heap` que representa o endereço inicial do heap de inicialização inicial;
* `input_data` - um ponteiro para o começo do Kernel descompactado ou ou em outras palavras, um ponteiro para o arquivo `arch/x86/boot/compressed/vmlinux.bin.bz2`;
* `ìnput_len` - o tamanho do kernel descompactado;
* `output` - o endereço de introdução do kernel descompactado;
* `output_len` - o tamanho do Kernel descompactado;

Todos os argumentos serão passado através registros de acordo como o [System V Application Binary Interface](https://github.com/hjl-tools/x86-psABI/wiki/x86-64-psABI-1.0.pdf). Finalizamos toda os preparativos e pode agora descompactar o kernel.

descompactar o kernel
--------------------------------------------------------------------------------

Como vimos no prarágrafo anterior, a função `extract_kernel` é definido no arquivo do código fonte [arch/x86/boot/compressed/misc.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c) e leva 6 argumentos. Essa função começa com a inicialização vídeo/console que nós já vimos na parte anterior. Nós precisamos fazer isso de novo porque não sabemos se começou no [modo real](https://en.wikipedia.org/wiki/Real_mode); se um bootloader foi usado; se o bootloader usado o protocolo boot `32` ou `64-bit`. 

Depois dos primeiro passos da inicialização, nós armazenamos ponteiro para o começo da memória livre e o término dela:

```C
free_mem_ptr     = heap;
free_mem_end_ptr = heap + BOOT_HEAP_SIZE;
```

Here, `heap` is the second parameter of the `extract_kernel` function as passed to it in [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S):

```assembly
leaq	boot_heap(%rip), %rsi
```

Como você viu acima, `boot_heap` é definido como:

```assembly
boot_heap:
	.fill BOOT_HEAP_SIZE, 1, 0
```

Onde `BOOT_HEAP_SIZE` é um macro que expande para `0x10000` (`0x400000` neste caso de um kernel `bzip2`) e representa o tamanho do heap.

Depois inicializamos o ponteiro heap, o próxima etapa é chamar a função `choose_random_location` do arquivo código fonte [arch/x86/boot/compressed/kaslr.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/kaslr.c). Como nós podemos ver no nome da função, escolhe uma localização da memória para escrever o kernel descompactado. Pode parecer estranho que nós precisamos encontrar ou até `choose` (escolher) onde o descompactar a imagem do kernel descompactado, mas o kernel Linux suporta [kASLR](https://en.wikipedia.org/wiki/Address_space_layout_randomization) que permite descompactação do kernel em um endereço aleatório, por razões de segurança.

Nós veremos como o endereço carregado do kernel é randomizado na próxima parte.

Agora voltaremos ao [misc.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/misc.c). Depois de conseguir o endereço para o imagem kernel, checaremos o endereço aleatório está alinhado corretamente e em geral sem erro:

```C
if ((unsigned long)output & (MIN_KERNEL_ALIGN - 1))
	error("Destination physical address inappropriately aligned");

if (virt_addr & (MIN_KERNEL_ALIGN - 1))
	error("Destination virtual address inappropriately aligned");

if (heap > 0x3fffffffffffUL)
	error("Destination address too large");

if (virt_addr + max(output_len, kernel_total_size) > KERNEL_IMAGE_SIZE)
	error("Destination virtual address is beyond the kernel mapping area");

if ((unsigned long)output != LOAD_PHYSICAL_ADDR)
    error("Destination address does not match LOAD_PHYSICAL_ADDR");

if (virt_addr != LOAD_PHYSICAL_ADDR)
	error("Destination virtual address changed when not relocatable");
```
Depois de tudo checado nós veremos uma mensagem familiar:

```
Decompressing Linux...
```

Agora chamamos a função `__decompress` para descompactar o kernel:

```C
__decompress(input_data, input_len, NULL, NULL, output, output_len, NULL, error);
```

A implementação da função `__decompress` depende de qual algorítimo  de descompactação foi escolhido durante a compilação:

```C
#ifdef CONFIG_KERNEL_GZIP
#include "../../../../lib/decompress_inflate.c"
#endif

#ifdef CONFIG_KERNEL_BZIP2
#include "../../../../lib/decompress_bunzip2.c"
#endif

#ifdef CONFIG_KERNEL_LZMA
#include "../../../../lib/decompress_unlzma.c"
#endif

#ifdef CONFIG_KERNEL_XZ
#include "../../../../lib/decompress_unxz.c"
#endif

#ifdef CONFIG_KERNEL_LZO
#include "../../../../lib/decompress_unlzo.c"
#endif

#ifdef CONFIG_KERNEL_LZ4
#include "../../../../lib/decompress_unlz4.c"
#endif
```

Depois o kernel estar descompactado, mais duas função são chamados; `parse_elf` e `handle_relocations`. O principal ponto dessas funções é para mover a imagem do kernel descompactado para um lugar correto na memória. Esse é pois a descompactação é feito [in-place](https://en.wikipedia.org/wiki/In-place_algorithm) e ainda necessita mover o kernel em endereço correto. Já conhecemos, imagem do kernel é um executável [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format). O principal objetivo da função `parse_elf` é mover segmentos carregáveis para o endereço correto. Vemos o segmentos carregáveis do kernel na saída do programa `readelf`:

```
readelf -l vmlinux

Elf file type is EXEC (Executable file)
Entry point 0x1000000
There are 5 program headers, starting at offset 64

Program Headers:
  Type           Offset             VirtAddr           PhysAddr
                 FileSiz            MemSiz              Flags  Align
  LOAD           0x0000000000200000 0xffffffff81000000 0x0000000001000000
                 0x0000000000893000 0x0000000000893000  R E    200000
  LOAD           0x0000000000a93000 0xffffffff81893000 0x0000000001893000
                 0x000000000016d000 0x000000000016d000  RW     200000
  LOAD           0x0000000000c00000 0x0000000000000000 0x0000000001a00000
                 0x00000000000152d8 0x00000000000152d8  RW     200000
  LOAD           0x0000000000c16000 0xffffffff81a16000 0x0000000001a16000
                 0x0000000000138000 0x000000000029b000  RWE    200000
```

O objetivo da função `parse_elf` é carregar esses segmentos para o endereço `output` que obtivemos da função `choose_random_location`. Essa função começa pela checagem da assinatura [ELF](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format):

```C
Elf64_Ehdr ehdr;
Elf64_Phdr *phdrs, *phdr;

memcpy(&ehdr, output, sizeof(ehdr));

if (ehdr.e_ident[EI_MAG0] != ELFMAG0 ||
    ehdr.e_ident[EI_MAG1] != ELFMAG1 ||
    ehdr.e_ident[EI_MAG2] != ELFMAG2 ||
    ehdr.e_ident[EI_MAG3] != ELFMAG3) {
        error("Kernel is not a valid ELF file");
        return;
}
```

Se o cabeçalho ELF não está válido, mostra uma mensagem de erro e desliga. Se nós queremos um arquivo `ELF` válido, temos que ir através de todos os cabeçalho do programa dado pelo arquivo `ELF` e copiar segmentos carregável com endereços alinhados de 2 megabyte corretos para a saída do buffer:

```C
	for (i = 0; i < ehdr.e_phnum; i++) {
		phdr = &phdrs[i];

		switch (phdr->p_type) {
		case PT_LOAD:
#ifdef CONFIG_X86_64
			if ((phdr->p_align % 0x200000) != 0)
				error("Alignment of LOAD segment isn't multiple of 2MB");
#endif
#ifdef CONFIG_RELOCATABLE
			dest = output;
			dest += (phdr->p_paddr - LOAD_PHYSICAL_ADDR);
#else
			dest = (void *)(phdr->p_paddr);
#endif
			memmove(dest, output + phdr->p_offset, phdr->p_filesz);
			break;
		default:
			break;
		}
	}
```

Isto é tudo!

Neste momento todos os segmentos carregável estão e um lugar correto.

A próxima etapa depois da função `parse_elf` é chamar a função `handle_relocations`. A implementação desta função depende da opção de configuração do kernel `CONFIG_X86_NEED_RELOCS` e se está ativado, essa função ajusta endereço na imagem do kernel. Essa função é também chamada apenas se a opção de configuração `CONFIG_RANDOMIZE_BASE` foi ativada durante configuração do kernel. A implementação da função `handle_relocations` é bastante fácil. Essa função subtraí o valor do `LOAD_PHISICAL_ADDR` do valor de endereços carregado básico do kernel e então nós obtivemos a diferença entre onde o kernel foi linkado para carregar e onde foi atualmente carregado. Depois disso nós podemos realocar o kernel, pois sabemos o endereço atual onde o kernel foi carregado, o endereço onde foi linkado para correr e a tabela realocação que está no final da imagem kernel.

Depois o kernel está realocado, nós returnamos da função `extract_kernel` para [arch/x86/boot/compressed/head_64.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/compressed/head_64.S).

O endereço do kernel vai ser no registro `rax` e pulamos nele:

```assembly
jmp	*%rax
```

Isso é tudo. Agora estamos no kernel!

Conclusão
--------------------------------------------------------------------------------

Isso é o final do quinta parte sobre o processo booting do kernel Linux. Não veremos qualquer outro conteúdo sobre o processo de booting do kernel (pode haver as atualizações e conteúdo anteriores), mas haverá muito conteúdo sobre outras partes internas do kernel.

O próximo capítulo vai ser descrito com detalhes mais avançados sobre processo booting kernel do Linux, como randomização de endereços load e etc.

Se você ter qualquer dúvida ou sugestãome escreva um comentário no [e-mail](rodgger@protonmail.com).

**Se você encontrar qualquer erro por favor envia-me PR para [linux-insides-pt](https://github.com/h4child/linux-insides-pt).**

Links
--------------------------------------------------------------------------------

* [address space layout randomization](https://en.wikipedia.org/wiki/Address_space_layout_randomization)
* [initrd](https://en.wikipedia.org/wiki/Initrd)
* [Modo longo](https://en.wikipedia.org/wiki/Long_mode)
* [bzip2](http://www.bzip.org/)
* [Instrução RdRand](https://en.wikipedia.org/wiki/RdRand)
* [Time Stamp Counter](https://en.wikipedia.org/wiki/Time_Stamp_Counter)
* [Programmable Interval Timers](https://en.wikipedia.org/wiki/Intel_8253)
* [parte anterior](https://github.com/0xAX/linux-insides/blob/v4.16/Booting/linux-bootstrap-4-pt.md)
