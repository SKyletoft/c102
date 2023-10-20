## Minne

Vad är en pekare?

Det är *inte* en minnesaddress. En pekare är ett magiskt okänt värde
som oftast, men inte alltid, representeras med en
minnesaddress. T.ex. pekare på osx och cheri.

Det finns två sätt att få en väldefinierad pekare i C. Antingen ta
addressen av ett existerande objekt med `&`:

```c
int x;
int *x_ptr = &x;
```

Eller från någon av standardbibliotekets dynamiska
allokeringsfunktioner, `malloc`, `realloc`, `calloc` eller
`aligned_alloc`:

```c
int *x_ptr = (int *) malloc(sizeof(int));
```

Det enda sättet att komma åt hårdvaruregister i välformad C är att
kalla på en assemblyfunktion (antingen byggd separat eller med inline
assembly med en GCC-extension). Dock är makrotricket som ni tidigare
sett och använt tillräckligt välanvänt att kompilatorn borde
respektera det oftast, så länge ni använder `volatile` korrekt.

## Dynamiskt minne

Ta exemplet:
```c

int *get_pointer() {
    int nums[10] = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    int *third = &nums[3];
    return third;
}
```

Vad händer här?
Vi skapar en array på 10 siffror.
Sen tar vi en pekare till 4:an och returnerar den.
Vad händer efter det?
