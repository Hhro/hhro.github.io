<br>

**분할 컴파일**

컴파일러에 들어가는 입력 파일을 번역 단위(Translation Unit, TU)라고 한다. 번역 단위는 전처리 과정을 거친 소스코드 이기 때문에 헤더 포함(header inclusion) 이나 매크로 치환(macro replacement)등이 모두 적용된 상태이다. 

어떤 번역 단위에는 함수의 선언과 정의가 같이 존재할 수도 있고, 아닐 수도 있다. 번역 단위 내부에 선언만 있고, 정의가 존재하지 않는다면, 컴파일러는 함수의 주소를 링크 과정에서 채울 수 있도록 심볼만 만들어 놓는다. 그러면 나중에 링커가 함수의 구현체를 찾아서 연결시켜준다.

다음 코드로 이해를 돕겠다.

```c
// test.c
#include <stdio.h>
#include "add.h"

int main(){
    printf("%d",add(3,4));
}

// add.h
int add(int a, int b);


// add.c
#include "add.h"

int add(int a, int b){
    return a + b;
}
```

*Figure3. 간단한 분할 컴파일 예제*

위 코드를 컴파일 할 때, (test.c/add.h/stdio.h/...) 그리고 (add.c/add.h) 각각 두 번역 단위가 존재할 것이다. 전자에는 `add`함수의 선언만 존재하고, 정의는 후자에 포함돼있다. 따라서 각 번역단위를 **gcc**로 컴파일 하고 **readelf**로 심볼정보를 확인해 보면, 전자의 오브젝트 파일에는 UNDEF `add` 심볼이, 후자의 오브젝트 파일에는 FUNC 타입의 `add`심볼이 있는 것을 확인할 수 있다. 링크 과정에서는 후자에서 함수의 구현체를 찾아 전자의 UNDEF를 해소한다.

만약 함수의 정의가 어느 번역 단위에도 존재하지 않는다면, Undefined reference에러가 발생하게 된다.

```bash
$ readelf -s test.o
Symbol table '.symtab' contains 13 entries:
Num:    Value          Size Type    Bind   Vis      Ndx Name
 0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
 1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS test.c
 2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
 3: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
 4: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
 5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
 6: 0000000000000000     0 SECTION LOCAL  DEFAULT    7
 7: 0000000000000000     0 SECTION LOCAL  DEFAULT    8
 8: 0000000000000000     0 SECTION LOCAL  DEFAULT    6
 9: 0000000000000000    45 FUNC    GLOBAL DEFAULT    1 main
10: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND _GLOBAL_OFFSET_TABLE_
11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND add #symbol required defining
12: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND printf
    
$ readelf -s add.o
Symbol table '.symtab' contains 9 entries:
Num:    Value          Size Type    Bind   Vis      Ndx Name
 0: 0000000000000000     0 NOTYPE  LOCAL  DEFAULT  UND
 1: 0000000000000000     0 FILE    LOCAL  DEFAULT  ABS add.c
 2: 0000000000000000     0 SECTION LOCAL  DEFAULT    1
 3: 0000000000000000     0 SECTION LOCAL  DEFAULT    2
 4: 0000000000000000     0 SECTION LOCAL  DEFAULT    3
 5: 0000000000000000     0 SECTION LOCAL  DEFAULT    5
 6: 0000000000000000     0 SECTION LOCAL  DEFAULT    6
 7: 0000000000000000     0 SECTION LOCAL  DEFAULT    4
 8: 0000000000000000    20 FUNC    GLOBAL DEFAULT    1 add #compiled function
```

*Figure4. 심볼들*

<br>

**Inline에서는 분할 컴파일이 어렵다**

