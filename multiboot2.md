## Что такое Multiboot2

**Multiboot2** — это спецификация загрузочного интерфейса между bootloader’ом и ядром ОС.
Она была разработана проектом GNU Project как развитие Multiboot v1 для современных систем.

Главная идея:

> bootloader загружает ядро и передаёт ему стандартизированную информацию о системе.

Это позволяет писать собственную ОС без необходимости создавать свой bootloader с нуля.

На практике чаще всего используется:

* GRUB
* иногда Limine
* реже собственные загрузчики

---

# Зачем нужен Multiboot2

Без Multiboot2 ядро должно самостоятельно:

* переключать CPU режимы
* читать файловую систему
* находить ядро на диске
* загружать модули
* работать с BIOS/UEFI

С Multiboot2 всё это делает bootloader.

Ядро получает:

* карту памяти
* framebuffer
* ACPI таблицы
* ELF sections
* модули
* cmdline
* информацию о CPU/BIOS/UEFI

---

# Архитектура загрузки

Схема:

```text
BIOS/UEFI
    ↓
Bootloader (GRUB)
    ↓
Multiboot2 protocol
    ↓
Kernel entry point
```

---

# Отличия Multiboot v1 и v2

| Возможность     | Multiboot1 | Multiboot2 |
| --------------- | ---------- | ---------- |
| Гибкие теги     | Нет        | Да         |
| Framebuffer     | Ограничен  | Полный     |
| EFI поддержка   | Плохая     | Нормальная |
| Расширяемость   | Низкая     | Высокая    |
| 64-bit friendly | Частично   | Да         |

---

# Основные концепции

Multiboot2 состоит из двух частей:

1. Header внутри ядра
2. Boot Information Structure

---

# 1. Multiboot2 Header

Bootloader ищет специальный заголовок внутри бинарника ядра.

Он должен находиться:

* в первых 32 KiB файла
* выровнен по 8 байт

Структура:

```c
struct multiboot_header {
    uint32_t magic;
    uint32_t architecture;
    uint32_t header_length;
    uint32_t checksum;
};
```

---

# Magic Number

Всегда:

```text
0xE85250D6
```

---

# Checksum

Формула:

```text
magic + architecture + header_length + checksum == 0
```

---

# Пример header

```asm
section .multiboot
align 8

header_start:

dd 0xE85250D6
dd 0
dd header_end - header_start
dd -(0xE85250D6 + 0 + (header_end - header_start))

dw 0
dw 0
dd 8

header_end:
```

---

# Разбор

```asm
dd 0xE85250D6
```

magic

```asm
dd 0
```

architecture:

* 0 = i386

---

```asm
dd header_end - header_start
```

размер header

---

```asm
dd -(...)
```

checksum

---

```asm
dw 0
dw 0
dd 8
```

end tag

---

# Header Tags

Multiboot2 использует систему тегов.

Каждый tag:

```c
struct multiboot_tag {
    uint16_t type;
    uint16_t flags;
    uint32_t size;
};
```

---

# Важные header tags

| Tag                 | Назначение        |
| ------------------- | ----------------- |
| Information request | Запрос boot info  |
| Address tag         | Физические адреса |
| Entry address       | Entry point       |
| Console flags       | Console режим     |
| Framebuffer         | Видео режим       |
| EFI tags            | EFI boot          |

---

# Framebuffer Tag

Пример:

```asm
dw 5
dw 0
dd 20

dd 1024
dd 768
dd 32
```

Просим:

* 1024x768
* 32 bpp

---

# Entry Point Tag

```asm
dw 3
dw 0
dd 12

dd start
```

---

# Boot Information Structure

После загрузки bootloader передаёт:

```text
EAX = 0x36D76289
EBX = pointer to multiboot info
```

Magic число:

```text
0x36D76289
```

---

# Проверка magic

```c
if (magic != 0x36D76289) {
    panic();
}
```

---

# Структура boot info

```c
struct multiboot_info {
    uint32_t total_size;
    uint32_t reserved;
    struct multiboot_tag first_tag;
};
```

Далее идут теги подряд.

---

# Парсинг тегов

Пример:

```c
struct multiboot_tag* tag;

for (
    tag = (struct multiboot_tag*)(mbi + 8);
    tag->type != 0;
    tag = (struct multiboot_tag*)
        ((uint8_t*)tag + ((tag->size + 7) & ~7))
) {

}
```

---

# Выравнивание

Все теги:

* aligned to 8 bytes

Формула:

```c
(tag->size + 7) & ~7
```

---

# Основные boot info tags

| Tag              | Назначение         |
| ---------------- | ------------------ |
| CMDLINE          | Аргументы ядра     |
| BOOT_LOADER_NAME | Имя bootloader     |
| MODULE           | Загруженные модули |
| BASIC_MEMINFO    | Память             |
| MMAP             | Memory map         |
| FRAMEBUFFER      | Видео              |
| ELF_SECTIONS     | ELF данные         |
| ACPI             | ACPI таблицы       |
| EFI              | EFI info           |

---

# Memory Map

Один из важнейших тегов.

Позволяет узнать:

* свободную RAM
* reserved regions
* ACPI memory
* MMIO

---

# Структура mmap

```c
struct multiboot_mmap_entry {
    uint64_t addr;
    uint64_t len;
    uint32_t type;
    uint32_t zero;
};
```

---

# Типы памяти

| Type | Значение         |
| ---- | ---------------- |
| 1    | Available        |
| 2    | Reserved         |
| 3    | ACPI reclaimable |
| 4    | NVS              |
| 5    | Bad RAM          |

---

# Пример чтения mmap

```c
if (tag->type == MULTIBOOT_TAG_TYPE_MMAP) {

    struct multiboot_tag_mmap* mmap_tag =
        (struct multiboot_tag_mmap*)tag;

    for (
        entry = mmap_tag->entries;
        (uint8_t*)entry <
            (uint8_t*)tag + tag->size;
        entry =
            (multiboot_memory_map_t*)
            ((unsigned long)entry
            + mmap_tag->entry_size)
    ) {

    }
}
```

---

# Framebuffer Info

Позволяет писать графику без VGA.

Структура:

```c
struct multiboot_tag_framebuffer {
    uint64_t framebuffer_addr;
    uint32_t framebuffer_pitch;
    uint32_t framebuffer_width;
    uint32_t framebuffer_height;
    uint8_t framebuffer_bpp;
};
```

---

# Пример рисования пикселя

```c
uint32_t* fb = framebuffer_addr;

fb[y * width + x] = 0xFF0000;
```

---

# ELF Sections Tag

Нужен для:

* symbol lookup
* kernel debugging
* stack traces

Очень полезен для собственного kernel panic.

---

# Modules

Bootloader может загружать дополнительные файлы:

* initrd
* ramdisk
* драйверы
* userspace

---

# Пример GRUB config

```cfg
menuentry "MyOS" {
    multiboot2 /boot/kernel.bin
    module2 /boot/initrd.img
    boot
}
```

---

# Получение module

```c
struct multiboot_tag_module {
    uint32_t type;
    uint32_t size;
    uint32_t mod_start;
    uint32_t mod_end;
    char cmdline[];
};
```

---

# BIOS vs UEFI

Multiboot2 поддерживает оба режима.

---

# BIOS boot

Традиционный:

```text
BIOS → GRUB → kernel
```

---

# UEFI boot

Современный:

```text
UEFI → GRUB EFI → kernel
```

---

# EFI Tags

В UEFI доступны:

* EFI memory map
* system table
* image handle

---

# Long Mode и x86_64

ВАЖНО:

Multiboot2 НЕ переключает CPU автоматически в long mode.

GRUB обычно запускает ядро:

* в protected mode
* 32-bit

Ядро само включает:

* PAE
* paging
* long mode

---

# Включение Long Mode

Основные шаги:

1. Enable A20
2. Setup GDT
3. Enable PAE
4. Build page tables
5. Set EFER.LME
6. Enable paging
7. Far jump в 64-bit code

---

# Схема перехода

```text
Real Mode
    ↓
Protected Mode
    ↓
Long Mode
```

---

# Минимальное ядро

## boot.asm

```asm
global start
extern kernel_main

section .text

start:
    cli

    push ebx
    push eax

    call kernel_main

.hang:
    hlt
    jmp .hang
```

---

# kernel.c

```c
#include <stdint.h>

void kernel_main(
    uint32_t magic,
    uint32_t addr
) {

    volatile char* vga =
        (volatile char*)0xB8000;

    vga[0] = 'O';
    vga[1] = 0x0F;
}
```

---

# Linker Script

```ld
ENTRY(start)

SECTIONS
{
    . = 1M;

    .multiboot :
    {
        *(.multiboot)
    }

    .text :
    {
        *(.text)
    }

    .rodata :
    {
        *(.rodata)
    }

    .data :
    {
        *(.data)
    }

    .bss :
    {
        *(.bss)
    }
}
```

---

# Компиляция

## NASM

NASM

```bash
nasm -f elf32 boot.asm -o boot.o
```

---

## GCC

GCC

```bash
gcc -m32 -ffreestanding -c kernel.c -o kernel.o
```

---

## Link

```bash
ld -m elf_i386 \
   -T linker.ld \
   boot.o kernel.o \
   -o kernel.bin
```

---

# Проверка Multiboot2

Утилита:

```bash
grub-file --is-x86-multiboot2 kernel.bin
```

---

# Создание ISO

Структура:

```text
iso/
 └── boot/
     ├── grub/
     │   └── grub.cfg
     └── kernel.bin
```

---

# grub.cfg

```cfg
set timeout=0
set default=0

menuentry "MyOS" {
    multiboot2 /boot/kernel.bin
    boot
}
```

---

# Генерация ISO

```bash
grub-mkrescue -o myos.iso iso/
```

---

# Запуск в QEMU

QEMU

```bash
qemu-system-i386 -cdrom myos.iso
```

---

# Для x86_64

```bash
qemu-system-x86_64 -cdrom myos.iso
```

---

# Отладка

Очень полезно:

## GDB

GDB

```bash
qemu-system-i386 \
    -s -S \
    -cdrom myos.iso
```

Подключение:

```bash
gdb kernel.bin
```

```gdb
target remote localhost:1234
```

---

# Частые ошибки

## 1. Header не найден

Причины:

* не в первых 32 KiB
* нет align 8
* неверный checksum

---

# 2. Triple Fault

Обычно:

* плохой GDT
* invalid page tables
* stack corruption

---

# 3. GRUB Error: no multiboot header

Проверь:

```bash
grub-file --is-x86-multiboot2 kernel.bin
```

---

# 4. Черный экран

Причины:

* kernel crashed
* interrupts
* framebuffer issue

---

# Рекомендуемая структура проекта

```text
project/
├── boot/
│   ├── boot.asm
│   └── linker.ld
├── kernel/
│   ├── kernel.c
│   ├── memory.c
│   ├── paging.c
│   └── interrupts.c
├── drivers/
├── libc/
├── iso/
└── Makefile
```

---

# Полезные возможности Multiboot2

## Information Request Tag

Можно запросить только нужные теги.

---

# ACPI Support

Для:

* SMP
* power management
* APIC

---

# EFI Runtime Services

Позволяют работать с UEFI runtime.

---

# ELF Symbols

Можно реализовать:

* kernel panic trace
* symbol resolver
* dynamic loader

---

# Limine vs Multiboot2

Сегодня многие OSDev разработчики используют:

Limine

Почему:

* проще long mode
* лучше UEFI
* проще framebuffer
* чище API

Но Multiboot2 всё ещё очень важен:

* совместимость с GRUB
* огромное количество документации
* стандарт де-факто

---

# Что изучать дальше

После Multiboot2 обычно изучают:

1. GDT
2. IDT
3. Paging
4. Physical memory manager
5. Virtual memory
6. Interrupts
7. Scheduler
8. Syscalls
9. ELF loader
10. Filesystems

---

# Лучшие ресурсы

## OSDev Wiki

[OSDev Wiki](https://wiki.osdev.org/Multiboot?utm_source=chatgpt.com)

---

## GNU GRUB manual

[GNU GRUB Documentation](https://www.gnu.org/software/grub/manual/multiboot2/multiboot.html?utm_source=chatgpt.com)

---

## Intel SDM

[Intel Software Developer Manuals](https://www.intel.com/content/www/us/en/developer/articles/technical/intel-sdm.html?utm_source=chatgpt.com)

---

# Краткое резюме

Multiboot2 даёт ядру:

* стандартизированную загрузку
* memory map
* framebuffer
* модули
* EFI/BIOS abstraction

Минимальный pipeline:

```text
GRUB
  ↓
Multiboot2 Header
  ↓
Kernel Entry
  ↓
Parse Tags
  ↓
Init Memory
  ↓
Enable Paging
  ↓
Long Mode
```

Именно с этого начинается почти любая современная hobby OS.