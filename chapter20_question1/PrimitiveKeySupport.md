I used `Value` instead of `ObjString*` for the keys. Then, I added `VAL_EMPTY` to `ValueType` in value.h, followed with:

```c
#define IS_EMPTY(v) ((v).type == VAL_EMPTY)
#define EMPTY_VAL ((Value){VAL_EMPTY, {.number = 0}})
uint32_t hashValue(Value value);
```

In value.c, I added `case VAL_EMPTY: return true;` to valuesEqual(). In printValue(),I also added `case VAL_EMPTY: printf("<empty>"); break;`.

The hashing code:

```c
static uint32_t hashDouble(double value) {
  union BitCast {
    double value;
    uint32_t ints[2];
  };

  union BitCast cast;
  cast.value = value + 1.0;
  return cast.ints[0] + cast.ints[1];
}

uint32_t hashValue(Value value) {
  switch (value.type) {
    case VAL_BOOL:   return AS_BOOL(value) ? 3 : 5;
    case VAL_NIL:    return 7;
    case VAL_NUMBER: return hashDouble(AS_NUMBER(value));
    case VAL_OBJ:    return AS_STRING(value)->hash;
    case VAL_EMPTY:  return 0;
  }
  return 0;
}
```

From the research I did on the implementation, it would have taken too long, so here's my idea of what to do instead.

In table.h, change Entry's key to `Value`. Also change the key parameter to `Value` in tableGet(), tableSet(), and tableDelete(), in both the declarations and definitions. Do the same for findEntry() in table.c. In table.c, use `hashValue(key)` instead of `key->hash`. Use `valuesEqual()` instead of comparing key pointers. Replace the NULL key checks with `IS_EMPTY()`, and set new or deleted keys to `EMPTY_VAL`. Keep the existing tombstone setup: empty key plus nil value means unused, and empty key plus true value means deleted. Keep searching past deleted entries. Reusing one shouldn't increase count again. In tableFindString(), only return NULL for an unused bucket. For occupied string keys, use `AS_STRING(entry->key)` before checking the length, hash, and characters. Check `IS_STRING()` first so other key types aren't treated as strings.

In object.c, wrap the string key:

```c
tableSet(&vm.strings, OBJ_VAL(string), NIL_VAL);
```

`0` and `-0` need the same hash because they compare equal, and the hashing above handles that. NaN still needs a separate lookup policy since it doesn't equal itself. For class instances, I'd use object identity because it's simpler. If classes define their own equality and hashes, equal objects need equal hashes. Changing fields used by the hash could break lookup, and garbage collection has to keep key objects alive.