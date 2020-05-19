---
layout: page
title: Inline function in C & C++
comments: true
---

## Function-call Overhead

다음과 같은 코드가 있다고 합시다.

```c
#include <stdio.h>

int inc(int n){ return n + 1; }

int main(){
    for(int i=0; i<0x1000000; i=inc(i)){}
}
```

*Figure1. no inline*



함수는 컴파일 될 때, <u>prologue</u>와 <u>epilogue</u> 코드가 부가적으로 생성됩니다. 이들은 <u>호출 규약</u>을 지키기 위해 컴파일러가 삽입하는 어셈블리 코드입니다. 함수가 단순하면 단순할 수록 함수 실행 과정에서 발생하는 부하 중 이들이 차지하는 비중이 커집니다. 따라서 함수의 길이가 짧다면, 이러한 호출 과정을 없애는 것이 성능 향상에 도움이 될 수 있습니다.

![KakaoTalk_20200515_020547997](/public/images/KakaoTalk_20200515_020547997-1589481308882.jpg)

*Figure2. Overheads: simple func. vs complex func.* 



*Figure1*의 `inc()` 함수는 컴파일 되면 다음과 같은 어셈블리 코드가 됩니다.

```assembly
inc:
0x000000000000064a <+0>:     push   rbp
0x000000000000064b <+1>:     mov    rbp,rsp
0x000000000000064e <+4>:     mov    DWORD PTR [rbp-0x4],edi
0x0000000000000651 <+7>:     mov    eax,DWORD PTR [rbp-0x4]
0x0000000000000654 <+10>:    add    eax,0x1
0x0000000000000657 <+13>:    pop    rbp
0x0000000000000658 <+14>:    ret
```

*Figure3. Compiled 'inc()'*

여기서 실제로 함수의 기능과 관련된 코드는 `0x654: add eax, 0x1` 뿐임을 확인할 수 있습니다. 그 외의 코드는 함수 호출과정에서의 인자 전달, 스택 프레임 관리 등을 위해 컴파일러가 작성한 코드입니다

<br>

## Inline: escape from overhead

Inline은 C와 C++의 키워드 입니다. 함수의 반환형 앞에 위치할 수 있고, 이 키워드가 붙은 inline 함수는 컴파일 타임에 호출이 함수의 정의로 <u>대체될 수 있습니다</u>. 호출을 위해 생성되는 부가적인 코드는 모두 제거되고, 컴파일된 함수의 코드가 그대로 삽입됩니다.

그런데 왜 '<u>대체될 수 있다</u>'고 하냐면, inline키워드는 컴파일러에게 inline처리 되기를 바란다는 힌트를 주는 것일 뿐 강제하지는 않기 때문입니다. 컴파일러가 판단하기에 적절할 때에만 inline처리가 됩니다.

매크로와 기능이 유사하게 느껴질 수 있는데, 매크로는 전처리기에 의해 코드의 문자열을 치환하는 것이므로 inline과는 차이가 있습니다. 구글은 [style guide](https://google.github.io/styleguide/cppguide.html#Preprocessor_Macros)를 통해 매크로 사용을 지양하고 inline으로 기능을 대신하라고 하는데, 이유는 한 번 읽어보시면 좋습니다. 실제로 코딩을 해보면 매크로는 type을 명시하지 않아서 실수를 유발하기 쉽고, 일반 함수와 생김새가 달라서 가독성을 해치며 작성하기도 불편합니다. 경험상 매크로가 필수적인 경우는 `.S` 확장자를 갖는 어셈블리 코드에서 헤더 파일로 include할 때와 같은 특수한 경우 뿐이었습니다. 이럴 때는 어셈블러(e.g. `as`)가 일반 함수를 처리하지 못하기 때문에 매크로 함수를 사용해야 합니다.



<br>

## Inline function in real programming

다음은 제가 실제로 inline함수를 사용하면서 알게된 것들 입니다.

### 1. inline함수는 웬만하면 'header'에 'static'으로 정의되야 한다.

이를 이해하려면 컴파일러의 <u>번역 단위(Translation Unit, TU)</u>에 대해 알아야 하는데, 번역 단위는 전처리가 끝난 코드, 즉 header inclusion이나 매크로 치환등의 과정이 완료된 코드를 의미합니다. 하나의 거대한 프로젝트를 빌드 할 때, 컴파일러는 여러개의 번역 단위를 순차적으로 번역해서 최종 결과물을 만들어 냅니다.

일반적으로 함수를 작성할 때는 헤더 파일에 선언만 포함시키고, 정의는 다른 소스 코드에 작성하는게 일반적입니다. 그러면 다른 코드에서는 그 헤더만을 include해도 해당 함수를 사용할 수 있습니다. 그런데 컴파일러의 입장에서 그 코드들은 전처리를 마친 뒤에도 해당 함수들의 선언만 존재하고 정의가 없습니다. 이런 경우에 컴파일러는 symbol을 만들어서 나중에 링커가 symbol을 해당 함수에 연결할 수 있게 해줍니다.

다음 코드로 예를 들겠습니다.

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

*Figure4. no definition in TU of (test.c/add.h)*



*Figure4*에서 (test.c/add.h), (add.c/add.h)가 각각 다른 번역 단위가 될 것입니다. 그리고 첫번째 번역 단위를 컴파일할 때는 함수의 정의가 없음이 당연합니다. 궁금하신 분은 `gcc -E test.c`로 확인해 보시면 됩니다. 

이 번역 단위들을 링크없이 컴파일만한 결과물은 `gcc -c test.c add.c `로 확인할 수 있습니다. symbol 정보를 추출하면 다음과 같습니다.

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
11: 0000000000000000     0 NOTYPE  GLOBAL DEFAULT  UND add #symbol for link
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

test.o에 있는 add symbol이 링크과정을 거치면 add.o의 add함수를 가리키게 됩니다.

---

그런데 inline함수는 이런 방식으로 컴파일하기가 어렵습니다.



1. inline은 header파일에 정의까지 포함해라.
2. C++에서 클래스 정의에 포함된 멤버 함수의 정의는 자동으로 inline 키워드가 붙은 걸로 처리된다.
3. C++에서 클래스 정의에 포함된 멤버 함수의 정의에는 static을 붙이지 마라.





