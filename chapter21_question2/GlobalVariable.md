I would keep global values in an array. The compiler finds each name's array index and saves it in the bytecode's constants, so normal access at runtime does not need a hash-table search.

In vm.h, replace the globals table with:

```c
Table globalSlots;
ValueArray globalValues;
```

Replace the old globals setup and cleanup in vm.c:

```c
// In initVM():
initTable(&vm.globalSlots);
initValueArray(&vm.globalValues);

// In freeVM():
freeTable(&vm.globalSlots);
freeValueArray(&vm.globalValues);
```

Add VAL_UNDEFINED to ValueType in value.h, then add:

```c
#define UNDEFINED_VAL ((Value){VAL_UNDEFINED, {.number = 0}})
#define IS_UNDEFINED(v) ((v).type == VAL_UNDEFINED)
```

In value.c, add `case VAL_UNDEFINED: printf("<undefined>"); break;` to printValue() and `case VAL_UNDEFINED: return true;` to valuesEqual(). Undefined is internal and separate from nil.

Remove the earlier compiler-only globalNames cache and its init/free calls. In compiler.c, include vm.h and replace identifierConstant() with:

```c
static uint8_t identifierConstant(Token* name) {
  ObjString* key = copyString(name->start, name->length);
  Value slot;

  if (!tableGet(&vm.globalSlots, key, &slot)) {
    slot = NUMBER_VAL(vm.globalValues.count);
    writeValueArray(&vm.globalValues, UNDEFINED_VAL);
    tableSet(&vm.globalSlots, key, slot);
  }

  return makeConstant(slot);
}
```

The constant holds the global slot number. In vm.c, add this alongside the other reading macros in run(), and undefine it at the end:

```c
#define READ_SLOT() ((int)AS_NUMBER(READ_CONSTANT()))
```

Replace the three global cases with:

```c
case OP_GET_GLOBAL: {
  Value value = vm.globalValues.values[READ_SLOT()];
  if (IS_UNDEFINED(value)) {
    runtimeError("Undefined variable.");
    return INTERPRET_RUNTIME_ERROR;
  }
  push(value);
  break;
}
case OP_DEFINE_GLOBAL: {
  vm.globalValues.values[READ_SLOT()] = pop();
  break;
}
case OP_SET_GLOBAL: {
  int slot = READ_SLOT();
  if (IS_UNDEFINED(vm.globalValues.values[slot])) {
    runtimeError("Undefined variable.");
    return INTERPRET_RUNTIME_ERROR;
  }
  vm.globalValues.values[slot] = peek(0);
  break;
}
```

The VM keeps these tables between REPL entries. Reading or assigning an undefined global still gives an error, and nil is still a valid value. This moves the name lookup to compilation. Normal global access uses array indexing, which avoids repeated hashing while the program runs.