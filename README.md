# Vector template class.
## Возможная реализация std::vector стандартной шаблонной библиотеки.

Проект разрабатывался с целью глубокого понимания внутреннего устройства одного из самого эффективного контейнера STL. При написании шаблонного класса Vector использовались приемы работы с сырой памятью с помощью собственной RAII-обертки, функции стандартной библиотеки создающие и удаляющие группы объектов в неинициализированной области памяти, методы обеспечения гарантий безопасности исключений, оптимизации вставки элементов. 

### Шаблонный класс RawMemory

Шаблонный класс RawMemory - RAII-класс, отвечающий за хранение буфера, который вмещает заданное количество элементов, и предоставляющий доступ к элементам по индексу.

Операция копирования не имеет смысла для класса *RawMemory*, так как у него нет информации о количестве находящихся в сырой памяти элементов. Копирование элементов определено для класса Vector, который использует сырую память для размещения элементов и знает об их количестве. Поэтому конструктор копирования и копирующий оператор присваивания в классе *RawMemory* запрещены. Это исключит возможность их случайного вызова.

Перемещающие конструктор и оператор присваивания не выбрасывают исключений и выполняются за O(1).

Приватные методы *Allocate* и *Deallocate* отвечают за выделение и освобождение сырой памяти. Если размер выделяемого массива нулевой, функция *Allocate* не выделяет память, а просто возвращает *nullptr*. Такая оптимизация направлена на экономию памяти — нет смысла выделять динамическую память под хранение пустого массива и зря расходовать память под служебные данные.

### Шаблонный класс Vector.

Vector (далее - вектор) резервирует «сырую» память, а элементы конструирует в ней только по мере надобности. Также вектор по возможности использует конструирование объектов на месте и операцию перемещения объектов, а не копирования.

Вектор должен не только эффективно работать с объектами в памяти, но и быть надёжным при возникновении исключений. Это шаблонный класс, способный хранить объекты произвольного типа, в том числе и объекты, конструкторы которых могут выбрасывать исключения. Например, при нехватке памяти или в других нештатных ситуациях.

Функции стандартной библиотеки [**std::uninitialized***](https://en.cppreference.com/w/cpp/header/memory), создающие объекты в неинициализированной памяти, позволили упростить реализацию класса вектор и оптимизировать методы, использующие перемещение объектов в памяти.

#### Функционал Vector.

- Конструктор по умолчанию инициализирует вектор нулевого размера и вместимости, данные которого указывают на *nullptr*.
- Конструктор *Vector(size_t size)* внутри себя использует функцию *std::uninitialized_default_construct_n*, которая конструирует в сырой памяти элементы массива в количестве *size*.
- Конструктор копирования использует функцию *std::uninitialized_copy_n*.
- Перемещающий конструктор класса вектор использует соответствующий конструктор *RawMemory*. После перемещения новый вектор станет владеть данными исходного вектора. Перемещающий конструктор. Выполняется за O(1) и не выбрасывает исключений.
- Деструктор вызывает функцию *std::destroy_n*.
- Копирующий оператор присваивания обрабатывает 2 ситуации:
    - Размер вектора-источника меньше размера вектора-приёмника;
    - Размер вектора-источника больше или равен размеру вектора-приёмника.
    
    Если операция присваивания одного из элементов вектора выбросит исключение, вектор останется в согласованном состоянии. Но восстановить состояние, которое было до операции присваивания, не получится. Оператор копирующего присваивания выполняется за O(N), где N — максимум из размеров векторов, участвующих в операции.
- Оператор перемещающего присваивания использует внутренний метод Swap, обменивающий состояния двух векторов. Оператор перемещающего присваивания выполняется за O(1) и не выбрасывает исключений.
- В константном *operator[]* используется оператор *const_cast*, чтобы снять константность с ссылки на текущий объект и вызвать неконстантную версию *operator[]*. Так получится избавиться от дублирования проверки *assert(index < size)*. Оператор *const_cast* позволяет сделать то, что нельзя, но, если очень хочется, можно. В данном случае нельзя вызвать неконстантный метод из константного. Но неконстантный *operator[]* тут не модифицирует состояние объекта, поэтому его можно вызвать, предварительно сняв константность с объекта.

***
- Метод *EmplaceBack* должен уметь вызывать любые конструкторы типа *T*, передавая им как константные и неконстантные ссылки, так и *rvalue-ссылки* на временные объекты. Для этого *EmplaceBack* является вариативным шаблоном, который принимает аргументы конструктора *T* по *Forwarding-ссылкам*. Метод *EmplaceBack* предоставляет строгую гарантию безопасности исключений, когда выполняется любое из условий:
    - *move-конструктор* у типа *T* объявлен как *noexcept*;
    - тип *T* имеет публичный конструктор копирования.
    
    Если у типа *T* нет конструктора копирования и move-конструктор может выбрасывать исключения, метод *EmplaceBack* предоставляет базовую гарантию безопасности исключений. Метод выполняется за время O(1).

- Метод *Emplace* конструирует элемент по указанной позиции. Имеет тот же принцип работы что и *EmplaceBack*, но имеет более сложную логику и время выполнения. Возвращает итератор, указывающий на вставленный элемент в новом блоке памяти.

- Методы *PushBack* и его перегруженная версия по *rvalue-ссылке* реализованы через метод *EmplaceBack*.

- Методы *Insert* и его перегруженная версия по *rvalue-ссылке* реализованы через метод *Emplace*. Возвращает итератор, указывающий на вставленный элемент в новом блоке памяти.

- Метод *Erase* удаляет элемент, на который указывает переданный итератор. Метод *Erase* возвращает итератор, который ссылается на элемент, следующий за удалённым. А если удалялся последний элемент вектора, возвращает *end-итератор*.

- Метод *PopBack* разрушает последний элемент вектора и уменьшает размер вектора на единицу. Вызов *PopBack* на пустом векторе приводит к неопределённому поведению. Имеет сложность O(1) и не выбрасывает исключений при вызове у непустого вектора.

- Метод *Resize* - изменяет количество элементов в векторе. Сложность метода *Resize* должна линейно зависеть от разницы между текущим и новым размером вектора. Если новый размер превышает текущую вместимость вектора, сложность операции может дополнительно линейно зависеть от текущего размера вектора.

- Метод *Reserve* выделяет запрашиваемый размер памяти.

- Метод *Swap*, выполняющий обмен содержимого вектора с другим вектором. Операция имеет сложность O(1) и не выбрасывает исключений.

- Приватный метод *ReplaceElementsInMemory*. Шаблоны *std::is_copy_constructible_v* и *std::is_nothrow_move_constructible_v* помогают узнать, есть ли у типа копирующий конструктор и *noexcept-конструктор* перемещения. Выполняются эти шаблоны во время компиляции и, используя *if constexpr*, можно избежать runtime-проверки веток условий.

- Метод *Size* возвращает размер вектора.

- Метод *Capacity* возвращает допустимую вместимость вектора.

***

### Сборка, установка и системные требования.
Тривиальное использование - скопировать и вставить в свой проект заголовочный файл **vector.h**. Для успешной сборки необходим любой современный компилятор с поддержкой стандрта C++17 и выше.

Для отработки заявленных в main.cpp тестов:

clang\++ -std=c\++17 main.cpp
или
g\++ -std=c\++17 main.cpp

Автор: [MrGorkiy](https://github.com/MrGorkiy)
