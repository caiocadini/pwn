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


