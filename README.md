# pwn
Prática sobre pwn referente à matéria de Segurança Cibernética

Resolução do exercício do site [Hack The Box](https://app.hackthebox.com/challenges/restaurant)

## Descrição do problema
Bem-vindo ao nosso restaurante. Aqui você pode comer e beber o quanto quiser! Só não exagere...

## Analisando o problema
Descompactando o arquivo .zip, resulta em 2 arquivos: um arquivo de biblioteca libc e um arquivo binário.

# Imagem foda

Vamos verificar o tipo binário e suas proteções.

# Imagem foda (64 bit binary file, dynamically linked, not stripped.)
# Imagem foda (VULN -> No Canary Found, No PIE (ASLR disabled))

Dado um arquivo de biblioteca libc com a vulnerabilidade que obtivemos do arquivo binário, sabemos que a exploração que faremos é o ataque ret2libc. Agora vamos descompilar o binário.

![função main do binário](https://github.com/caiocadini/pwn/assets/100872066/afa0fe88-66e2-4d79-bc03-ca466a5d2676 "Função main")


Analisando a função principal, se a entrada do usuário for 1, o usuário deverá pular para a função fill() e se a entrada for 2, o usuário deverá pular para a função drink(). Vamos verificar fill() primeiro.

![função fill do binário](https://github.com/caiocadini/pwn/assets/100872066/cd33ee9a-5926-4596-9d82-a3b56f1580e1 "Função fill")


Observe que podemos vazar o endereço de tempo de execução da libc por meio da chamada puts. Também podemos estourar o buffer da variável local_28 para controlar o RIP. Vamos verificar a função drink() agora.

![função drink do binário](https://github.com/caiocadini/pwn/assets/100872066/622371a3-4739-4fd6-ad0e-3a4dc2ce72fc "Função drink")


Nada de interessante aqui, parece que a função fill() será de nosso interesse agora.

## Vazando o endereço de tempo de execução libc

### Obtenha o deslocamento RIP


