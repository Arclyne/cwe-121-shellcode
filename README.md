# CWE-121: Stack-based Buffer Overflow con Shellcode

Práctica guiada de **Programación Segura** que ejemplifica la vulnerabilidad
**CWE-121 (Stack-based Buffer Overflow)** mediante un ataque *"Smash the Stack"*.
Se inyecta un **shellcode** en la pila a través de una entrada maliciosa y se
sobrescribe la dirección de retorno para desviar el flujo de ejecución hacia él,
imprimiendo un mensaje (`ganaste angel!`) que **no existe** en el código original.

> Universidad Veracruzana — Facultad de Estadística e Informática

## Estructura del proyecto

| Archivo | Descripción |
|---|---|
| `programa.c` | Programa vulnerable: usa `gets()` sobre un `char buf[80]`. |
| `shellcode.asm` | Shellcode en ensamblador que imprime `ganaste angel!` vía `write`/`exit`. |
| `payload.py` | Genera el payload (NOPs + shellcode + padding + dirección de retorno). |
| `holanombre.asm` | Programa "hola mundo" en ensamblador previo al shellcode. |

## Cómo funciona

El programa declara dos variables locales, `cookie` y `buf`, y lee la entrada con
`gets(buf)` sin validar su longitud. El payload desborda `buf` y reescribe la
dirección de retorno para que apunte al inicio del búfer, donde se encuentra un
tobogán de NOPs (`\x90`) que conduce al shellcode.

Mapa de la pila (búfer de 80 bytes):

```
[ NOPs + shellcode + padding ] [ BBBB ] [ CCCC ] [ ret_addr ]
            buf (80)            cookie    ebp     dir. retorno
```

## Requisitos

- Linux de 32 bits (la práctica usa Ubuntu Server i686), `gcc`, `nasm`, `python` (2.x).

## Compilación y ejecución

```bash
# 1. Desactivar ASLR para que la dirección de la pila sea estable
sudo sysctl -w kernel.randomize_va_space=0

# 2. Compilar el shellcode
nasm -f elf shellcode.asm
ld -N shellcode.o -o shellcode.out

# 3. Compilar el programa vulnerable
gcc programa.c -o programa.out -fno-stack-protector -z execstack -ggdb -mpreferred-stack-boundary=2

# 4. Verificar la dirección de buf (debe ser constante entre corridas)
./programa.out

# 5. Ajustar ret_addr en payload.py con esa dirección y lanzar el exploit
python payload.py | ./programa.out
```

Salida esperada:

```
buf: bffff5b4 cookie: bffff604
ganaste angel!
```

## Notas

- **ASLR:** si no se desactiva (`kernel.randomize_va_space=0`), la dirección de la
  pila cambia en cada ejecución y el `ret_addr` fijo del payload falla con
  *violación de segmento*.
- El shellcode evita el byte `\x00` para no cortar la copia en memoria y usa la
  técnica `jmp`/`call`/`pop` para obtener la dirección de la cadena.
- Práctica con fines **educativos**: muestra por qué `gets()` y los búferes sin
  validación de longitud son peligrosos (CWE-121).
