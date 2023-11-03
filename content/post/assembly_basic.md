+++
title = 'Основные инструкции ассемблера x86_64'
date = 2023-11-03T19:06:04+07:00
draft = true
tags = ['ассемблер', 'assembly', 'низкоуровневое программирование', 'компьютер']
categories = ['assembly', 'linux']
+++

# Шпаргалка по основным инструкциям ассемблера x86/x64

[В прошлой статье](https://okulusdev.github.io/post/write_debug_asm_code) мы написали наше первое hello world приложение на асме, научились его компилировать и отлаживать, а также узнали, как делать системные вызовы в Linux. Сегодня же мы познакомимся непосредственно с ассемблерными инструкциями, понятием регистров, стека и вот этого всего. Ассемблеры для архитектур x86 (a.k.a i386) и x64 (a.k.a amd64) очень похожи, в связи с чем нет смысла рассматривать их в отдельных статьях. Притом акцент я постараюсь делать на x64, попутно отмечая отличия от x86, если они есть. Далее предполагается, что вы уже знаете, например, чем стек отличается от кучи, и объяснять такие вещи не требуется.

## Регистры общего назначения

Регистр — это небольшой (обычно 4 или 8 байт) кусочек памяти в процессоре с чрезвычайно большой скоростью доступа. Регистры делятся на регистры специального назначения и регистры общего назначения. Нас сейчас интересуют регистры общего назначения. Как можно догадаться по названию, программа может использовать эти регистры под свои нужды, как ей вздумается.

На x86 доступно восемь 32-х битных регистров общего назначения — eax, ebx, ecx, edx, esp, ebp, esi и edi. Регистры не имеют заданного наперед типа, то есть, они могут трактоваться как знаковые или беззнаковые целые числа, указатели, булевы значения, ASCII-коды символов, и так далее. Несмотря на то, что в теории эти регистры можно использовать как угодно, на практике обычно каждый регистр используется определенным образом. Так, esp указывает на вершину стека, ecx играет роль счетчика, а в eax записывается результат выполнения операции или процедуры. Существуют 16-и битные регистры ax, bx, cx, dx, sp, bp, si и di, представляющие собой 16 младших бит соответствующих 32-х битных регистров. Также доступны и 8-и битовые регистры ah, al, bh, bl, ch, cl, dh и dl, которые представляют собой старшие и младшие байты регистров ax, bx, cx и dx соответственно.

Рассмотрим пример. Допустим, выполняются следующие три инструкции:

```txt
(gdb) x/3i $pc
=> 0x8048074: mov    $0xaabbccdd,%eax
   0x8048079: mov    $0xee,%al
   0x804807b: mov    $0x1234,%ax
```

Значения регистров после записи в eax значения 0xAABBCCDD:

```txt
(gdb) p/x $eax
$1 = 0xaabbccdd
(gdb) p/x $ax
$2 = 0xccdd
(gdb) p/x $ah
$3 = 0xcc
(gdb) p/x $al
$4 = 0xdd
```

Значения после записи в регистр al значения 0xEE:

```txt
(gdb) p/x $eax
$5 = 0xaabbccee
(gdb) p/x $ax
$6 = 0xccee
(gdb) p/x $ah
$7 = 0xcc
(gdb) p/x $al
$8 = 0xee
```

Значения регистров после записи в ax числа 0x1234:

```txt
(gdb) p/x $eax
$9 = 0xaabb1234
(gdb) p/x $ax
$10 = 0x1234
(gdb) p/x $ah
$11 = 0x12
(gdb) p/x $al
$12 = 0x34
```

Как видите, ничего сложного.

*Примечание*: Синтаксис GAS позволяет явно указывать размеры операндов путем использования суффиксов b (байт), w (слово, 2 байта), l (длинное слово, 4 байта), q (четверное слово, 8 байт) и некоторых других. Например, вместо команды mov $0xEE, %al можно написать movb $0xEE, %al, вместо mov $0x1234, %ax — movw $0x1234, %ax, и так далее. В современном GAS эти суффиксы являются опциональными и я лично их не использую. Но не пугайтесь, если увидите их в чужом коде.

На x64 размер регистров был увеличен до 64-х бит. Соответствующие регистры получили название rax, rbx, и так далее. Кроме того, регистров общего назначения стало шестнадцать вместо восьми. Дополнительные регистры получили названия r8, r9, …, r15. Соответствующие им регистры, которые представляют младшие 32, 16 и 8 бит, получили название r8d, r8w, r8b, и по аналогии для регистров r9-r15. Кроме того, появились регистры, представляющие собой младшие 8 бит регистров rsi, rdi, rbp и rsp — sil, dil, bpl и spl соответственно.

## Адресация

Как уже отмечалось, регистры могут трактоваться, как указатели на данные в памяти. Для разыменования таких указателей используется специальный синтаксис:

```asm
mov  (%rsp), %rax
```

Эта запись означает «прочитай 8 байт по адресу, записанному в регистре rsp, и сохрани их в регистр rax». При запуске программы rsp указывает на вершину стека, где хранится число аргументов, переданных программе (argc), указатели на эти аргументы, а также переменные окружения и кое-какая другая информация. Таким образом, в результате выполнения приведенной выше инструкции (разумеется, при условии, что перед ней не выполнялось каких-либо других инструкций) в rax будет записано количество аргументов, с которыми была запущена программа.

В одной команде можно указывать адрес и смешение (как положительное, так и отрицательное) относительно него:

```asm
mov  8(%rsp), %rax
```

Эта запись означает «возьми rsp, прибавь к нему 8, прочитай 8 байт по получившемуся адресу и положи их в rax». Таким образом, в rax будет записан адрес строки, представляющей собой первый аргумент программы, то есть, имя исполняемого файла.

При работе с массивами бывает удобно обращаться к элементу с определенным индексом. Соответствующий синтаксис:

```asm
# инструкция xchg меняет значения местами
xchg 16(%rsp,%rcx,8), %rax
```

Читается так: «посчитай rcx умножить на 8 + rsp + 16, и поменяй местами 8 байт (размер регистра) по получившемуся адресу и значение регистра rax». Другими словами, rsp и 16 все так же играют роль смещения, rcx играет роль индекса в массиве, а 8 — это размер элемента массива. При использовании данного синтаксиса допустимыми размерами элемента являются только 1, 2, 4 и 8. Если требуется какой-то другой размер, можно использовать инструкции умножения, бинарного сдвига и прочие, которые мы рассмотрим далее.

Наконец, следующий код тоже валиден:

```asm
data
msg:
  .ascii "Hello, world!\n"
.text

.globl _start
_start:
  # обнуление rcx
  xor %rcx, %rcx
  mov msg(,%rcx,8), %al
  mov msg, %ah
```

В смысле, что можно не указывать регистр со смещением или вообще какие-либо регистры. В результате выполнения этого кода в регистры al и ah будет записан ASCII-код буквы H, или 0x48.

В этом контексте хотелось бы упомянуть еще одну полезную ассемблерную инструкцию:

```asm
# rax := rcx*8 + rax + 123
lea 123(%rax,%rcx,8), %rax
```

Инструкция lea очень удобна, так как позволяет сразу выполнить умножение и несколько сложений.

__Fun fact__! На x64 в байткоде инструкций никогда не используются 64-х битовые смещения. В отличие от x86, инструкции часто оперируют не абсолютными адресами, а адресами относительно адреса самой инструкции, что позволяет обращаться к ближайшим +/- 2 Гб оперативной памяти. Соответствующий синтаксис:

```asm
movb msg(%rip), %al
```

Сравним длины опкодов «обычного» и «относительного» mov (objdump -d):

```txt
4000b0: 8a 0c 25 e8 00 60 00  mov    0x6000e8,%cl
4000b7: 8a 05 2b 00 20 00     mov    0x20002b(%rip),%al # 0x6000e8
```

Как видите, «относительный» mov еще и на один байт короче! Что это за регистр такой rip мы узнаем чуть ниже.

Для записи же полного 64-х битового значения в регистр предусмотрена специальная инструкция:

```asm
movabs $0x1122334455667788, %rax
```

Другими словами, процессоры x64 так же экономно кодируют инструкции, как и процессоры x86, и в наше время нет особо смысла использовать процессоры x86 в системах, имеющих пару гигабайт оперативной памяти или меньше (мобильные устройства, холодильники, микроволновки, и так далее). Скорее всего, процессоры x64 будут даже более эффективны за счет большего числа доступных регистров и большего размера этих регистров.

## Арифметические операции

Рассмотрим основные арифметические операции:

```asm
# инциализируем значения регистров
mov  $123, %rax
mov  $456, %rcx

# инкремент: rax = rax + 1 = 124
inc  %rax

# декремент: rax = rax - 1 = 123
dec  %rax

# сложение: rax = rax + rcx = 579
add  %rcx, %rax

# вычитание: rax = rax - rcx = 123
sub  %rcx, %rax

# изменение знака: rcx = - rcx = -456
neg  %rcx
```

Здесь и далее операндами могут быть не только регистры, но и участки памяти или константы. Но оба операнда не могут быть участками памяти. Это правило применимо ко всем инструкциям ассемблера x86/x64, по крайней мере, из рассмотренных в данной статье.

Пример умножения:

```asm
mov $100, %al
mov $3, %cl
mul %cl
```

В данном примере инструкция mul умножает al на cl, и сохраняет результат умножения в пару регистров al и ah. Таким образом, ax примет значение 0x12C или 300 в десятичной нотации. В худшем случае для сохранения результата перемножения двух N-байтовых значений может потребоваться до 2N байт. В зависимости от размера операнда результат сохраняется в al:ah, ax:dx, eax:edx или rax:rdx. Притом в качестве множителей всегда используется первый из этих регистров и переданный инструкции аргумент.

Знаковое умножение производится точно так же при помощи инструкции imul. Кроме того, существуют варианты imul с двумя и тремя аргументами:

```asm
mov  $123, %rax
mov  $456, %rcx

# rax = rax * rcx = 56088
imul %rcx, %rax

# rcx = rax * 10 = 560880
imul $10, %rax, %rcx
```

Инструкции div и idiv производят действия, обратные mul и imul. Например:

```asm
mov  $0,   %rdx
mov  $456, %rax
mov  $123, %rcx

# rax = rdx:rax / rcx = 3
# rdx = rdx:rax % rcx = 87
div  %rcx
```

Как видите, был получен результат целочисленного деления, а также остаток от деления.

Это далеко не все арифметические инструкции. Например, есть еще adc (сложение с учетом флага переноса), sbb (вычитание с учетом займа), а также соответствующие им инструкции, выставляющие и очищающие соответствующие флаги (ctc, clc), и многие другие. Но они распространены намного меньше, и потому в рамках данной статьи не рассматриваются.

## Логические и битовые операции

Как уже отмечалось, особой типизации в ассемблере x86/x64 не предусмотрено. Поэтому не стоит удивляться, что в нем нет отдельных инструкций для выполнения булевых операций и отдельных для выполнения битовых операций. Вместо этого есть один набор инструкций, работающих с битами, а уж как интерпретировать результат — решает конкретная программа.

Так, например, выглядит вычисление простейшего логического выражения:

```asm
mov  $0, %rax # a = false
mov  $1, %rbx # b = true
mov  $0, %rcx # c = false

# rdx := a || !(b && c)
mov  %rcx, %rdx  # rdx = c
and  %rbx, %rdx  # rdx &= b
not  %rdx        # rdx = ~ rdx
or   %rax, %rdx  # rdx |= a
and  $1,   %rdx  # rdx &= 1
```

Заметьте, что здесь мы использовали по одному младшему биту в каждом из 64-х битовых регистров. Таким образом, в старших битах образуется мусор, который мы обнуляем последней командой.

Еще одна полезная инструкция — это xor (исключающее или). В логических выражениях xor используется нечасто, однако с его помощью часто происходит обнуление регистров. Если посмотреть на опкоды инструкций, то становится понятно, почему:

```txt
  4000b3: 48 31 db              xor    %rbx,%rbx
  4000b6: 48 ff c3              inc    %rbx
  4000b9: 48 c7 c3 01 00 00 00  mov    $0x1,%rbx
```

Как видите, инструкции xor и inc кодируются всего лишь тремя байтами каждая, в то время, как делающая то же самое инструкция mov занимает целых семь байт. Каждый отдельный случай, конечно, лучше бенчмаркать отдельно, но общее эвристическое правило такое — чем короче код, тем больше его помещается в кэши процессора, тем быстрее он работает.

В данном контексте также следует вспомнить инструкции побитового сдвига, тестирования битов (bit test) и сканирования битов (bit scan):

```asm
# положим что-нибудь в регистр
movabs $0xc0de1c0ffee2beef, %rax

# сдвиг влево на 3 бита
# rax = 0x0de1c0ffee2beef0
shl $4,  %rax

# сдвиг вправо на 7 бит
# rax = 0x001bc381ffdc57dd
shr $7,  %rax

# циклический сдвиг вправо на 5 бит
# rax = 0xe800de1c0ffee2be
ror $5,  %rax

# циклический сдвиг влево на 5 бит
# rax = 0x001bc381ffdc57dd
rol $5,  %rax

# положить в CF (см далее) значение 13-го бита
# CF = !!(0x1bc381ffdc57dd & (1 << 13)) = 0
bt  $13, %rax

# то же самое + установить бит (bit test and set)
# rax = 0x001bc381ffdc77dd, CF = 0
bts $13, %rax

# то же самое + сбросить бит (bit test and reset)
# rax = 0x001bc381ffdc57dd, CF = 1
btr $13, %rax

# то же самое + инвертировать бит (bit test and complement)
# rax = 0x001bc381ffdc77dd, CF = 0
btc $13, %rax

# найти самый младший ненулевой байт (bit scan forward)
# rcx = 0, ZF = 0
bsf %rax, %rcx

# найти самый старший ненулевой байт (bit scan reverse)
# rdx = 52, ZF = 0
bsr %rax, %rdx

# если все биты нулевые, ZF = 1, значение rdx неопределено
xor %rax, %rax
bsf %rax, %rdx
```

Еще есть битовые сдвиги со знаком (sal, sar), циклические сдвиги с флагом переноса (rcl, rcr), а также сдвиги двойной точности (shld, shrd). Но используются они не так уж часто, да и утомишься перечислять вообще все инструкции. Поэтому их изучение я оставляю вам в качестве домашнего задания.

## Условные выражения и циклы

Выше несколько раз упоминались какие-то там флаги, например, флаг переноса. Под флагами понимаются биты специального регистра eflags / rflags (название на x86 и x64 соответственно). Напрямую обращаться к этому регистру при помощи инструкций mov, add и подобных нельзя, но он изменяется и используется различными инструкциями косвенно. Например, уже упомянутый флаг переноса (carry flag, CF) хранится в нулевом бите eflags / rflags и используется, например, в той же инструкции bt. Еще из часто используемых флагов можно назвать zero flag (ZF, 6-ой бит), sign flag (SF, 7-ой бит), direction flag (DF, 10-ый бит) и overflow flag (OF, 11-ый бит).

Еще из таких неявных регистров следует назвать eip / rip, хранящий адрес текущей инструкции. К нему также нельзя обращаться напрямую, но он виден в GDB вместе с eflags / rflags, если сказать info registers, и косвенно изменяется всеми инструкциям. Большинство инструкций просто увеличивают eip / rip на длину этой инструкции, но есть и исключения из этого правила. Например, инструкция jmp просто осуществляет переход по заданному адресу:

```asm
  # обнуляем rax
  xor  %rax, %rax
  jmp next
  # эта инструкция будет пропущена
  inc  %rax
next:
  inc  %rax
```

В результате значение rax будет равно единице, так как первая инструкция inс будет пропущена. Заметьте, что адрес перехода также может быть записан в регистре:

```asm
  xor %rax, %rax
  mov $next, %rcx
  jmp *%rcx
  inc %rax
next:
  inc %rax
```

Впрочем, на практике такого кода лучше избегать, так как он ломает предсказание переходов и потому менее эффективен.

*Примечание*: GAS позволяет давать меткам цифирные имена типа 1:, 2:, и так далее, и переходить к ближайшей предыдущей или следующей метке с заданным номером инструкциями вроде jmp 1b и jmp 1f. Это довольно удобно, так как иногда бывает трудно придумать меткам осмысленные имена. Подробности можно найти здесь.

Условные переходы обычно осуществляются при помощи инструкции cmp, которая сравнивает два своих операнда и выставляет соответствующие флаги, за которой следует инструкция из семейства je, jg и подобных:

```asm
  cmp %rax, %rcx

  je  1f # перейти, если равны (equal)
  jl  1f # перейти, если знаково меньше (less)
  jb  1f # перейти, если беззнаково меньше (below)
  jg  1f # перейти, если знаково больше (greater)
  ja  1f # перейти, если беззнаково больше (above)

1:
```

Существует также инструкции jne (перейти, если не равны), jle (перейти, если знаково меньше или равны), jna (перейти, если беззнаково не больше) и подобные. Принцип их именования, надеюсь, очевиден. Вместо je / jne часто пишут jz / jnz, так как инструкции je / jne просто проверяют значение ZF. Также есть инструкции, проверяющие другие флаги — js, jo и jp, но на практике они используются редко. Все эти инструкции вместе взятые обычно называют jcc. То есть, вместо конкретных условий пишутся две буквы «c», от «condition». Здесь можно найти хорошую сводную таблицу по всем инструкциям jcc и тому, какие флаги они проверяют.

Помимо cmp также часто используют инструкцию test:

```asm
  test %rax, %rax
  jz 1f # перейти, если rax == 0
  js 2f # перейти, если rax < 0
1:
  # какой-то код
2:
  # какой-то еще код
```

__Fun fact!__ Интересно, что cmp и test в душе являются теми же sub и and, только не изменяют своих операндов. Это знание можно использовать для одновременного выполнения sub или and и условного перехода, без дополнительных инструкций cmp или test.

Еще из инструкций, связанных с условными переходами, можно отметить следующие.

```asm
jrcxz  1f
  # какой-то код
1:
```

Инструкция jrcxz осуществляет переход только в том случае, если значение регистра rcx равно нулю.

```asm
cmovge %rcx, %rax
```

Инструкции семейства cmovcc (conditional move) работают как mov, но только при выполнении заданного условия, по аналогии с jcc.

```asm
setnz %al
```

Инструкции setcc присваивают однобайтовому регистру или байту в памяти значение 1, если заданное условие выполняется, и 0 иначе.

```asm
cmpxchg %rcx, (%rdx)
```

Сравнить rax с заданным куском памяти. Если равны, выставить ZF и сохранить по указанному адресу значение указанного регистра, в данном примере rcx. Иначе очистить ZF и загрузить значение из памяти в rax. Также оба операнда могут быть регистрами.

```asm
cmpxchg8b (%rsi)
cmpxchg16b (%rsi)
```

Инструкция cmpxchg8b главным образом нужна в x86. Она работает аналогично cmpxchg, только производит compare and swap сразу 8-и байт. Регистры edx:eax используются для сравнения, а регистры ecx:ebx хранят то, что мы хотим записать. Инструкция cmpxchg16b по тому же принципу производит compare and swap сразу 16-и байт на x64.

__Важно!__ Примите во внимание, что без префикса lock все эти compare and swap инструкции не атомарны.

```asm
mov $10, %rcx
1:
# какой-то код
  loop   1b
# loopz  1b
# loopnz 1b
```

Инструкция loop уменьшает значение регистра rcx на единицу, и если после этого rcx != 0, осуществляет переход на заданную метку. Инструкции loopz и loopnz работают аналогично, только условия более сложные — (rcx != 0) && (ZF == 1) и (rcx != 0) && (ZF == 0) соответственно.

Не нужно быть семи пядей во лбу, чтобы изобразить при помощи этих инструкций конструкцию if-then-else или циклы for / while, поэтому двигаемся дальше.

## Операции со строками

Рассмотрим следующий кусок кода:

```asm
mov $str1, %rsi
mov $str2, %edi
cld
cmpsb
```

В регистры rsi и rdi кладутся адреса двух строк. Командой cld очищается флаг направления (DF). Инструкция, выполняющая обратное действие, называется std. Затем в дело вступает инструкция cmpsb. Она сравнивает байты (%rsi) и (%rdi) и выставляет флаги в соответствии с результатом сравнения. Затем, если DF = 0, rsi и rdi увеличиваются на единицу (количество байт в том, что мы сравнивали), иначе — уменьшаются. Аналогичные инструкции cmpsw, cmpsl и cmpsq сравнивают слова, длинные слова и четверные слова соответственно.

Инструкции cmps интересны тем, что могут использоваться с префиксом rep, repe (repz) и repne (repnz). Например:

```asm
mov $str1, %rsi
mov $str2, %edi
mov $len,  %rcx
cld
repe cmpsb
jne not_equal
```

Префикс rep повторяет инструкцию заданное в регистре rcx количество раз. Префиксы repz и repnz делают то же самое, но только после каждого выполнения инструкции дополнительно проверяется ZF. Цикл прерывается, если ZF = 0 в случае c repz и если ZF = 1 в случае с repnz. Таким образом, приведенный выше код проверяет равенство двух буферов одинакового размера.

Аналогичные инструкции movs перекладывает данные из буфера, адрес которого указан в rsi, в буфер, адрес которого указан в rdi (легко запомнить — rsi значит source, rdi значит destination). Инструкции stos заполняет буфер по адресу из регистра rdi байтами из регистра rax (или eax, или ax, или al, в зависимости от конкретной инструкции). Инструкции lods делают обратное действие — копируют байты по указанному в rsi адресу в регистр rax. Наконец, инструкции scas ищут байты из регистра rax (или соответствующих регистров меньшего размера) в буфере, адрес которого указан в rdi. Как и cmps, все эти инструкции работают с префиксами rep, repz и repnz.

На базе этих инструкций легко реализуются процедуры memcmp, memcpy, strcmp и подобные. Интересно, что, например, для обнуления памяти инженеры Intel рекомендуют использовать на современных процессорах rep stosb, то есть, обнулять побайтово, а не, скажем, четверными словами.

## Работа со стеком и процедуры

Со стеком все очень просто. Инструкция push кладет свой аргумент на стек, а инструкция pop извлекает значение со стека. Например, если временно забыть про инструкцию xchg, то поменять местами значение двух регистров можно так:

```asm
push %rax
mov %rcx, %rax
pop %rcx
```

Существуют инструкции, помещающие на стек и извлекающие с него регистр rflags / eflags:

```asm
pushf
# делаем что-то, что меняет флаги
popf
# флаги восстановлены, самое время сделать jcc
```

А так, к примеру, можно получить значение флага CF:

```asm
pushf
pop %rax
and $1, %rax
```

На x86 также существуют инструкции pusha и popa, сохраняющие на стеке и восстанавливающие с него значения всех регистров. В x64 этих инструкций больше нет. Видимо, потому что регистров стало больше и сами регистры теперь длиннее — сохранять и восстанавливать их все стало сильно дороже.

Процедуры, как правило, «создаются» при помощи инструкций call и ret. Инструкция call кладет на стек адрес следующей инструкции и передает управление по указанному в аргументе адресу. Инструкция ret читает со стека адрес возврата и передает по нему управление. Например:

```asm
someproc:
  # типичный пролог процедуры
  # для примера выделяем 0x10 байт на стеке под локальные переменные
  # rbp - указатель на фрейм стека
  push %rbp
  mov %rsp, %rbp
  sub $0x10, %rsp

  # тут типа какие-то вычисления ...
  mov $1, %rax

  # типичный эпилог процедуры
  add $0x10, %rsp
  pop %rbp

  # выход из процедуры
  ret

_start:
  # как и в случае с jmp, адрес перехода может быть в регистре
  call someproc
  test %rax, %rax
  jnz error
```

*Примечание*: Аналогичный пролог и эпилог можно написать при помощи инструкций enter $0x10, $0 и leave. Но в наше время эти инструкции используются редко, так как они выполняются медленнее из-за дополнительной поддержки вложенных процедур.

Как правило, возвращаемое значение передается в регистре rax или, если его размера не достаточно, записывается в структуру, адрес которой передается в качестве аргумента. К вопросу о передаче аргументов. Соглашений о вызовах существует великое множество. В одних все аргументы всегда передаются через стек (отдельный вопрос — в каком порядке) и за очистку стека от аргументов отвечает сама процедура, в других часть аргументов передается через регистры, а часть через стек, и за очистку стека от аргументов отвечает вызывающая сторона, плюс множество вариантов посередине, с отдельными правилами касательно выравнивания аргументов на стеке, передачи this, если это ООП язык, и так далее. В общем случае для произвольно взятой архитектуры, компилятора и языка программирования соглашение о вызовах может быть вообще каким угодно.

Для примера рассмотрим ассемблерный код, сгенерированный CLang 3.8 для простой программки на языке C под x64. Так выглядит одна из процедур:

```c
unsigned int
hash(const unsigned char *data, const size_t data_len) {
  unsigned int hash = 0x4841434B;
  for(int i = 0; i < data_len; i++) {
    hash = ((hash << 5) + hash) + data[i];
  }
  return hash;
}
```

Дизассемблерный листинг (при компиляции с -O0, комментарии мои):

```txt
# типичный пролог процедуры
# регистр rsp не изменяется, так как процедура не вызывает никаких
# других процедур
  400950: 55                    push   %rbp
  400951: 48 89 e5              mov    %rsp,%rbp

# инициализация локальных переменных:
# -0x08(%rbp) - const unsigned char *data (8 байт)
# -0x10(%rbp) - const size_t data_len (8 байт)
# -0x14(%rbp) - unsigned int hash (4 байта)
# -0x18(%rbp) - int i (4 байта)
  400954: 48 89 7d f8           mov    %rdi,-0x8(%rbp)
  400958: 48 89 75 f0           mov    %rsi,-0x10(%rbp)
  40095c: c7 45 ec 4b 43 41 48  movl   $0x4841434b,-0x14(%rbp)
  400963: c7 45 e8 00 00 00 00  movl   $0x0,-0x18(%rbp)

# rax := i. если достигли data_len, выходим из цикла
  40096a: 48 63 45 e8           movslq -0x18(%rbp),%rax
  40096e: 48 3b 45 f0           cmp    -0x10(%rbp),%rax
  400972: 0f 83 28 00 00 00     jae    4009a0 <hash+0x50>

# eax := (hash << 5) + hash
  400978: 8b 45 ec              mov    -0x14(%rbp),%eax
  40097b: c1 e0 05              shl    $0x5,%eax
  40097e: 03 45 ec              add    -0x14(%rbp),%eax

# eax += data[i]
  400981: 48 63 4d e8           movslq -0x18(%rbp),%rcx
  400985: 48 8b 55 f8           mov    -0x8(%rbp),%rdx
  400989: 0f b6 34 0a           movzbl (%rdx,%rcx,1),%esi
  40098d: 01 f0                 add    %esi,%eax

# hash := eax
  40098f: 89 45 ec              mov    %eax,-0x14(%rbp)

# i++ и перейти к началу цикла
  400992: 8b 45 e8              mov    -0x18(%rbp),%eax
  400995: 83 c0 01              add    $0x1,%eax
  400998: 89 45 e8              mov    %eax,-0x18(%rbp)
  40099b: e9 ca ff ff ff        jmpq   40096a <hash+0x1a>

# возвращаемое значение (hash) кладется в регистр eax
  4009a0: 8b 45 ec              mov    -0x14(%rbp),%eax

# типичный эпилог
  4009a3: 5d                    pop    %rbp
  4009a4: c3                    retq
```

Здесь мы встретили две новые инструкции — movs и movz. Они работают точно так же, как mov, только расширяют один операнд до размера второго, знаково и беззнаково соответственно. Например, инструкция movzbl (%rdx,%rcx,1),%esi читайт байт (b) по адресу (%rdx,%rcx,1), расширяет его в длинное слово (l) путем добавления в начало нулей (z) и кладет результат в регистр esi.

Как видите, два аргумента были переданы процедуре через регистры rdi и rsi. По всей видимости, используется конвенция под названием System V AMD64 ABI. Утверждается, что это стандарт де-факто под x64 на unix.

## Заключение
Само собой разумеется, в рамках одной статьи, описать весь ассемблер x86/x64 не представляется возможным (более того, я не уверен, что сам знаю его прямо таки весь). Как минимум, за кадром остались такие темы, как операции над числами с плавающей точкой, MMX-, SSE- и AVX-инструкции, а также всякие экзотические инструкции вроде lidt, lgdt, bswap, rdtsc, cpuid, movbe, xlatb, или prefetch. Я постараюсь осветить их в следующих статьях, но ничего не обещаю. Следует также отметить, что в выводе objdump -d для большинства реальных программ вы очень редко увидите что-то помимо описанного выше.

Еще интересный топик, оставшийся за кадром — это атомарные операции, барьеры памяти, спинлоки и вот это все. Например, compare and swap часто реализуется просто как инструкция cmpxchg с префиксом lock. По аналогии реализуется атомарный инкремент, декремент, и прочее. Увы, все это тянет на тему для отдельной статьи.

В качестве источников дополнительной информации можно рекомендовать книгу Modern X86 Assembly Language Programming, и, конечно же, мануалы от Intel. Также довольно неплоха книга x86 Assembly на wikibooks.org.

Из онлайн-справочников по ассемблерным инструкциям стоит обратить внимание на следующие:

 + [x86 ASM Refernces](http://ref.x86asm.net/)
 + [Felix Cloutier x86](http://www.felixcloutier.com/x86/)
 + [x86 renejeschke](http://x86.renejeschke.de/)
 + [EN Wiki x86 instructions](https://en.wikipedia.org/wiki/X86_instruction_listings)
