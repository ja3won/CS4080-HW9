In `object.h`, I added an `ownsChars` flag and made the character pointer `const`:

```c
struct ObjString {
  Obj obj;
  bool ownsChars;
  int length;
  const char* chars;
};
```

The flag tells the VM whether it should free the character array. A false flag means the characters belong somewhere else. The `const` pointer lets both kinds of strings be read without allowing the stored characters to be changed through that pointer.

I replaced the `takeString()` and `copyString()` declarations with:

```c
ObjString* makeString(bool ownsChars, const char* chars, int length);
```

In `object.c`, I removed the old `allocateString()`, `takeString()`, and `copyString()` functions and added:

```c
ObjString* makeString(bool ownsChars, const char* chars, int length) {
  ObjString* string = ALLOCATE_OBJ(ObjString, OBJ_STRING);
  string->ownsChars = ownsChars;
  string->length = length;
  string->chars = chars;
  return string;
}
```

`makeString()` only allocates the object. It stores the pointer it receives, along with the length and ownership flag. It does not allocate or copy a character array.

In `compiler.c`, I changed `string()` to borrow the characters from the source:

```c
static void string() {
  const char* chars = parser.previous.start + 1;
  int length = parser.previous.length - 2;
  emitConstant(OBJ_VAL(makeString(false, chars, length)));
}
```

The token includes both quotation marks, so I skip the first one and leave both out of the length. Passing false means the resulting string borrows this part of the source. No character copy is needed.

In `vm.c`, I updated `concatenate()`:

```c
static void concatenate() {
  ObjString* b = AS_STRING(pop());
  ObjString* a = AS_STRING(pop());

  int length = a->length + b->length;
  char* chars = ALLOCATE(char, length + 1);
  memcpy(chars, a->chars, a->length);
  memcpy(chars + a->length, b->chars, b->length);
  chars[length] = '\0';

  ObjString* result = makeString(true, chars, length);
  push(OBJ_VAL(result));
}
```

Both input strings are copied into a new buffer with room for the ending `\0`. Since this buffer was allocated for the result, I pass true so the VM will free it later.

In `object.c`, I also changed `printObject()`:

```c
void printObject(Value value) {
  switch (OBJ_TYPE(value)) {
    case OBJ_STRING: {
      ObjString* string = AS_STRING(value);
      printf("%.*s", string->length, string->chars);
      break;
    }
  }
}
```

The borrowed characters still have a closing quote after them, rather than a `\0`. Using the stored length with `%.*s` keeps printing inside the string. The same code also works for concatenated strings.

Finally, in `memory.c`, I changed the `OBJ_STRING` case in `freeObject()`:

```c
case OBJ_STRING: {
  ObjString* string = (ObjString*)object;
  if (string->ownsChars) {
    FREE_ARRAY(char, (char*)string->chars, string->length + 1);
  }
  FREE(ObjString, object);
  break;
}
```

Every string object is allocated on the heap, so the object itself is always freed. The character array is freed only when the flag says it belongs to that object. The cast allows the owned buffer to be passed to the memory function. Borrowing is safe only while the original characters stay valid. The source must remain in memory while any of its strings are still being used.
