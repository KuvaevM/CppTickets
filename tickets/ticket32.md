# Билет 32 (One Definition Rule и нарушения IFNDR)

## ODR

ODR (One Definition Rule) - во всей программе (то есть включая все файлы) у любой сущности должно быть ровно одно определение. 
Определение с cppreference: Only one definition of any variable, function, class type, enumeration type, concept (since C++20) or template is allowed in any one translation unit (some of these may have multiple declarations, but only one definition is allowed).

### Что с перегрузкой функций
#### Напоминание про перегрузку функций: 
Есть несколько функций с одинаковым именем, но с разными типами аргументов. Тогда компилятор может выбрать наиболее подходящую перегрузку.
#### Что происходит внутри:
API (Application Programming Interface) - показывает, какой программный интерфейс у различных translation unit'ов. (Какие типы у аргументов функции, какие типы возвращаются, в каком namespace лежит и тд). Всё API запоминает компилятор и им можно пользоваться. (Подробнее - [лекция Егора](https://youtu.be/X-6unqJz_uY?list=PL8a-dtqmQc8obAqSKqGkau8qiafPRCxV7&t=1683))
```C++
void foo();
foo();
```
При компиляции все превращается в ABI (Application Binary Interface) - тоже самое, но более низкоуровневое. У компилятора есть регламенты, как и через что возвращается (через какие регистры процессора, через какие места памяти и тд). Например, поэтому нельзя компилировать разные части программы разными компиляторами, в итоге какие-то регламенты могут не совпасть.  
Затем происходит name mangling:
```C++
void foo(); -->  \_Z3foov // v - тип аргумента. 
```
Все это к тому, что перегруженные функции различаются компилятором и нарушения ODR не возникает. Пример:  
foo.cpp:
```C++
void foo(int) {
}
```
main.cpp:
```C++
void foo(int);
void foo() {
}

int main() {
    foo();
    foo(10);
}
```
### Пример ошибок линковщика
#### Multiple definition 
Функция имеет более одного определения.  
foo.cpp:
```C++
void foo() {
}
```
main.cpp:
```C++
void foo() {
}

int main() {
}
```
#### Undefined reference
Функция не имеет опрделения, но при этом где-то используется.
```C++
int main() {
    foo();
}
```
### Примеры IFNDR
IFNDR (Ill-Formed, No Diagnostic Required) - "программа некорректна, сообщать не требуется" - УБ в момент запуска, при этом компилятор об этом не сообщает. Ниже несколько примеров таких УБ (при этом все компилируется).
#### Несовпадение объявлений функций
Везде будет УБ, при этом компилятор не будет предупреждать нас об этом.
С аргументами:

С аргументами по умолчанию:  
foo.cpp:
```C++
void foo(int x = 10) {
    std::cout << x << "\n";
}
```
main.cpp:
```C++
void foo(int = 1000);

int main() {
    foo();
}
```
Возвращаемое значение:  
foo.cpp:
```C++
float foo() {
    return 1000000;
}
```
main.cpp:
```C++
#include <iostream>

double foo();

int main() {
    std::cout << foo();
}
```
#### Несовподение определений классов
Если структура определена в одной трансляции и используется в другой, то там, где она используется, нужно еще раз написать структуру, а также перечислить все ее поля и объявить все методы.
Если метод определен вне структуры:  
foo.cpp:
```C++
#include <vector>
#include <iostream>

struct Foo {
    int a = 10;
    std::vector<int> v;

    void method();
};

Foo get_foo() {
    return Foo{};
}

void Foo::method() {
    std::cout << "method() called " << a << "\n";
}
```
main.cpp:
```C++
#include <vector>
#include <iostream>

struct Foo {
    int a = 10;  // Should be exactly 10, IFNDR otherwise.
    std::vector<int> v;

    void method();
};

Foo get_foo();

int main() {
    get_foo().method();

    Foo f = get_foo();
    f.method();

    Foo f2;
    f2.method();
}
```
Метод может быть определен и в структуре. В этом случае везде нужно написать одно и тоже определение этого метода. При этом это не будет нарушением ODR (смотри inline в методах классов).  
foo.cpp:
```C++
#include <vector>
#include <iostream>

struct Foo {
    int a = 10;
    std::vector<int> v;

    void method() {
        std::cout << "method() called " << a << "\n";
    }
};

Foo get_foo() {
    return Foo{};
}
```
main.cpp:
```C++
#include <vector>
#include <iostream>

struct Foo {
    int a = 10;  // Should be exactly 10, IFNDR otherwise.
    std::vector<int> v;

    void method() {
        std::cout << "method() called " << a << "\n";
    }
};

Foo get_foo();

int main() {
    get_foo().method();

    Foo f = get_foo();
    f.method();

    Foo f2;
    f2.method();
}
```
#### Потенциальные проблемы с любыми глобальными переменными
Ниже программа с глобальной переменной write. Если ее запустить как обычно, то она успешно скомпилируется и сработает.
```C++
#include <iostream>

int write;

int main() {
    std::ios_base::sync_with_stdio(false);
    std::cout << 10;
}
```
Но если скомпилировать эту программу с ключом `-static` ("что значит возьми стандартную библиотеку и встрой ее внутрь exeшника"), то будет UB, так как в программе ODR-violation. В стандартной библиотеке есть глобальная функция с название write. Получилось, что в программе появилось два объекта с одинаковыми названиями. В данном случае процессор пытается вызвать переменную write как функцию. Мораль: используйте namespace!
### Использование inline-переменных/функций/методов для обхода ODR
Замечание: все сетоды, определенные внутри классов по умолчанию inline, но если метод определен вне класса, то inline не является (нужно указывать ручками). При этом это не расспространяется на поля классов.  
```C++
struct Foo {
    int x;
    
    /* inline */ void doSmthWithFoo() { // - inline подставляется автоматически
    }
}
```
Теперь как работает inline: раньше работало по типу макроса, возьми определение и вставь его вместо вызова. Сейчас компиляторы умнее, выбирают что сделать лучше, но точный смысл остался: функция может быть реализована в нескольких единицах трансляции, но одинаково. Вот пример кода с [лекции](https://github.com/hse-spb-2021-cpp/lectures/tree/master/08-211020/03-linkage/01-inline).
### Отличия inline от static/unnamed namespace
Static, примененный к сущностям работает как unnamed namespace. Таким образом данная сущность будет видна в данной единицы трансляции и не видна в других. Как пример: собственная библиотека с автотестами. Там могут найтись несколько функций с одинаковым названием в разных единицах трансляции. Используем static/unnamed namespace -- все хорошо.  
Ключевое отличие: если использовать static/unnamed namespace, то у сущности будет internal linkage (то есть сущность видна только в текущей единице трансляции). inline разрешает создавать определение в нескольких единицах трансляции, но при этом определения должны быть одинаковыми и все помечены inline.

















