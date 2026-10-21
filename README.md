# linux-rdtsc

Ядро Linux 7.2.5 с патчем KVM, который прячет от Windows-гостя главный
«временной» признак виртуалки: скачок RDTSC при VM-exit.

Для intel на Amd проверок не было

Пара к [qemu-stealth](https://github.com/NeXtagG/qemu-stealth): QEMU
закрывает строки, SMBIOS и ACPI, а этот признак живёт в KVM, то есть в ядре.

## Что делает патч

Детекторы (pafish `rdtsc_diff_vmexit`, al-khaser) замеряют RDTSC до и после
CPUID: в виртуалке CPUID вызывает VM-exit, и между замерами набегают тысячи
тактов. Патч (`arch/x86/kvm/vmx/vmx.c`, `vmx.h`):

- включает **RDTSC exiting** — каждое RDTSC/RDTSCP гостя выходит в KVM;
- `handle_rdtsc()` / `handle_rdtscp()` отдают TSC, который идёт 1:1 с хостом,
  но с каждого шага срезано `VMEXIT_COST` тактов (по умолчанию 2500). Скачок
  исчезает, а гость всё равно видит настоящую частоту;
- обрабатывает RDTSCP (exit reason 51) — без этого гость зависает с
  «unexpected exit reason 0x33».

Только Intel (VMX). На AMD (SVM) патч ничего не меняет.

**Цена:** каждое чтение TSC в госте — это VM-exit, так что игры и программы,
которые часто читают время, работают немного медленнее.

**Ограничения:** состояние фальшивого TSC — общие глобальные переменные без
блокировок, одни на все vCPU и все виртуалки. Патч рассчитан на одну VM за раз:
при нескольких одновременно запущенных VM их чтения TSC влияют друг на друга,
а смещение и масштабирование TSC из KVM (`tsc_offset`/`tsc_scaling`)
не учитываются. Режим включается для всех гостей сразу, отдельно для одной
VM его не выключить.

## Сборка (Arch)

```bash
git clone https://github.com/NeXtagG/linux-rdtsc && cd linux-rdtsc
makepkg -si                         # скачает 7.2.5 с kernel.org, наложит патч, соберёт
VMEXIT_COST=3000 makepkg -si        # срезать больше (1500-4000): прячет лучше, но
                                    # при плотной калибровке частота в госте ниже
```

Если в `PATH` раньше системного стоит `rustup` (`~/.cargo/bin`), ядро соберётся
без Rust: у rustup обычно нет `rust-src`. Тогда запускай
`PATH=/usr/bin:$PATH makepkg -si` или поставь `rustup component add rust-src`.

`config` — полный конфиг ядра (на базе дистрибутивного, `CONFIG_LOCALVERSION="-rdtsc"`).
После установки добавь `linux-rdtsc` в загрузчик (на Limine/Omarchy —
`BOOT_ORDER` в `/etc/limine-entry-tool.d/`) и в `IgnorePkg` в
`/etc/pacman.conf`, чтобы его не заменило одноимённым пакетом из AUR.

Проверка после загрузки:

```bash
uname -r                                  # 7.2.5-rdtsc
sudo dmesg | grep handle_rdtsc            # появится после старта первой VM
```

Другие дистрибутивы: распакуй `linux-7.2.5.tar.xz`, `patch -p1 < rdtsc.patch`,
`cp config .config && make olddefconfig && make -j$(nproc)` и установи как обычно.

## Лицензия

GPL-2.0, как у ядра Linux (`LICENSE`): можно свободно использовать и
изменять, производные работы тоже должны оставаться открытыми.
