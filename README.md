# landlock

Bindingi do Linuksowego [Landlocka](https://docs.kernel.org/userspace-api/landlock.html)
(LSM do dobrowolnego sandboxowania procesu) dla języka **H#**.

**Cała biblioteka to czyste pliki `.h#`.** Zero własnego kodu C/Rust w tym
repozytorium — każde wywołanie systemowe idzie przez `extern static [c]`
prosto do `libc`, która i tak jest już w każdym systemie Linux (dokładnie ten
sam wzorzec, co `extern dynamic [c] fn strlen(...)` w H#'owym
`examples/showcase.h#`).

## Co to jest Landlock

Landlock pozwala procesowi **dobrowolnie** ograniczyć własny dostęp do
plików i sieci, bez potrzeby bycia rootem ani CAP_SYS_ADMIN. Typowe
zastosowania: piaskownica dla wtyczek, parserów niezaufanych danych,
worker procesów, interpreterów uruchamiających cudzy kod.

Wymaga **Linuksa >= 5.13** z włączonym `CONFIG_SECURITY_LANDLOCK` (domyślnie
włączone w większości dystrybucji od kilku lat). Im nowsze jądro, tym więcej
możliwości — biblioteka wykrywa wersję ABI w czasie działania
(`landlock::abi_version()`) i pozwala dopasować do niej zakres ograniczeń.

## Instalacja (bytes)

W `Bytes.hk` swojego projektu:

```
[deps]
-> landlock => https://github.com/<twoj-user>/landlock.git
```

a potem:

```
bytes install
```

W kodzie:

```h#
use "bytes -> landlock" from "landlock"
```

## Szybki start

```h#
use "bytes -> landlock" from "landlock"

fn main() is
    if !landlock::is_supported() is
        write("Landlock niewspierany - pomijam sandboxing")
        return
    end

    let paths: [string] = ["/usr", "/lib", "/etc/resolv.conf"]

    match landlock::sandbox_readonly(paths) is
        landlock::LandlockResult::Ok(_) => write("piaskownica aktywna")
        landlock::LandlockResult::Err(e) => write("blad: " + e.message)
    end

    ;; Od teraz proces (i kazdy jego potomek) moze tylko CZYTAC
    ;; z /usr, /lib i /etc/resolv.conf - nic wiecej, nigdzie indziej.
end
```

Pełniejszy przykład, budujący ruleset ręcznie (prawa plikowe + sieciowe,
dopasowane do wykrytej wersji ABI) jest w `examples/full_sandbox.h#`.

## API

| Funkcja | Opis |
|---|---|
| `landlock::abi_version() -> int` | Wersja ABI wspierana przez jądro (`0` = brak wsparcia) |
| `landlock::is_supported() -> bool` | Skrót na `abi_version() > 0` |
| `landlock::new_ruleset(handled_fs, handled_net) -> LandlockResult<Ruleset>` | Tworzy pusty ruleset |
| `landlock::new_ruleset_scoped(handled_fs, handled_net, scoped) -> LandlockResult<Ruleset>` | Jak wyżej + `LANDLOCK_SCOPE_*` (ABI ≥ 6) |
| `landlock::allow_path(rs, path, allowed_access) -> LandlockResult<bool>` | Dodaje regułę dla ścieżki (plik/katalog, rekurencyjnie) |
| `landlock::allow_tcp_port(rs, port, allowed_access) -> LandlockResult<bool>` | Dodaje regułę dla portu TCP (ABI ≥ 4) |
| `landlock::set_no_new_privs() -> LandlockResult<bool>` | Samodzielne `prctl(PR_SET_NO_NEW_PRIVS)` |
| `landlock::restrict_self(rs) -> LandlockResult<bool>` | **Nieodwracalnie** aplikuje ruleset do procesu |
| `landlock::close_ruleset(rs)` | Zamyka fd rulesetu bez aplikowania (sprzątanie po błędzie) |
| `landlock::sandbox_readonly(paths) -> LandlockResult<bool>` | Wygodna funkcja: RO na listę ścieżek + `restrict_self` |
| `landlock::error_string(code) -> string` | Czytelny opis kodu errno w kontekście Landlocka |

Stałe praw dostępu (`landlock_consts::access_fs_execute()`,
`access_fs_write_file()`, `access_fs_read_file()`, ... pełna lista bitów
`LANDLOCK_ACCESS_FS_*`/`LANDLOCK_ACCESS_NET_*`/`LANDLOCK_SCOPE_*`, plus
gotowe kombinacje `access_fs_read_only()` i `access_fs_all_for_abi(abi)`)
są w module `landlock_consts` — patrz `src/landlock_consts.h#`, każda stała
opisana komentarzem.

## Struktura pakietu

```
Bytes.hk
src/
  landlock.h#         - publiczne API (ten moduł importujesz)
  landlock_consts.h#  - stałe ABI, numery syscalli, flagi
  landlock_ffi.h#      - extern static [c]: syscall, prctl, open, close, errno
  landlock_attr.h#     - budowanie surowych bajtów struktur ABI jądra
examples/
  readonly_sandbox.h#
  full_sandbox.h#
tests/
  landlock_test.h#
```

Wewnętrzne moduły (`landlock_consts`/`landlock_ffi`/`landlock_attr`)
celowo mają nazwy poprzedzone `landlock_`, a nie ogólne `consts`/`ffi`/
`attr` — `mod X` w H# ląduje w płaskiej, globalnej przestrzeni nazw
programu (bez wymuszanej prywatności między pakietami), więc krótka,
ogólna nazwa modułu łatwo kolidowałaby z modułem o tej samej nazwie
z innego pakietu użytego w tym samym programie.

## Dlaczego surowe bufory zamiast `struct ... is ... end`

Backend H# reprezentuje każde pole struktury H# jako jednolity 64-bitowy
slot (patrz komentarz przy `htype_to_llvm` w kompilatorze) i nie potrafi
dziś przekazywać przez FFI struktur C o mieszanej szerokości pól (np.
`landlock_path_beneath_attr { __u64; __s32 } __attribute__((packed))`).
Dlatego `src/landlock_attr.h#` układa te struktury ręcznie, bajt po
bajcie, w buforze z `arc_alloc` + `ptr_write_i64`/`ptr_write_i32` — dokładnie
w układzie, jakiego oczekuje jądro. To standardowa technika przy FFI do
C-owych structs o mieszanych typach pól, niezależnie od języka.

`syscall()` i `prctl()` są deklarowane jako zwykłe (niewariadyczne) funkcje
`extern static [c]` o ustalonej liczbie argumentów, mimo że w libc są
wariadyczne — na `x86_64`/`aarch64` (SysV / AAPCS) pierwsze *N* argumentów
całkowitoliczbowych trafia do tych samych rejestrów niezależnie od
wariadyczności deklaracji, więc to bezpieczne i spotykane w bindingach do
innych języków (np. w Ruście crate `libc`/`nix`).

## Architektury

Numery syscalli Landlocka (`444`/`445`/`446`) są takie same na `x86_64`,
`aarch64`, `arm` (EABI), `riscv` i `mips`, bo Landlock został dodany już po
ujednoliceniu generycznej tablicy syscalli między architekturami
(`asm-generic/unistd.h`). **Nie są** przenośne na architektury z własną,
osobną (nie-generyczną) tablicą sprzed tej unifikacji, np. `alpha` czy
`sparc` — na tych platformach ta biblioteka nie zadziała poprawnie. W
praktyce oznacza to pełne wsparcie dla zdecydowanej większości maszyn i
kontenerów Linux dostępnych dziś w chmurze i na desktopie.

## Bezpieczeństwo i ograniczenia

- `landlock::restrict_self(rs)` jest **nieodwracalne** dla procesu, który je
  wywołał (i każdego jego potomka), aż do jego zakończenia — to jest cały
  sens Landlocka. Wołaj je świadomie, najlepiej jako ostatni krok startu
  aplikacji, po otwarciu wszystkich plików/gniazd, których będziesz
  potrzebować.
- Testy pakietu (`tests/landlock_test.h#`) celowo **nigdy nie wołają**
  `restrict_self` — zaaplikowałyby ograniczenia do procesu uruchamiającego
  same testy.
- Ta biblioteka nie tworzy, nie ładuje ani nie linkuje żadnego dodatkowego
  pliku binarnego poza systemową `libc` — nie ma więc własnej powierzchni
  ataku poza tym, co i tak dają `syscall(2)`/`prctl(2)`/`open(2)`.

## Licencja

MIT — zobacz `LICENSE`.
