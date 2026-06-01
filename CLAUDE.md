# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Compilador Accesible** — a six-phase compiler for `.acc`, a Spanish-keyword programming language targeting visually impaired users. Implemented in C++17 with a separate Python voice-input layer (Vosk).

## Build & Run

```bash
make              # Build → build/compilador
make run          # Build + launch interactive mode
make test         # Run all example files in ejemplos/
make tokens       # Tokenize-only output (--tokens flag)
make ast          # AST-only output (--ast flag)
make ir           # Intermediate code (TAC) output (--ir flag)
make gen          # Generated Python output (--gen flag)
make check        # Silent validation (--check flag, exits 0/1)
make clean        # Remove build/ artifacts
```

Direct CLI usage:

```bash
./build/compilador archivo.acc                    # Full compile report
./build/compilador --tokens archivo.acc           # Token table only
./build/compilador --ast archivo.acc              # AST only
./build/compilador --ir archivo.acc               # TAC + optimized TAC
./build/compilador --gen archivo.acc              # Generated Python code
./build/compilador --salida salida.py archivo.acc # Write Python to file
./build/compilador --check archivo.acc            # Silent; exit code 0/1
./build/compilador                                # Interactive REPL
```

Voice layer (Python/Vosk, offline):

```bash
cd voz && bash instalar.sh   # One-time setup
bash voz/ejecutar.sh          # Launch voice input
```

## Architecture

Full six-phase compiler pipeline:

```
Source (.acc) → Lexer → []Token → Parser → AST
              → Semantic Analysis → IR/TAC → Optimization → Python Code
```

All source lives in [src/](src/):

| File | Role |
|---|---|
| [token.hpp](src/token.hpp) / [token.cpp](src/token.cpp) | `TipoToken` enum (32 types), `Token` struct with line/column |
| [lexer.hpp](src/lexer.hpp) / [lexer.cpp](src/lexer.cpp) | Phase 1 — produces `std::vector<Token>`; O(1) case-insensitive keyword lookup |
| [ast.hpp](src/ast.hpp) / [ast.cpp](src/ast.cpp) | AST node types inheriting from abstract `Nodo`; all ownership via `std::unique_ptr` |
| [parser.hpp](src/parser.hpp) / [parser.cpp](src/parser.cpp) | Phase 2 — recursive descent; `parsear()` returns `unique_ptr<NodoPrograma>` |
| [semantico.hpp](src/semantico.hpp) / [semantico.cpp](src/semantico.cpp) | Phase 3 — type checking, undeclared variable detection, redeclaration warnings |
| [codintermedio.hpp](src/codintermedio.hpp) / [codintermedio.cpp](src/codintermedio.cpp) | Phase 4 — generates Three-Address Code (TAC) from the AST |
| [optimizador.hpp](src/optimizador.hpp) / [optimizador.cpp](src/optimizador.cpp) | Phase 5 — constant folding, constant propagation, dead code elimination |
| [generador.hpp](src/generador.hpp) / [generador.cpp](src/generador.cpp) | Phase 6 — generates executable Python 3 code from the AST |
| [main.cpp](src/main.cpp) | CLI flag handling, file I/O, interactive mode, color output, GCC-style error reporting |

### Lexer (Phase 1)

- `tokenizar()` returns `vector<Token>` and continues past errors (collecting all lexical errors before stopping).
- `NUEVA_LINEA` tokens are significant — they act as statement separators.
- Keywords are normalized to lowercase before lookup.

### Parser (Phase 2)

- Pure recursive descent, no parser generator.
- Expression precedence: `parsearExpresion()` → `parsearTermino()` → `parsearFactor()` (additive → multiplicative → atomic).
- Error recovery skips to end of line and continues.
- `ver(offset)` provides arbitrary lookahead over the token stream.

### AST Node Hierarchy

- **Expressions:** `NodoNumero`, `NodoTexto`, `NodoBooleano`, `NodoIdentificador`, `NodoOpBinaria`
- **Conditions:** `NodoComparacion`, `NodoOpLogica`, `NodoNegar`
- **Statements:** `NodoDeclaracion`, `NodoCondicional`, `NodoBucleRepetir`, `NodoBucleMientras`, `NodoMostrar`
- **Root:** `NodoPrograma` holds a `vector<NodoPtr>` of top-level statements

### Semantic Analysis (Phase 3)

- Checks: undeclared variable use, redeclaration (warning), type incompatibility in arithmetic and comparisons, non-numeric loop count.
- Flat symbol table (`unordered_map<string, InfoVariable>`) — no scoped blocks yet.

### Intermediate Code / TAC (Phase 4)

`GeneradorIR` walks the AST and emits `InstrIR` records:
- `ASIGNAR`: `dest = src`
- `OP_BIN`: `dest = arg1 op arg2` (arithmetic `+`, `-`, `*`, `/` and boolean `==`, `>`, `<`, `>=`, `<=`, `&&`, `||`)
- `NEGACION_LOGICA`: `dest = !arg1`
- `ETIQUETA`: label declaration
- `SALTO`: unconditional jump
- `SALTO_SI_FALSO`: conditional jump if false
- `MOSTRAR`: print instruction

### Optimization (Phase 5)

Three sequential passes in `Optimizador`:
1. **Constant folding** — evaluates `3 + 4` → `7`, algebraic identities (`x + 0 = x`, `x * 1 = x`)
2. **Constant propagation** — substitutes known temporals with their constant value across basic blocks
3. **Dead code elimination** — removes assignments to temporals (`_tN`) that are never read

### Code Generation / Python (Phase 6)

`GeneradorPython` translates the AST directly to Python 3:
- `definir x como 5` → `x = 5`
- `mostrar x` → `print(x)`
- `si … entonces … sino … fin si` → `if … : … else: …`
- `repetir N veces … fin repetir` → `for _i in range(N): …`
- `mientras … fin mientras` → `while …: …`

Boolean literals map to Python's `True`/`False`. Text strings are escaped properly.

### Output

- Token table: ANSI-colored by category (blue=keyword, yellow=operator, green=literal, cyan=identifier).
- AST: printed with tree connectors (`├──`, `└──`).
- TAC: numbered instruction listing with color-coded fields.
- Generated Python: printed to stdout or written to file via `--salida`.
- Errors: GCC/Clang style — source line + column + caret `^` pointer.

## Language Grammar (EBNF)

```
programa     ::= instruccion* EOF
instruccion  ::= declaracion | condicional | bucle_repetir | bucle_mientras | mostrar
declaracion  ::= "definir" IDENT "como" expresion
condicional  ::= "si" condicion "entonces" NEWLINE bloque ["sino" NEWLINE bloque] "fin" "si"
bucle_repetir::= "repetir" expresion "veces" NEWLINE bloque "fin" "repetir"
bucle_mientras::= "mientras" condicion NEWLINE bloque "fin" "mientras"
mostrar      ::= "mostrar" expresion
bloque       ::= instruccion*

expresion    ::= termino ((mas | menos) termino)*
termino      ::= factor ((por | entre) factor)*
factor       ::= NUMERO | TEXTO | verdadero | falso | IDENT | "(" expresion ")"
condicion    ::= no condicion_base | condicion_base ((y | o) condicion_base)*
condicion_base::= expresion comparador expresion
comparador   ::= "es igual a" | "es mayor que" | "es menor que" | "es mayor o igual que" | "es menor o igual que"
```

Key language rules:
- Keywords are case-insensitive Spanish words.
- Newlines are statement separators; tabs and spaces are otherwise ignored.
- Multi-word comparators (`es igual a`, etc.) are each a single token type.
- Comments start with `#`.

## Example Programs

- [ejemplos/validos/](ejemplos/validos/) — `basico.acc`, `condicion.acc`, `completo.acc`
- [ejemplos/errores/](ejemplos/errores/) — `errores_lex.acc`, `errores_sint.acc` (intentional errors for testing)
