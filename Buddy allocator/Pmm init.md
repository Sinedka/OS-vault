1. firmware memory map
2. memblock allocator
3. struct page initialization
4. zone initialization
5. buddy free lists setup


## 1. firmware memory map

Получает memory map от bios/efi

## 2. memblock allocator

Очень простой early boot allocator.
Работает до включения normal memory manager
``` C
struct memblock {
    memory regions
    reserved regions
}
```


