# Kernel Boot Process

Esse capítulo descreve o processo boot do kernel Linux. Aqui você verá uma série de postagem o que descreve todos os ciclos de carregamento do processo no kernel:


* [Do bootloader ao kernel](linux-bootstrap-1-pt.md) - descreve todos os estágios de ligar o computador ao executar o primeira instrução do kernel Linux.
* [Primeiros passo na código de configuração do Kernel](linux-bootstrap-2-pt.md) - Descreve primeiros passos no código de configuração do Kernel. Você vai ver a inicialização do heap, query de diferentes parâmetros como EDD, IST e etc...
* [Inicialização do modo vídeo e a transição para o modo protegido](linux-bootstrap-3-pt.md) - descreve inicialização do modo vídeo no código de configuração do kernel e transição para o modo protegido.
* [Transição para o modo 64-bit](linux-bootstrap-4-pt.md) - descreve preparação para a transição no modo 64-bit e detalhes da transição
* [Descompressão do Kernel](linux-bootstrap-5-pt.md) - descreve preparação antes descomprimir o kernel e detalhes da descompressão direta.
* [Endereço aleatório carregamento do kernel](linux-bootstrap-6-pt.md) - descreve 
o aleatóriamente do endereço de carregamento do kernel Linux.


Esse capítulo coincide com `Linux kernel v4.17`.
