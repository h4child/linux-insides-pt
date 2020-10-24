Processo de inicialização do kernel Part 3.
================================================================================

Inicialização do modo vídeo e transição para o modo protegido
--------------------------------------------------------------------------------

Esse é a terceira parte da série `processo booting do kernel`.A [parte anterior](linux-bootstrap-2.md#kernel-booting-process-part-2), nós paramos logo antes a chamada para a rotina `set_video` do [main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c).

Nesta parte, nós veremos:

* inicialização do modo vídeo na inicialização do linux,
* os preparativos feitos antes de mudar para o modo protegido,
* a transição para modo protegido

**NOTA** se você não sabe qualquer coisa sobre modo protegido, você pode encontrar algumas informações sobre [parte anterior](linux-bootstrap-2.md#protected-mode). Também, há outros [links](linux-bootstrap-2.md#links) que pode ajudar você.

Como eu escrevi, começaremos da função `set_video` que está definida no [arch/x86/boot/video.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video.c). Começa obtendo o modo video da estrutura `boot_params.hdr`:

```C
u16 mode = boot_params.hdr.vid_mode;
```

que nós preenchemos na função`copy_boot_params` (lê sobre post anterior). `vid_mode`é um campo obrigatório que é preenchido pelo bootloader. Vocé encontrará informação sobre `boot protocol` do kernel:

```
Offset	Proto	Name		Meaning
/Size
01FA/2	ALL	    vid_mode	Video mode control
```

Como podemos ler do boot protocol:

```
vga=<mode>
	<mode> here is either an integer (in C notation, either
	decimal, octal, or hexadecimal) or one of the strings
	"normal" (meaning 0xFFFF), "ext" (meaning 0xFFFE) or "ask"
	(meaning 0xFFFD).  This value should be entered into the
	vid_mode field, as it is used by the kernel before the command
	line is parsed.
```

Então nós podemos adicionar opção `vga` para a arquivo de configuração do grub (ou outro bootloader) e passaremos essa opção para linha de comando do kernel. Essa opção pode ter diferentes valores como mencionados na descrição. Por examplo, pode ser um número inteiro `0xFFFD` ou `ask`. Se você passar `ask` para `vga`, você verá um menun como esse:

![configuração do menu em modo vídeo](images/video_mode_setup_menu.png)

Que questionará para selecionar um modo vídeo. Nós observaremos a implementação, mas antes dividimos a implementação temos que olhar algumas outras coisas.


Tipo de dados do kernel
--------------------------------------------------------------------------------

Anteriormente nós vimos a definições de diferentes tipo de dados como `u16` etc. Vejamos alguns tipos de dados fornecido pelo kernel:

u = unsigned

| Tipo | char | short | int | long | u8 | u16 | u32 | u64 |
|------|------|-------|-----|------|----|-----|-----|-----|
| Size |  1   |   2   |  4  |   8  |  1 |  2  |  4  |  8  |


Se você ler o código fonte do kernel, verá com muita frequência.


API Heap
--------------------------------------------------------------------------------

Depois obtivemos `vid_mode` do `boot_params.hdr` na função `set_video`, podemos ver a chamada para a função `RESET_HEAP`. `RESET_HEAP` é um macro que é definido no arquivo do cabeçalho [arch/x86/boot/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/boot.h).

Esse macro é definido como:

```C
#define RESET_HEAP() ((void *)( HEAP = _end ))
```
Se você ler a segunda parte, você irá lembrar que inicializamos o heap com a função [`init_heap`](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c). Nós temos macros de utilidades e funções para gerenciar o heap que são definidos no arquivo de cabeçalho `arch/x86/boot/boot.h`.

Eles são:

```C
#define RESET_HEAP()
```
Como nós vimos acima, redefini o heap pelo configurando a variável `HEAP` para `_end`, onde `_end` é apenas `extern char _end[];`

Próximo é o macro `GET_HEAP`:

```C
#define GET_HEAP(type, n) \
	((type *)__get_heap(sizeof(type),__alignof__(type),(n)))
```
para alocação heap. É chamado a função interno `__get_heap` com 3 parâmetro:

* o tamanho do tipo de dados ser alocado para
* especificar `__alignof__(type)` como variável deste tipo são ser alinhado 
* especificar `n` como muito itens alocado.

A implementação do `__get_heap` é:

```C
static inline char *__get_heap(size_t s, size_t a, size_t n)
{
	char *tmp;

	HEAP = (char *)(((size_t)HEAP+(a-1)) & ~(a-1));
	tmp = HEAP;
	HEAP += s*n;
	return tmp;
}
```

e veremos ainda seu uso, algo como:

```C
saved.data = GET_HEAP(u16, saved.x * saved.y);
```
Tentaremos entender como trabalha `__get_heap`. Podemos ver aqui que `HEAP` (qual é igual para `_end` depois `RESET_HEAP()`) é atribuído o endereço da memória alinhado de acordo com o parâmetro `a`. Depois disso salvamos o endereço de memória do `HEAP` para a variável `tmp`, move `HEAP` para o final do bloco alocado e retorne `tmp`, que é o endereço inicial da memória alocada.

E a última função é:

```C
static inline bool heap_free(size_t n)
{
	return (int)(heap_end - HEAP) >= (int)n;
}
```

que subtrai o valor do ponteiro `HEAP` do `heap_end` (nós calculamos na [parte](linux-bootstrap-2.md) anterior) e retorna 1 se houver memória suficiente disponível para `n`.

Que é tudo. Agora nós temos um simples API para heap e podemos definir modo de vídeo.

Configurar o modo vídeo
--------------------------------------------------------------------------------

Podemos mover diretamente para inicialização do modo vídeo. Paramos na chamada `RESET_HEAP()` na função `set_video`. Em seguida é a chamada para `store_mode_params` que armazenar parâmetro no modo vídeo na estrutura `boot_params.screen_info` que é definido no arquivo de cabeçalho [include/uapi/linux/screen_info.h](https://github.com/torvalds/linux/blob/v4.16/include/uapi/linux/screen_info.h).

Se nós olhar na função `store_mode_params`, podemos ver que está começando com uma chamada para  função `store_cursor_position`. Como você pode entender do nome da função, ele obtém informção sobre o cursor e as armazena.

Primeiro de tudo, `store_cursor_position` inicializa duas variáveis que tem o tipo `biosregs` com `AH = 0x3`e chama interrupção da BIOS `0x10`. Depois a interrupção é executado com sucesso retorna linha e colunas nos registros `DL` e `DH`. Linha e coluna serão armazenado nos campos `orig_x` e `orig_y` da estrutura `boot_params.screen_info`.

Depois `store_cursor_position` é executado, a função `store_video_mode` será chamado. Ele apenas adquiri o modo vídeo atual e armazena em `boot_params.screen_info.orig_video_mode`.

Depois disso, `store_mode_params` checa o modo vídeo atual e defini o `video_segment`. Depois a BIOS transfere o controle para o setor do boot, o endereço seguinte são para memória de vídeo:

```
0xB000:0x0000 	32 Kb 	Memória de vídeo de texto monocromático
0xB800:0x0000 	32 Kb 	Memória de vídeo de texto com cor
```
Enão definimos a variável `video_segment` para `0xb000` se o modo de vídeo atual é MDA, HGC ou VGA modo monocromático e para `0xb800` se o modo vídeo atual está em modo colorido. Depois configuramos o endereço do segmento de vídeo, o tamanho da fonte precisa ser armazenado em `boot_params.screen_info.orig_video_points` com:

```C
set_fs(0);
font_size = rdfs16(0x485);
boot_params.screen_info.orig_video_points = font_size;
```

Primeiro de tudo nós colocamos 0 no registro `FS` com a função `set_fs`. Nós já vimos a função como `set_fs` na parte anterior. Eles são todos definidos no [arch/x86/boot/boot.h](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/boot.h). Nós vimos o valor que é localizado no endereço `0x485` (localização da memória é usada para ter o tamanho da fonte) e salvar o tamanho da fonte em `boot_params.screen_info.orig_video_points`.


```C
x = rdfs16(0x44a);
y = (adapter == ADAPTER_CGA) ? 25 : rdfs8(0x484)+1;
```

Depois, obtemos a quantidade de colunas pelo endereços `0x44a` e linhas pelo endereço `0x484` e armazena eles em `boot_params.screen_info.orig_video_cols` e `boot_params.screen_info.orig_video_lines`. Depois disso, execução do `store_mode_params` é concluída.

Depois iremos ver a função `save_screen` que salva o conteúdo da tela para o Heap. Essa função calota todos os dados que temos na função (como as linhas e colunas e outras coisas) anterior e guarda na estrutura  `saved_screen`, que é definido como:


```C
static struct saved_screen {
	int x, y;
	int curx, cury;
	u16 *data;
} saved;
```

Isso então checa se o Heap tem espaços livres para:

```C
if (!heap_free(saved.x*saved.y*sizeof(u16)+512))
		return;
```

e aloca espaços na Heap se é suficiente e armazena o `saved_screen`.

A próxima chamada é `probe_cards(0)` do arquivo do código fonte [arch/x86/boot/video-mode.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video-mode.c).  Está sobre todos video_cards e coleta o número de modos fornecido pela cards. Aqui é uma parte interessante, nós podemos ver o loop:


```C
for (card = video_cards; card < video_cards_end; card++) {
  /* collecting number of modes here */
}
```

mas `video_cards` não é declarado em qualquer lugar. A resposta é simples: todo modo vídeo presente no código de inicialização do kernel x86 tem definição que parecida com isso:

```C
static __videocard video_vga = {
	.card_name	= "VGA",
	.probe		= vga_probe,
	.set_mode	= vga_set_mode,
};
```

onde `__videocard` é uma macro:

```C
#define __videocard struct card_info __attribute__((used,section(".videocards")))
```

o que significa que a estrutura do `card_info`:

```C
struct card_info {
	const char *card_name;
	int (*set_mode)(struct mode_info *mode);
	int (*probe)(void);
	struct mode_info *modes;
	int nmodes;
	int unsafe;
	u16 xmode_first;
	u16 xmode_n;
};
```

está nos segmento `.videocards`. Deixe olhar no script do linker onde podemos encontrar:

```
	.videocards	: {
		video_cards = .;
		*(.videocards)
		video_cards_end = .;
	}
```

Isso significa que o `video_cards` é apenas um endereço de memória e todas estruturas `card_info` são colocadas neste segmento. Significa que todas as estrutura `card_info` são colocados entre `video_cards` e `video_cards_end`, então podemos usar o loop sobre todos eles. Depois de executar `probe_cards` teremos várias estruturas como `static __videocard video_vga` com o `nmodes` (o número do modo de vídeo) preenchidos.

Depois a função `probe_cards` é concluída, moveremos para o principla loop na função `set_video`. Tem um loop infinito que tenta configurar o modo vídeo com a função `set_mode` ou mostra um menu se passarmos `vid_mode=ask` para a linha de comando do kernel ou se modo vídeo está indefinido.

A função `set_mode` é definida em [video-mode.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/video-mode.c) e tem um parâmetro `mode`, o parâmetro é o número do modos do vídeo (esse valor é do menu ou no começo do `setup_video` do header de inicialização do kernel).

A função `set_mode` checa o `mode` e chama a função `raw_set_mode`. A `raw_set_mode` chama o card selecionado da função `set_mode`, isto é, `card->set_mode(struct mode_info*)`. Poderemos ter acesso para essa função da estrutura `card_info`. Todo modo vídeo define essa estrutura com valores preenchidos dependendo do modo vídeo (por exemplo pela `vga` é na função `video_vga.set_mode`. Veja acima o exemplo da estrutura `card_info` para `vga`). `video_vga.set_mode` é `vga_set_mode` ue checa o modo vga e chama a respectiva função:

```C
static int vga_set_mode(struct mode_info *mode)
{
	vga_set_basic_mode();

	force_x = mode->x;
	force_y = mode->y;

	switch (mode->mode) {
	case VIDEO_80x25:
		break;
	case VIDEO_8POINT:
		vga_set_8font();
		break;
	case VIDEO_80x43:
		vga_set_80x43();
		break;
	case VIDEO_80x28:
		vga_set_14font();
		break;
	case VIDEO_80x30:
		vga_set_80x30();
		break;
	case VIDEO_80x34:
		vga_set_80x34();
		break;
	case VIDEO_80x60:
		vga_set_80x60();
		break;
	}
	return 0;
}
```

Toda função que defini modo vídeo chama a interrupção da BIOS `0x10` como um certo valor no registro `AH`.

Depois teremos definido modo vídeo, passaremos para `boot_params.hdr.vid_mode`.

Em seguida `vesa_store_edid` é chamado. Essa função simplesmente armazena a informação [EDID](https://en.wikipedia.org/wiki/Extended_Display_Identification_Data) (**E**xtended **D**isplay **I**dentification **D**ata) para o kernel usar. Essa  `store_mode_params` é chamado de novo. Por último, se `do_restore` é definido, a tela é restaurada para o estado anterior. 

Feito isso, o ajuste para o modo vídeo está completo e agora podemos mudar para o modo protegido.


última preparação antes da transição no modo protegido
--------------------------------------------------------------------------------

Nós podemos ver a última chamada de função `go_to_protected_mode` [arch/x86/boot/main.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/main.c).

A função `go_to_protected_mode` é definido em [arch/x86/boot/pm.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pm.c). Contém algumas funções que faz o última preparação antes de nós podemos pular para o modo protegido, então observaremos, entender o que elas fazem e como faz.

Primeira chamada para a função `realmode_switch_hook` em `go_to_protected_mode`. Essa função invoca os hooks para mudar do modo real se estiver presente e desativar [NMI](http://en.wikipedia.org/wiki/Non-maskable_interrupt). Hooks são usado se o bootloader corre em um ambiente hostil. Você pode ler mais sobre hooks no [boot protocol](https://www.kernel.org/doc/Documentation/x86/boot.txt) (veja **ADVANCED BOOT LOADER HOOKS**).

O hook `realmode_switch` apresente um ponteiro para uma sub-rotina para o modo real 16-bit que desativa interrupções non-Maskable. Depois o hook `realmode_switch` (isto não é presente para mim) é checado, interruptor é non-Maskable Interrupts(NMI) é desabilitado:

```assembly
asm volatile("cli");
outb(0x80, 0x70);	/* Disable NMI */
io_delay();
```

Em primeiro, há uma declaração de assembly inline com uma instrução `cli` que limpa a flag (`IF`). Depois disso, interrupções externas são desabilitadas. O próximo linha desativa NMI (non-maskable interrupt).

Uma interrupção é um sinal para o CPU que é omitido por hardware ou software. Depois obter esse sinal, a CPU suspende o sequencia de instrução atual, salva o estado e transfere o controle para manipular a interrupção. Depois que o manipulador de interrupções termina o trabalho, ele transfere o controle de volta para a instrução interrompida. Non-maskable interrupts (NMI) são interrupções que são sempre processado, independentemente da permissão. Eles não pode ser ignorado e são tipicamente usado para o sinal para os erros de hardware non-recoverable. Nós não iremos mergulhar no detalhe de interrupções agora, mas iremos discuti-los no próximo posts.

Vamos voltar ao código. Nós podemos ver no segunda linha que nós estamos escrevendo o byte `0x80` (desabilitar bit) para `0x70` (o registro de endereço CMOS). Depois, ocorre a chamada de função `io_delay`. `io_delay` causa um pequeno atraso e parece como:

```C
static inline void io_delay(void)
{
	const u16 DELAY_PORT = 0x80;
	asm volatile("outb %%al,%0" : : "dN" (DELAY_PORT));
}
```
Saída de qualquer byte para a porta `0x80` deveria atrasar exatamente 1 microssegundo. Então nós podemos escrever qualquer valor (o valor do `AL` em nossa casa) para a porta `0x80`. Depois desse atraso, a função `realmode_switch_hook` foi concluída e podemos ir para próximo função.

A próximo função é `enable_a20`, que ativa o [A20 line](http://en.wikipedia.org/wiki/A20_line). Essa função é definida em [arch/x86/boot/a20.c](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/a20.c) e tenta habilitar a porta A20 com diferentes métodos. O primeiro é a função `a20_test_short` que checa se A20 está habilitada ou não com a função `a20_test`:


```C
static int a20_test(int loops)
{
	int ok = 0;
	int saved, ctr;

	set_fs(0x0000);
	set_gs(0xffff);

	saved = ctr = rdfs32(A20_TEST_ADDR);

        while (loops--) {
		wrfs32(++ctr, A20_TEST_ADDR);
		io_delay();	/* Serialize and make delay constant */
		ok = rdgs32(A20_TEST_ADDR+0x10) ^ ctr;
		if (ok)
			break;
	}

	wrfs32(saved, A20_TEST_ADDR);
	return ok;
}
```

Primeiro e tudo, nós colocamos `0x0000` no registro `FS` e `0xffff` no registro `GS`. Próximo, lemos o valor no endereço `A20_TEST_ADDR` e colocamos esse valor nas variáveis `saved` and `ctr`.

Em seguida escrevemos para atualizar o valor `ctr` no `fs:A20_TEST_ADDR` ou `fs:0x200` com a função `wrfs32`, então atrasa para 1ms, e então lê o valor do registro `GS` no endereço `A20_TEST_ADDR+0x10`. Em uma função quando linha `A20` é desabilitada, o endereço vai ser sobreposto, em outros casos se não for zero a linha `A20`, então já está habilitada A20.

Se A20 está desativada, temos habilitar com diferentes métodos que você poderá encontrar em  `a20.c`. Por exemplo, pode ser feito com um chamado para interrupção da BIOS `0x15` com `AH=0x2041`.

Se a função `enable_a20` terminar com um erro, mostra uma mensagem de erro e chama a função `die`. Você lembra do primeiro arquivo do código fonte  aonde começamos - [arch/x86/boot/header.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/header.S):


```assembly
die:
	hlt
	jmp	die
	.size	die, .-die
```

Continuando depois da porta A20 está habilitada sem problemas, a função `reset_coprocessor` é chamada:

```C
outb(0, 0xf0);
outb(0, 0xf1);
```

Essa função limpa o coprocessador matemático escrevendo `0` para `0xf0`, então reset escrevendo `0` para `0xf1`.

Continuando, a função `mask_all_interrupts` é chamada:

```C
outb(0xff, 0xa1);       /* Mask all interrupts on the secondary PIC */
outb(0xfb, 0x21);       /* Mask all but cascade on the primary PIC */
```

Essa mascara todas as interrupções no PIC (Programmable Interrupt Controller) secundário e PIC primário exceto  para IRQ2 no PIC primário.

Quando finalizar todas essas operações, vemos a transição atual para o modo protegido.


Defini a Interrupt Descriptor Table
--------------------------------------------------------------------------------

Agora nós definimos Interrupt Descriptor table (IDT) no função `setup_idt`: 

```C
static void setup_idt(void)
{
	static const struct gdt_ptr null_idt = {0, 0};
	asm volatile("lidtl %0" : : "m" (null_idt));
}
```

Que defini o Interrupt Descriptor Table (descreve manipuladores de interrupções e etc). Por agora, o IDT não e instalado (mais tarde), mas apenas carrega o IDT com a instrução `lidtl`. O `null_idt` contém o endereço e o tamanho do IDT, mas agora são zero. O `null_idt` é uma estrutura `gdt_ptr`, é definido como:

```C
struct gdt_ptr {
	u16 len;
	u32 ptr;
} __attribute__((packed));
```

Onde vemos comprimento (`len`) 16-bit do IDT e o ponteiro de 32-bit para isso (mais detalhes sobre o IDT e interrupções visto no próximo postagem). ` __attribute__((packed))` significa que o tamanho do `gdt_ptr` é o tamanho mínimo requerido. Então o tamanho do `gdt_ptr` será 6 bytes aqui ou 48 bits. (Iremos carrega o ponteiro `gdt_ptr` para o registro `GDTR` e lembre que 48-bits no tamanho).

Definir Global Descriptor Table
--------------------------------------------------------------------------------

Setup do Global Descriptor Table (GDT). Vemos na função `setup_gdt` que defini o GDT (ler sobre essa postagem [Kernel booting process. Part 2.](linux-bootstrap-2.md#protected-mode)). Existe uma definição do array `boot_gdt` nesta função, que contém a definição dos três segmentos:

```C
static const u64 boot_gdt[] __attribute__((aligned(16))) = {
	[GDT_ENTRY_BOOT_CS] = GDT_ENTRY(0xc09b, 0, 0xfffff),
	[GDT_ENTRY_BOOT_DS] = GDT_ENTRY(0xc093, 0, 0xfffff),
	[GDT_ENTRY_BOOT_TSS] = GDT_ENTRY(0x0089, 4096, 103),
};
```

Para codigo, dados e TSS (Task state segment). Não usaremos o TSS por agora. ele foi adicionado para fazer Intel VT happy como nós podemos ver no linha de comentário (se você está interessado pode encontrar o commit que descreve isso - [aqui](https://github.com/torvalds/linux/commit/88089519f302f1296b4739be45699f06f728ec31)). Deixe-nos observar o `boot_gdt`. Primeiro de tudo nota que tem o atributo `__attribute__((aligned(16)))`. Significa que essa estrutura  vai ser alinhada por 16 bytes.

Deixe-nos observar um exemplo simples:

```C
#include <stdio.h>

struct aligned {
	int a;
}__attribute__((aligned(16)));

struct nonaligned {
	int b;
};

int main(void)
{
	struct aligned    a;
	struct nonaligned na;

	printf("Not aligned - %zu \n", sizeof(na));
	printf("Aligned - %zu \n", sizeof(a));

	return 0;
}
```

Tecnicamente uma estrutura que contém 1 campo `int` deve ser 4 bytes de tamanho, mas uma estrutura `aligned` precisará de 16 bytes para memória.

```
$ gcc test.c -o test && test
Not aligned - 4
Aligned - 16
```

O `GDT_ENTRY_BOOT_CS` tem índice - 2 aqui, `GDT_ENTRY_BOOT_DS` é `GDT_ENTRY_BOOT_CS + 1` e etc. Ele começa do 2, porquê o primeiro é um descritor nulo obrigatório (índice - 0) e o segundo não está usado (index - 1).

`GDT_ENTRY` é um macro que recebe flags, base, limite e construir um entrada GDT. Por exemplo, deixe-nos olhar no entrada de segmento de código. `GDT_ENTRY` leva o valores seguidos:

* base  - 0
* limite - 0xfffff
* flags - 0xc09b

O que isso significa? O endereço base de segment é 0 e o limite (tamanho de segmento) é - `0xfffff` (1 MiB). Deixe-nos olhar no flags. Isso é `0xc09b` e será:

```
1100 0000 1001 1011
```

Em binário. Deixe-nos tentar entender o que todo bit significativo. Nós iremos ir aravés de todos os bits da esquerda para direita:

* 1    - (G) granularity bit
* 1    - (D) Se 0 segmento 16-bit; 1 = segmento 32-bit
* 0    - (L) executado em modo 64-bit se 1
* 0    - (AVL) disponível para usar pelo software do sistema
* 0000 - comprimento 4-bit 19:16 bits no descritor
* 1    - (P) presença de sgmento na memória
* 00   - (DPL) - nível de privilégio, 0 é alto privilégio
* 1    - (S) segmento de código ou dados, não um segmento de sistema
* 101  - tipo de segmento executar/ler
* 1    - bit acessado

Você pode ler mais na [postagem](linux-bootstrap-2.md) anterior ou no [Intel® 64 and IA-32 Architectures Software Developer's Manuals 3A](http://www.intel.com/content/www/us/en/processors/architectures-software-developer-manuals.html).

Depois esse recebemos o comprimento do GDT com:

```C
gdt.len = sizeof(boot_gdt)-1;
```

Nós recebemos o tamanho do `boot_gdt` e subtrai 1 (o último endereço válido no GDT).

Nós obtemos um ponteiro para o GDT com:

```C
gdt.ptr = (u32)&boot_gdt + (ds() << 4);
```

Aqui nós temos o endereço do `boot_gdt` e adicionamos para o endereço do segmento de dados deslocar para esquerda por 4 bis (lembre que estamos em modo real).


Por último executamos a instrução `lgdtl` para carregar o GDT no registro GSTR:

```C
asm volatile("lgdtl %0" : : "m" (gdt));
```

Transição atual no modo protegido
--------------------------------------------------------------------------------




This is the end of the `go_to_protected_mode` function. We loaded the IDT and GDT, disabled interrupts and now can switch the CPU into protected mode. The last step is calling the `protected_mode_jump` function with two parameters:

```C
protected_mode_jump(boot_params.hdr.code32_start, (u32)&boot_params + (ds() << 4));
```

which is defined in [arch/x86/boot/pmjump.S](https://github.com/torvalds/linux/blob/v4.16/arch/x86/boot/pmjump.S).

It takes two parameters:

* address of the protected mode entry point
* address of `boot_params`

Let's look inside `protected_mode_jump`. As I wrote above, you can find it in `arch/x86/boot/pmjump.S`. The first parameter will be in the `eax` register and the second one is in `edx`.

First of all, we put the address of `boot_params` in the `esi` register and the address of the code segment register `cs` in `bx`. After this, we shift `bx` by 4 bits and add it to the memory location labeled `2` (which is `(cs << 4) + in_pm32`, the physical address to jump after transitioned to 32-bit mode) and jump to label `1`. So after this `in_pm32` in label `2` will be overwritten with `(cs << 4) + in_pm32`.

Next we put the data segment and the task state segment in the `cx` and `di` registers with:

```assembly
movw	$__BOOT_DS, %cx
movw	$__BOOT_TSS, %di
```

As you can read above `GDT_ENTRY_BOOT_CS` has index 2 and every GDT entry is 8 byte, so `CS` will be `2 * 8 = 16`, `__BOOT_DS` is 24 etc.

Next, we set the `PE` (Protection Enable) bit in the `CR0` control register:

```assembly
movl	%cr0, %edx
orb	$X86_CR0_PE, %dl
movl	%edx, %cr0
```

and make a long jump to protected mode:

```assembly
	.byte	0x66, 0xea
2:	.long	in_pm32
	.word	__BOOT_CS
```

where:

* `0x66` is the operand-size prefix which allows us to mix 16-bit and 32-bit code
* `0xea` - is the jump opcode
* `in_pm32` is the segment offset under protect mode, which has value `(cs << 4) + in_pm32` derived from real mode
* `__BOOT_CS` is the code segment we want to jump to.

After this we are finally in protected mode:

```assembly
.code32
.section ".text32","ax"
```

Let's look at the first steps taken in protected mode. First of all we set up the data segment with:

```assembly
movl	%ecx, %ds
movl	%ecx, %es
movl	%ecx, %fs
movl	%ecx, %gs
movl	%ecx, %ss
```

If you paid attention, you can remember that we saved `$__BOOT_DS` in the `cx` register. Now we fill it with all segment registers besides `cs` (`cs` is already `__BOOT_CS`).

And setup a valid stack for debugging purposes:

```assembly
addl	%ebx, %esp
```

The last step before the jump into 32-bit entry point is to clear the general purpose registers:

```assembly
xorl	%ecx, %ecx
xorl	%edx, %edx
xorl	%ebx, %ebx
xorl	%ebp, %ebp
xorl	%edi, %edi
```

And jump to the 32-bit entry point in the end:

```
jmpl	*%eax
```

Remember that `eax` contains the address of the 32-bit entry (we passed it as the first parameter into `protected_mode_jump`).

That's all. We're in protected mode and stop at its entry point. We will see what happens next in the next part.

Conclusion
--------------------------------------------------------------------------------

This is the end of the third part about linux kernel insides. In the next part, we will look at the first steps we take in protected mode and transition into [long mode](http://en.wikipedia.org/wiki/Long_mode).

If you have any questions or suggestions write me a comment or ping me at [twitter](https://twitter.com/0xAX).

**Please note that English is not my first language, And I am really sorry for any inconvenience. If you find any mistakes, please send me a PR with corrections at [linux-insides](https://github.com/0xAX/linux-internals).**

Links
--------------------------------------------------------------------------------

* [VGA](http://en.wikipedia.org/wiki/Video_Graphics_Array)
* [VESA BIOS Extensions](http://en.wikipedia.org/wiki/VESA_BIOS_Extensions)
* [Data structure alignment](http://en.wikipedia.org/wiki/Data_structure_alignment)
* [Non-maskable interrupt](http://en.wikipedia.org/wiki/Non-maskable_interrupt)
* [A20](http://en.wikipedia.org/wiki/A20_line)
* [GCC designated inits](https://gcc.gnu.org/onlinedocs/gcc-4.1.2/gcc/Designated-Inits.html)
* [GCC type attributes](https://gcc.gnu.org/onlinedocs/gcc/Type-Attributes.html)
* [Previous part](linux-bootstrap-2.md)
