I would use a hash table to remember which global names already have a constant slot. The key is the name, and the value is its index in the constant table.

These changes go in compiler.c. Add this with the includes and variables:

```c
#include "table.h"

static Table globalNames;
```

Initialize the table before parsing, then free it when compilation finishes:

```c
bool compile(const char* source, Chunk* chunk) {
  initScanner(source);
  compilingChunk = chunk;
  parser.hadError = false;
  parser.panicMode = false;
  initTable(&globalNames);

  advance();
  while (!match(TOKEN_EOF)) {
    declaration();
  }

  endCompiler();
  freeTable(&globalNames);
  return !parser.hadError;
}
```

Replace identifierConstant() with:

```c
static uint8_t identifierConstant(Token* name) {
  ObjString* key = copyString(name->start, name->length);
  Value saved;

  if (tableGet(&globalNames, key, &saved)) {
    return (uint8_t)AS_NUMBER(saved);
  }

  uint8_t slot = makeConstant(OBJ_VAL(key));
  tableSet(&globalNames, key, NUMBER_VAL(slot));
  return slot;
}
```

Now, using the same global name again reuses its old slot. For example, declaring score and then reading score several times only needs one constant entry for the name. This adds a little work and temporary memory during compilation. Checking the hash table takes constant time on average, so we avoid scanning the constant array for every name.

The runtime still executes the same instructions, so I would not expect much change in execution speed. The main benefit is using fewer constant slots and being less likely to hit the 256-constant limit. Strings are already interned, so this saves repeated entries in the constant array rather than extra copies of the text. I also think this is a good trade-off. The extra work happens during compilation, and the smaller constant table is kept for the program's whole execution.