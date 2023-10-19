# Odefinierat beteende

## Ett exempel

Vi börjar med ett javaexempel

```java
class Main {
    static int f(Integer num) {
        int x = num;

        if (num == null) {
            System.out.println("num was null!");
            return -1;
        }

        return x + 5;
    }

    public static void main(String[] args) {
        System.out.println(f(null));
    }
}

> Exception in thread "main" java.lang.NullPointerException: Cannot
> invoke "java.lang.Integer.intValue()" because "<parameter1>" is null
>   at Main.f(example.java:3)
>   at Main.main(example.java:14)
```

Vad händer här?
Vi läser från num före vi kollar om det är null och krashar med en
NullPointerException.  För att fixa det här så behöver vi flytta ner
läsningen till efter checken:

```java
class Main {
    static int f(Integer num) {
        if (num == null) {
            System.out.println("num was null!");
            return -1;
        }

        int x = num;
        return x + 5;
    }

    public static void main(String[] args) {
        System.out.println(f(null));
    }
}

> num was null!
> -1
```

Då kan vi titta på samma program i C istället:
```c
#include <stdio.h>

int f(int *num_ptr) {
    int x = *num_ptr;

    if (num_ptr == NULL) {
        puts("The pointer was null!");
        return -1;
    }

    return x + 5;
}
```

Samma problem finns kvar, läsning före check. Så vad händer här?
```asm
.LC0:
        .ascii  "The pointer was null!\000"
f(int*):
        push    {r7, lr}
        sub     sp, sp, #16
        add     r7, sp, #0
        str     r0, [r7, #4]
        ---
        ldr     r3, [r7, #4]
        ldr     r3, [r3]
        str     r3, [r7, #12]
        ldr     r3, [r7, #4]
        cmp     r3, #0
        bne     .L2
        ---
        movw    r0, #:lower16:.LC0
        movt    r0, #:upper16:.LC0
        bl      puts
        mov     r3, #-1
        b       .L3
.L2:
        ldr     r3, [r7, #12]
        adds    r3, r3, #5
.L3:
        mov     r0, r3
        adds    r7, r7, #16
        mov     sp, r7
        pop     {r7, pc}
```

Vi sparar returaddresser, reserverar lite mer stackutrymme men sen
börjar vår kod. Vi läser in `num_ptr` till r3 från stacken (r7 + 4),
sen läser vi in `x` till r3, sparar `x` till stacken (r7 + 12), läser
in `num_ptr` igen, jämför med 0 och sen hoppar till .L2 om de skiljer
sig åt. I .L2 läser vi in `x` från stacken igen och adderar 5 innan vi
sen går genom hela returgrejen.
Om vi inte hade hoppat till .L2 så hade vi läst in .LC0 i två steg och
sen kallat `puts` innan vi laddade -1 och returnerade.

Ser rätt jobbigt ut, va? Inte särskilt effektivt. Ni hade säkert kunna
skriva det bättre själva. Men det är inte det bästa kompilatorn kan
göra. Vi kan faktiskt sätta på kompilatoroptimeringar för att göra
hela grejen snabb.

```asm
f(int*):
        ldr     r0, [r0]
        adds    r0, r0, #5
        bx      lr
```

Oj.

Där försvann all vår kod. Det här visar på den stora skillnaden mellan
C/C++ och nästan alla andra språk. I C har vi något som kallas för
odefinierat beteende (Undefined Behaviour, UB).

## The Abstract Machine

Cs semantik är definierad i form av en abstrakt maskin och inte någon
form av beskrivning av vad faktiska maskiner ska
göra. C-implementationer försöker härma den här abstrakta maskinen på
faktisk hårdvara. Uttrycket 1 + 2 behöver inte vara en addition utan
behöver bara härma alla synliga resultat av operationen, i detta fall
att det ska bli 3. En C-kompilator får alltså ersätta uttrycket mot
resultatet direkt eftersom inga mellansteg är synliga på den abstrakta
maskinen.

Men maskinen är inte väldefinierad och många operationer har
odefinierade resultat. Till exempel overflowande addition av två
ints. Detta var ursprungligen till för att tillåta
implementationsfrihet. Hade man definierat heltal som 2s kompliment
hade det varit väldigt jobbigt att implementera addition på en dator
med 1-komplimenthårdvara.

När en odefinierad operation sker så är allt som sker efter helt
odefinierat, det finns inga garantier om något, inte ens annars
väldefinierade operationer. Man brukar skämta om näsdemoner (nasal
demons).

## Out-of-contract

Det finns även andra skäl att en operation skulle vara
odefinierad. Titta på det här haskellexemplet:

```hs
λ> length [0..]
```

Vad händer här? Vi försöker beräkna längden på en oändlig lista. Är
det här rimligt? Kanske det.
Titta på det här istället:

```c
int strlen(char *c) {
    int len = 0;
    while (*c++) {
        len++;
    }
    return len;
}
```

Här beräknar vi längden på en sträng. Men vi har ett viktigt antagande
nu. Vi antar att strängen avslutas med en nolla, en
nullterminator. Men om den inte gör det? Odefinierat.

Det är många operationer i C och funktioner i standardbiblioteket som
gör sånna antaganden.

I modern C++ används termerna wide and narrow contract.  Breda
kontrakt är operationer som inte har några krav som måste uppfyllas
först och är väldefinierade för alla inputs, smala kontrakt är
motsatsen, de har krav du måste uppfylla för att inte orsaka UB.

## Implementation defined behaviour

Om vi nu backar till 80-talets implementationsfrihet så har vi också
en annan typ av beteende, *implementationsdefinierat
beteende*. Resultatet från en sån operation är inte specifierad av
standarden, men lovas ändå ha ett välformat resultat. Ett exempel på
detta är hur många bitar som är i en int. Standarden säger att det
måste vara minst 16, men den exakta bredden varierar från platform
till platform. Just en int är 32 bitar i princip överallt idag, men om
vi tittar på nästa storlek upp, på en long så har vi rätt stor
variation. På en 32-bitmaskin så är den 32-bitar och på en 64-bitars
Unixmaskin (Linux/Mac/*BSD) är den 64 bitar. Men på 64bit Windows är
den fortfarande 32 bitar.

På MD407:an så är en long 32 bitar.

(Tips: Använd inte standardtyper, använd stdint.h)

## Optimeraren

Så, om vi nu backar längre, tillbaks till koden som försvann.

Här samspelade två regler för den abstrakta maskinen.

* Om odefinierat beteende sker får vad som helst hända och vi behöver
  inte bry oss.
* Att läsa från en nullpekare är odefinierat.

Så vad händer?

* Först läser vi från en pekare.
Om den inte är null så läser vi korrekt in till x.
Om den är null så spelar det inte någon roll vad som händer.

* Sen kollar vi om pekaren är null.
Vi har redan läst från pekaren, så antingen är den inte null och det
är fine, eller så är den null och vi är redan i odefinieratland.
Oavsett så kommer vi aldrig komma in i if-satsen i väldefinierad kod,
så då kan vi ju lika gärna ta bort den.

Och då hamnar vi där i började.

## Vanligt förekommande UB

Null pointer dereference
Dangling pointer derefererence
Comparison of two pointers of different provinence (later lecture)
Out of bounds access

Signed integer overflow
Too large bitshift
Integer division by zero

Overlapping memcpy

Basically any pointer cast
Modifying a const object
