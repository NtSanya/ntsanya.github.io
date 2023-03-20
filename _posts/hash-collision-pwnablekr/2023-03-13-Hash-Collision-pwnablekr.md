---
title:  "Hash Collision - pwnable.kr - решение"
mathjax: true
layout: post
date:   2023-03-13 00:00:30 +0300
description: CTF Writeup pwnable.kr
tags:   [CTF, pwnable.kr, hash]
image:  '/media/posts_thumb/collision.png'
---



# решение задания collision - pwnable.kr







## что такое [хеш-функция](https://ru.wikibrief.org/wiki/Cryptographic_hash_function)?



>  криптографическая хеш-функция (CHF ) - математический [алгоритм](https://ru.wikibrief.org/wiki/Algorithm), который [отображает](https://ru.wikibrief.org/wiki/Map_(mathematics)) данные произвольных размер (часто называемый «сообщением») в [битовый массив](https://ru.wikibrief.org/wiki/Bit_array) фиксированного размера («хеш-значение», «хеш» или «дайджест сообщения»[^1]



Хеш-функция берёт определённую информацию и преобразует эту информацию в строку определенной длины. Исходные данные могут быть любыми, от пароля вашего аккаунта до фильмов или книг.  Эта строка всегда будет иметь **одинаковую длину** вне зависимости от того, какого размера была входная информация.

Есть очень важный момент:

> - небольшое изменение в сообщении должно изменить значение хеш-функции настолько сильно, что новое значение хеш-функции будет не коррелировать со старым хеш-значение ([лавинный эффект](https://ru.wikibrief.org/wiki/Avalanche_effect) )[^1]



Например[^2]:



![хеш](/media/hash-collision-pwnablekr/Cryptographic-Hashing-Explained-Rus.png)





## pwnable.kr - col



![banner](/media/hash-collision-pwnablekr/banner.png)



Итак, у нас есть 3 файла, слева мы видим наши полномочия, где 's' означает SUID[^3], то есть, мы можем запускать файл с правами владельца. 



давайте посмотрим содержимое файла *col.c*

 

```c
#include <stdio.h>
#include <string.h>
unsigned long hashcode = 0x21DD09EC;
unsigned long check_password(const char* p){
        int* ip = (int*)p;
        int i;
        int res=0;
        for(i=0; i<5; i++){
                res += ip[i];
        }
        return res;
}

int main(int argc, char* argv[]){
        if(argc<2){
                printf("usage : %s [passcode]\n", argv[0]);
                return 0;
        }
        if(strlen(argv[1]) != 20){
                printf("passcode length should be 20 bytes\n");
                return 0;
        }

        if(hashcode == check_password( argv[1] )){
                system("/bin/cat flag");
                return 0;
        }
        else
                printf("wrong passcode.\n");
        return 0;
}
```



Программа принимает в качестве аргумента строку из 20 символов. Строка предстаёт в виде 5 чисел, эти числа суммируются и их сумма проверятся со значением 0x21DD09EC.

Проблема в том, что 0x21DD09EC не делится на 5. Нам нужно задать 5 чисел, сумма которых равна 0x21DD09EC.

Давайте поделим 0x21DD09EC  на 5, возьмём результат без остатка, умножим на 4 и вычтем из 0x21DD09EC. 







![решение-col](/media/hash-collision-pwnablekr/решение-col.png)





итак, наша строка будет выглядеть следующим образом: **0x6c5cec8 * 4 + 0x6c5cecc**.



 ![решение2-col](/media/hash-collision-pwnablekr/решение2-col.png)



## Список литературы:

[^1]: https://ru.wikibrief.org/wiki/Cryptographic_hash_function
[^2]:https://www.kaspersky.ru/blog/the-wonders-of-hashing/3633/
[^3]: https://ru.wikipedia.org/wiki/Suid

