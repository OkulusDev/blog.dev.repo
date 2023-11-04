+++
title = 'PutsF: Альтернатива printf для C'
date = 2023-11-03T19:14:54+07:00
draft = true
tags = ['ассемблер', 'assembly', 'низкоуровневое программирование', 'компьютер', 'c', 'fasm']
categories = ['assembly', 'linux', 'elf64', 'asm', 'fasm', 'c']
+++

# PutsF - альтернатива printf

Это небольшая библиотека на ассемблере (компилятор FASM), реализация printf из stdlib, для C

## Установка

Эта программа есть в [моем гитхаб репозитории](https://github.com/OkulusDev/asm-putsf)

Код написан для 64 битной linux системы (ELF 64). Если у вас другая, вам придется редактировать код, указывать другой формат, если у вас и разрядность другая - то изменять имена регистров.

Также у вас должен быть установлен gnu-линковщик, fasm (flat assembly) и система сборки make.

Если у вас возникли сложности или вопросы по использованию Metalfish OS, создайте 
[обсуждение](https://github.com/OkulusDev/asm-putsf/issues/new/choose) в данном репозитории или напишите на электронную почту <bro.alexeev@gmail.com>.

```bash
# Клонирование репозитория
git clone https://github.com/OkulusDev/asm-putsf.git
cd asm-putsf

# компиляция и линковка
make build clean

# запуск
make run
```

```c
typedef long long int int64_t;

extern void c_exit(int ret);
extern int64_t c_putsf(char *fmt, ...);

void _start(void) {
    char *string = "PutsF";
    int64_t decimal = 123;
    char symbol = '!';

    int64_t ret = c_putsf(
        "{ %s, %d, %c }\n",
        string, decimal, symbol
    );
    c_putsf("%d\n", ret); // print 3

    c_exit(0);
}
```

Вот флаги для компилятора gcc: ```gcc -nostdlib -o <нелинкованные файлы) -o <main.c>```

Компиляция в Makefile:

```bash
fasm src/putsf.asm bin/putsf.o
fasm src/c_putsf.asm bin/c_putsf.o
fasm src/c_exit.asm bin/c_exit.o
gcc -nostdlib -o bin/putsf.bin bin/putsf.o bin/c_putsf.o bin/c_exit.o bin/putsf_example.c
```

## Код putsf.asm

```asm
; -----------------------------------------------------------------------------
;  ASM PUTSF Source Code
;  File: putsf.asm
;  Title: Работа кода putsf
;  Last Change Date: 2 November 2023, 14:10 (UTC)
;  Author: Okulus Dev
;  License: GNU GPL v3
; -----------------------------------------------------------------------------
; Description: 
;   putsf - это альтернатива printf. PutsF минималичстичный, ограниченн символами
;  %c, %s, %d и %%. 
;   TODO: добавить реализации с %f (плавающие числа), %x (hex, 16-ые числа)
; -----------------------------------------------------------------------------

format ELF64							; указываем 64 битный линуксовый формат
; 64-битный формат означает, что ко всем регистрам будем добавлять букву r
; (например rax вместо ax). Буква R означает, что регистр 64-битный.
; Регистры бывают разных типов: AH, AL, AX, EAX, RAX - это все один регистр.
;  + RAX - 64 битный (8 байт)
;  + EAX - 32 битный (4 байта)
;  + AX - 16 битный (2 байта)
;  + AH, AL - 8 байтные (1 байт)
; Регистр RAX это дополнение EAX, EAX это дополнение AX, AX это дополнение 2
; регистров AH и AL

include "puts_decimal.asm"
include "puts_string.asm"
include "puts_char.asm"

public putsf

section '.putsf' executable				; секция putsf
; Ввод:
;  rax = format (rax = формат)
;  stack = values (stack = значения)
; Вывод:
;  rax = число, количество
putsf:									; метка putsf
	; Логика довольно простая, т.к. основные алгоритмы выполнены
	; Алгоритм:
	; 1. Прочитать i символ в строке
	; 2. Если символ равен %, тогда перейти на пункт 6
	; 3. Если символ равен нулю, тогда завершить выполнение
	; 4. Иначе, напечатать i символ
	; 5. Инкрементировать значение i (i++)
	; 6. Перейти на пункт 1
	; 7. Инкрементировать значение i (i++)
	; 8. Если i символ равен %, тогда напечатать его
	; 9. Если i символ равен d, тогда взять из стека значение и напечатать число
   	; 10. Если i символ равен s, тогда взять из стека значение и напечатать строку 	
	; 11. Если i символ равен c, тогда взять из стека значение и напечатать символ
	; 12. Иначе, вернуть ошибку и перейти на пункт 3
	; 13. Инкрементировать значение i
	; 14. Перейти на пункт 1
	push rbx
    push rcx

    ; call/ret    = 8byte
    ; rax+rbx+rcx = 24byte
    mov rbx, 32

    ; count of format elements
    xor rcx, rcx 
    .next_iter:
        cmp [rax], byte 0
        je .close
        cmp [rax], byte '%'
        je .special_char
        jmp .default_char
        .special_char:
            inc rax
            cmp [rax], byte 's'
            je .print_string
            cmp [rax], byte 'd'
            je .print_decimal
            cmp [rax], byte 'c'
            je .print_char
            cmp [rax], byte '%'
            je .default_char
            jmp .is_error
        .print_string:
            push rax
            mov rax, [rsp+rbx]
            call puts_string
            pop rax
            jmp .shift_stack
        .print_decimal:
            push rax
            mov rax, [rsp+rbx]
            call puts_decimal
            pop rax
            jmp .shift_stack
        .print_char:
            push rax
            mov rax, [rsp+rbx]
            call puts_char
            pop rax
            jmp .shift_stack
        .default_char:
            push rax
            mov rax, [rax]
            call puts_char
            pop rax
            jmp .next_step
        .shift_stack:
            inc rcx
            add rbx, 8
        .next_step:
            inc rax
            jmp .next_iter
    .is_error:
        mov rcx, -1
    .close:
        mov rax, rcx
        pop rcx
        pop rbx
        ret
```

## Код puts_char.asm

```asm
;; puts_char.asm
; Вывод символа

section '.puts_char' executable			; секция вывода символа
; Ввод:
;  rax = char (регистр rax = символ)
puts_char:
    push rax
    push rdx
    push rsi
    push rdi

    push rax
    
    mov rsi, rsp
    mov rdi, 1
    mov rdx, 1
    mov rax, 1
    call do_syscall

    pop rax

    pop rdi
    pop rsi
    pop rdx
    pop rax
    ret

section '.do_syscall' executable		; секция сисвызова
do_syscall:								; метка сисвызова
	push rcx
	push r11

	syscall
	
	pop r11
	pop rcx

	ret
```

## Код puts_string.asm

```asm
;; puts_string.asm
; Вывод строки

section '.puts_string' executable		; секция вывода строки
; Ввод:
;  rax = string (регистр rax = строка для вывода)
puts_string:							; метка вывода строки
	push rbx
	xor rbx, rbx

	.next_iter:
		cmp [rax+rbx], byte 0
		je .close
		push rax
		mov rax, [rax+rbx]
		call puts_char
		pop rax
		inc rbx
		jmp .next_iter
	.close:
		pop rbx
		ret
```

## Код puts_decimal.asm

```asm
;; pust_decimal.asm
; Вывод 64 битного числа

section '.puts_decimal' executable		; секция вывода числа
; Ввод:
;  rax - число
puts_decimal:							; метка вывода числа
	; Стоит учитывать, что данная метка работает только с 64-битными числами, и
	; следовательно, и отрицательное число она будет трактовать если оно будет
	; 64-битным. В противном случае, если число будет 32 битное, то оно со
	; стороны 64-битного числа будет трактоваться как unsigned (число без знака) 
	; Алгоритм:	
	; 1. Если число меньше нуля, то напечать символ минус.
	; 2. Разделить число на 10, взять частное и остаток от деления
	; 3. Положить остаток в стек
	; 4. Инкрементировать значение i (i++)
	; 5. Если частное не равно 0, тогда перейти на 2 пункт
	; 6. Если значение i равно 0, тогда закрыть выполнение
	; 7. Выгрузить число из стека (остаток)
	; 8. Привать к остатку символ '0' (число 48 по ASCII)
	; 9. Напечатать получившийся символ (puts_char)
	; 10. Декрементировать значение i (i--)
	; 11. Перейти на пункт 6
	push rax
    push rbx
    push rcx
    push rdx
    xor rcx, rcx
    cmp rax, 0
    
	jl .is_minus
    jmp .next_iter

    .is_minus:
        neg rax
        push rax
        mov rax, '-'
        call puts_char
        pop rax

    .next_iter:
        mov rbx, 10
        xor rdx, rdx
        div rbx
        push rdx
        inc rcx
        cmp rax, 0
        je .puts_iter
        jmp .next_iter

    .puts_iter:
        cmp rcx, 0
        je .close
        pop rax
        add rax, '0'
        call puts_char
        dec rcx
        jmp .puts_iter

    .close:
        pop rdx
        pop rcx
        pop rbx
        pop rax
        ret
```

## Код c_exit.asm

```asm
; c_exit.asm
format ELF64

public c_exit

section '.c_exit' executable
c_exit:
    mov rax, 60
    syscall 
```

## Код c_putsf.asm

```asm
; -----------------------------------------------------------------------------
;  ASM PUTSF Source Code
;  File: cputsf.asm
;  Title: Связка putsf с C
;  Last Change Date: 2 November 2023, 14:10 (UTC)
;  Author: Okulus Dev
;  License: GNU GPL v3
; -----------------------------------------------------------------------------
; Description: 
;   putsf - это альтернатива printf. PutsF минималичстичный, ограниченн символами
;  %c, %s, %d и %%. 
;   TODO: добавить реализации с %f (плавающие числа), %x (hex, 16-ые числа)
; -----------------------------------------------------------------------------
format ELF64

extrn putsf

public c_putsf

section '.c_putsf' executable
c_putsf:
    pop r10

    push r9
    push r8
    push rcx
    push rdx
    push rsi

    mov rax, rdi
    call putsf

    pop rsi
    pop rdx
    pop rcx
    pop r8
    pop r9

    push r10 
    ret 
```

## Код примера работы на C

```c
typedef long long int int64_t;

extern void c_exit(int ret);
extern int64_t c_putsf(char *fmt, ...);

void _start(void) {
    char *string = "PutsF";
    int64_t decimal = 123;
    char symbol = '!';

    int64_t ret = c_putsf(
        "{ %s, %d, %c }\n",
        string, decimal, symbol
    );
    c_putsf("%d\n", ret); // 3

    c_exit(0);
}
```

## Makefile

```make
# Директории
SRC_DIR=src
BIN_DIR=bin

# Компиляторы, линковщики
ASM=fasm
LD=ld
C=gcc
C_FLAGS=-nostdlib

# Исходный код, бинарники, вывод
CODE=putsf.asm
.phony: build clean run clean_all

build:
	$(ASM) $(SRC_DIR)/putsf.asm $(BIN_DIR)/putsf.o
	$(ASM) $(SRC_DIR)/c_putsf.asm $(BIN_DIR)/c_putsf.o
	$(ASM) $(SRC_DIR)/c_exit.asm $(BIN_DIR)/c_exit.o
	$(C) $(C_FLAGS) -o $(BIN_DIR)/putsf.bin $(BIN_DIR)/putsf.o $(BIN_DIR)/c_putsf.o $(BIN_DIR)/c_exit.o $(SRC_DIR)/putsf_example.c

run:
	./$(BIN_DIR)/putsf.bin

clean:
	rm $(BIN_DIR)/*.o

clean_all:
	rm $(BIN_DIR)/*
```

### Ставьте звезды на гитхабе, спасибо за прочтение!
