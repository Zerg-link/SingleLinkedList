# SingleLinkedList
Реализация односвязного списка (аналог *forward_list*). Содержит собственный итератор (**BasicIterator**) и методы по редактированию экземпляра класса.
## Описание
  В данном классе реализованы:

* Конструктор по умолчанию, создающий пустой список.
* Метод **GetSize**, возращающий количество элементов в списке.
* Метод **IsEmpty**, возращающий *true*, если список пустой, и *false* в противном случае.
* Метод **PushFront**, осуществляющий вставку элемента в начало односвязного списка.
* Метод **Clear**, очищающий список без выбрасования исключений.
* Деструктор.
* Поддержка перебора элементов контейнера **SingleLinkedList** с помощью шаблонного класса **BasicIterator**, на основе которого объявлены константный и неконстантный итераторы списка.
* Операции сравнения **==**, **!=**, **<**, **>**, **<=**, **>=**.
* Обмен содержимого двух списков с использованием метода **swap** и шаблонной функции **swap**.
* Конструирование односвязного списка на основе **initializer_list**.
* Надёжные конструктор копирования и операция присваивания (операция присваивания обеспечивает строгую гарантию безопасности исключений - если в процессе присваивания будет выброшено исключение, содержимое левого аргумента операции присваивания должно остаться без изменений).
* Метод **PopFront**, который удаляет первый элемента непустого списка за время O(1) (не выбрасывает исключений).
* Метод **InsertAfter** - за время O(1) вставляет в список новое значение следом за элементом, на который ссылается переданный в **InsertAfter** итератор (метод обеспечивает строгую гарантию безопасности исключений).
* Метод **EraseAfter** - за время O(1) удаляет из списка элемент, следующий за элементом, на который ссылается переданный в **InsertAfter** итератор (не выбрасывает исключений).
* Методы **before_begin** и **cbefore_begin**, возвращающие итераторы, которые ссылаются на фиктивную позицию перед первым элементом списка. Такой итератор используется как параметр для методов **InsertAfter** и **EraseAfter**, когда нужно вставить или удалить элемент в начале списка (разыменовывать этот итератор нельзя).

## Инструкция по использованию

Перенесите класс в свой проект и используйте по назначению.

> Ниже приведён тест для проверки работоспособности класса. 
```C++
void Test4() {
    struct DeletionSpy {
        ~DeletionSpy() {
            if (deletion_counter_ptr) {
                ++(*deletion_counter_ptr);
            }
        }
        int* deletion_counter_ptr = nullptr;
    };

    // Проверка PopFront
    {
        SingleLinkedList<int> numbers{ 3, 14, 15, 92, 6 };
        numbers.PopFront();
        assert((numbers == SingleLinkedList<int>{14, 15, 92, 6}));

        SingleLinkedList<DeletionSpy> list;
        list.PushFront(DeletionSpy{});
        int deletion_counter = 0;
        list.begin()->deletion_counter_ptr = &deletion_counter;
        assert(deletion_counter == 0);
        list.PopFront();
        assert(deletion_counter == 1);
    }

    // Доступ к позиции, предшествующей begin
    {
        SingleLinkedList<int> empty_list;
        const auto& const_empty_list = empty_list;
        assert(empty_list.before_begin() == empty_list.cbefore_begin());
        assert(++empty_list.before_begin() == empty_list.begin());
        assert(++empty_list.cbefore_begin() == const_empty_list.begin());

        SingleLinkedList<int> numbers{ 1, 2, 3, 4 };
        const auto& const_numbers = numbers;
        assert(numbers.before_begin() == numbers.cbefore_begin());
        assert(++numbers.before_begin() == numbers.begin());
        assert(++numbers.cbefore_begin() == const_numbers.begin());
    }

    // Вставка элемента после указанной позиции
    {  // Вставка в пустой список
        {
            SingleLinkedList<int> lst;
            const auto inserted_item_pos = lst.InsertAfter(lst.before_begin(), 123);
            assert((lst == SingleLinkedList<int>{123}));
            assert(inserted_item_pos == lst.begin());
            assert(*inserted_item_pos == 123);
        }

        // Вставка в непустой список
        {
            SingleLinkedList<int> lst{ 1, 2, 3 };
            auto inserted_item_pos = lst.InsertAfter(lst.before_begin(), 123);

            assert(inserted_item_pos == lst.begin());
            assert(inserted_item_pos != lst.end());
            assert(*inserted_item_pos == 123);
            assert((lst == SingleLinkedList<int>{123, 1, 2, 3}));

            inserted_item_pos = lst.InsertAfter(lst.begin(), 555);
            assert(++SingleLinkedList<int>::Iterator(lst.begin()) == inserted_item_pos);
            assert(*inserted_item_pos == 555);
            assert((lst == SingleLinkedList<int>{123, 555, 1, 2, 3}));
        };
    }

    // Вспомогательный класс, бросающий исключение после создания N-копии
    struct ThrowOnCopy {
        ThrowOnCopy() = default;
        explicit ThrowOnCopy(int& copy_counter) noexcept
            : countdown_ptr(&copy_counter) {
        }
        ThrowOnCopy(const ThrowOnCopy& other)
            : countdown_ptr(other.countdown_ptr)  //
        {
            if (countdown_ptr) {
                if (*countdown_ptr == 0) {
                    throw std::bad_alloc();
                }
                else {
                    --(*countdown_ptr);
                }
            }
        }
        // Присваивание элементов этого типа не требуется
        ThrowOnCopy& operator=(const ThrowOnCopy& rhs) = delete;
        // Адрес счётчика обратного отсчёта. Если не равен nullptr, то уменьшается при каждом копировании.
        // Как только обнулится, конструктор копирования выбросит исключение
        int* countdown_ptr = nullptr;
    };

    // Проверка обеспечения строгой гарантии безопасности исключений
    {
        bool exception_was_thrown = false;
        for (int max_copy_counter = 10; max_copy_counter >= 0; --max_copy_counter) {
            SingleLinkedList<ThrowOnCopy> list{ ThrowOnCopy{}, ThrowOnCopy{}, ThrowOnCopy{} };
            try {
                int copy_counter = max_copy_counter;
                list.InsertAfter(list.cbegin(), ThrowOnCopy(copy_counter));
                assert(list.GetSize() == 4u);
            }
            catch (const std::bad_alloc&) {
                exception_was_thrown = true;
                assert(list.GetSize() == 3u);
                break;
            }
        }
        assert(exception_was_thrown);
    }

    // Удаление элементов после указанной позиции
    {
        {
            SingleLinkedList<int> lst{ 1, 2, 3, 4 };
            const auto& const_lst = lst;
            const auto item_after_erased = lst.EraseAfter(const_lst.cbefore_begin());
            assert((lst == SingleLinkedList<int>{2, 3, 4}));
            assert(item_after_erased == lst.begin());
        }
        {
            SingleLinkedList<int> lst{ 1, 2, 3, 4 };
            const auto item_after_erased = lst.EraseAfter(lst.cbegin());
            assert((lst == SingleLinkedList<int>{1, 3, 4}));
            assert(item_after_erased == (++lst.begin()));
        }
        {
            SingleLinkedList<int> lst{ 1, 2, 3, 4 };
            const auto item_after_erased = lst.EraseAfter(++(++lst.cbegin()));
            assert((lst == SingleLinkedList<int>{1, 2, 3}));
            assert(item_after_erased == lst.end());
        }
        {
            SingleLinkedList<DeletionSpy> list{ DeletionSpy{}, DeletionSpy{}, DeletionSpy{} };
            auto after_begin = ++list.begin();
            int deletion_counter = 0;
            after_begin->deletion_counter_ptr = &deletion_counter;
            assert(deletion_counter == 0u);
            list.EraseAfter(list.cbegin());
            assert(deletion_counter == 1u);
        }
    }
}
```
## Требования: 
* C++17(C++1z)  
![ezgif com-gif-maker](https://user-images.githubusercontent.com/82444909/168464442-30227fe3-356c-4ebc-8c54-e4b9d2e38ce0.gif)

