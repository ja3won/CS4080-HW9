Both programs store the same 1,000 keys and do 5,000,000 lookups per run. They average five timed runs after one warm-up. I made sure creating strings and setting up the table happen outside the timer.

The first program measures successful lookups.

```c
#include <stdio.h>
#include <time.h>
#include "memory.h"
#include "object.h"
#include "vm.h"
#define LOOKUPS 5000000
VM vm;

int main(void) {
  Table table;
  ObjString* keys[2000];
  Value value;
  initTable(&table);
  initTable(&vm.strings);
  for (int i = 0; i < 2000; i++) {
    char text[20];
    int length = snprintf(text, sizeof(text), "key%d", i);
    keys[i] = copyString(text, length);
    if (i < 1000) tableSet(&table, keys[i], NUMBER_VAL(i));
  }
  for (int i = 0; i < 1000; i++) {
    if (!tableGet(&table, keys[i], &value)) return 1;
    if (!IS_NUMBER(value) || AS_NUMBER(value) != i) return 1;
  }

  double total = 0;
  for (int run = 0; run <= 5; run++) {
    int found = 0;
    clock_t start = clock();
    for (int i = 0; i < LOOKUPS; i++) {
      found += tableGet(&table, keys[i % 1000], &value);
    }
    double ms = 1000.0 * (clock() - start) / CLOCKS_PER_SEC;
    if (found != LOOKUPS) return 1;
    if (run > 0) total += ms; // Run 0 is the warm-up.
  }
  printf("Successful lookups: %.3f ms\n", total / 5);
  puts("Checks passed.");
  freeTable(&table);
  freeTable(&vm.strings);
  freeObjects();
  return 0;
}
```

The second program measures missing-key lookups.

```c
#include <stdio.h>
#include <time.h>
#include "memory.h"
#include "object.h"
#include "vm.h"
#define LOOKUPS 5000000
VM vm;

int main(void) {
  Table table;
  ObjString* keys[2000];
  Value value;
  initTable(&table);
  initTable(&vm.strings);
  for (int i = 0; i < 2000; i++) {
    char text[20];
    int length = snprintf(text, sizeof(text), "key%d", i);
    keys[i] = copyString(text, length);
    if (i < 1000) tableSet(&table, keys[i], NUMBER_VAL(i));
  }
  for (int i = 0; i < 1000; i++) {
    if (!tableGet(&table, keys[i], &value)) return 1;
    if (!IS_NUMBER(value) || AS_NUMBER(value) != i) return 1;
  }

  double total = 0;
  for (int run = 0; run <= 5; run++) {
    int found = 0;
    clock_t start = clock();
    for (int i = 0; i < LOOKUPS; i++) {
      found += tableGet(&table, keys[1000 + i % 1000], &value);
    }
    double ms = 1000.0 * (clock() - start) / CLOCKS_PER_SEC;
    if (found != 0) return 1;
    if (run > 0) total += ms; // Run 0 is the warm-up.
  }
  printf("Missing-key lookups: %.3f ms\n", total / 5);
  puts("Checks passed.");
  freeTable(&table);
  freeTable(&vm.strings);
  freeObjects();
  return 0;
}
```

Actual results averaged over five runs:

| Program | Average time for 5,000,000 lookups |
| --- | ---: |
| Successful lookups | 15.141 ms |
| Missing-key lookups | 25.667 ms |

Both programs passed their value and lookup checks. For these keys, missing-key lookups took about 1.70 times as long as successful lookups. A successful lookup stops when it finds the key. A missing-key lookup keeps searching until it reaches an unused slot. I chose these tests because finding an existing key and checking for a missing key are both common uses of a hash table. Using the same table size and lookup count makes their times easier to compare.

