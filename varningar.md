Finn massa fel:

```c
typedef struct MyStruct {
    int a;
    float b;
    long c;
} MyStruct;

MyStruct new_mystruct(float a, long b, int c) {
    return (MyStruct) {
        .a = a,
        .b = b,
    };
}

int main() {
    MyStruct ms = new_mystruct(1, 2, 3);

    printf(
        "%ld\n",
        ms.a,
        ms.b,
        ms.c
    );
}
```
https://godbolt.org/z/eWhM7E14c

-Wall -Wextra -Werror=format -xc++ -Werror=float-conversion -Werror=narrowing -Werror=missing-field-initializers

1. Saknad include
2. Saknat .c i compound-literalen
3. Oanvänd parameter
4. Formatera int som long
5. Oanvända printf-argument
6. iffy typkonverteringar
