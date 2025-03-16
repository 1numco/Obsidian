Недавно рассказывал в общих чертах про то, как malloc на самом деле аллоцирует память. Попробуем теперь самостоятельно найти нужную циферку.

Почти очевидно, что размер должен храниться с отрицательным смещением от указателя на начало блока. Размер ведь динамический, поэтому сохранять его после всех данных было бы плохой идеей, ибо мы бы просто не нашли никогда эту чиселку. И так, как у нас всего 2^64 адресуемых ячеек памяти, то размер выделенного блока должен помещаться в 8 байт. 

Попробуем подобрать нужный сдвиг.

![[Pasted image 20231112085811.png]]
```C++
#include <iostream>
#include <cstring>
#include <memory>
#include <malloc.h>

int main() {
	uint64_t * ptr = (uint64_t *)malloc(100);
	std::cout << *(ptr - 1) << std::endl;
	std::cout << *(ptr - 2) << std::endl;
	std::cout << *(ptr - 3) << std::endl;
	free(ptr);
	return 0;
}
```
Вывод получился таким:
113
0
0

Похоже, что информация о размере выделенного блока хранится прям перед эти самым блоком. Оно и понятно, выделить на 8 байт больше памяти, положить размер блока в первые 8 байт, а наружу дать указатель, сдвинутый на 8 байт - очень хорошая и простая идея.

Однако мы не видим 108. Мы видим 113. Откуда эти рандомные цифры берутся?

Проведем еще эксперимент. 

![[Pasted image 20231112114335.png]]

```C++
#include <iostream>
#include <cstring>
#include <memory>
#include <malloc.h>

int main() {
	std::vector<uint64_t> block_sizes = {1, 3, 5, 8, 10, 24, 25, 40, 41, 50, 56, 57, 100, 512, 1000, 10000, 100000};
	for (const auto & size: block_sizes)
	{
		uint64_t * ptr = (uint64_t *)malloc(size);
		std::cout << "Requested block size: " << size <<
					 ". Actual block size: " << *(ptr - 1) <<
					 ". Difference: " << *(ptr - 1) - size << std::endl;
	}
	free(ptr);
	return 0;
}
```


Зададим последовательно увеличивающиеся размеры блоков и будем смотреть, каков реальный выделенный размер. Вывод следующий:

Requested block size: 1. Actual block size: 33. Difference: 32
Requested block size: 3. Actual block size: 33. Difference: 30
Requested block size: 5. Actual block size: 33. Difference: 28
Requested block size: 8. Actual block size: 33. Difference: 25
Requested block size: 10. Actual block size: 33. Difference: 23
Requested block size: 24. Actual block size: 33. Difference: 9
Requested block size: 25. Actual block size: 49. Difference: 24
Requested block size: 40. Actual block size: 49. Difference: 9
Requested block size: 41. Actual block size: 65. Difference: 24
Requested block size: 50. Actual block size: 65. Difference: 15
Requested block size: 56. Actual block size: 65. Difference: 9
Requested block size: 57. Actual block size: 81. Difference: 24
Requested block size: 100. Actual block size: 113. Difference: 13
Requested block size: 512. Actual block size: 529. Difference: 17
Requested block size: 1000. Actual block size: 1009. Difference: 9
Requested block size: 10000. Actual block size: 10017. Difference: 17
Requested block size: 100000. Actual block size: 100017. Difference: 17

Я уже тут провел небольшой ресерч, чтобы было все нагляднее. Видим, что поначалу реальный размер блока одинаковый вне зависимости от запрашиваемого размера. И до тех пор, пока не останется 9 "свободных" байт в блоке, его размер не увеличивается. А далее видим, что размер реального блока увеличился на 16 байт. И каждое последующее увеличение блока происходит после того, как там осталось всего 9 байт, на фиксированные 16 байт. И так в принципе и дальше продолжается. Появляется какая-то закономерность. Пойдем дальше копать.

Посмотрим, как будет различаться значение, которое нам выдаст malloc_usable_size , и значение из первых 8 байт перед указателем на блок.

![[Pasted image 20231112115603.png]]

```C++
#include <iostream>
#include <cstring>
#include <memory>
#include <malloc.h>

int main() {
	std::vector<uint64_t> block_sizes = {1, 3, 5, 8, 10, 24, 25, 40, 41, 50, 56, 57, 100, 512, 1000, 10000, 100000};
	for (const auto & size: block_sizes)
	{
		uint64_t * ptr = (uint64_t *)malloc(size);
		std::cout << "Requested block size: " << size <<
					 ". Actual block size: " << *(ptr - 1) <<
					 ". malloc_usable_size: " << malloc_usable_size(ptr) << std::endl;
	}
	free(ptr);
	return 0;
}
```
Вот такие результаты получились:

Requested block size: 1. Actual block size: 33. malloc_usable_size: 24
Requested block size: 3. Actual block size: 33. malloc_usable_size: 24
Requested block size: 5. Actual block size: 33. malloc_usable_size: 24
Requested block size: 8. Actual block size: 33. malloc_usable_size: 24
Requested block size: 10. Actual block size: 33. malloc_usable_size: 24
Requested block size: 24. Actual block size: 33. malloc_usable_size: 24
Requested block size: 25. Actual block size: 49. malloc_usable_size: 40
Requested block size: 40. Actual block size: 49. malloc_usable_size: 40
Requested block size: 41. Actual block size: 65. malloc_usable_size: 56
Requested block size: 50. Actual block size: 65. malloc_usable_size: 56
Requested block size: 56. Actual block size: 65. malloc_usable_size: 56
Requested block size: 57. Actual block size: 81. malloc_usable_size: 72
Requested block size: 100. Actual block size: 113. malloc_usable_size: 104
Requested block size: 512. Actual block size: 529. malloc_usable_size: 520
Requested block size: 1000. Actual block size: 1009. malloc_usable_size: 1000
Requested block size: 10000. Actual block size: 10017. malloc_usable_size: 10008
Requested block size: 100000. Actual block size: 100017. malloc_usable_size: 100008

Еще интереснее. Сразу бросается в глаза, что разница между actual block size и malloc_usable_size постоянна и равна 9. И еще видим, что количество запрашиваемых байт и malloc_usable_size могут совпадать (для 24, 56, 1000). И еще. Количество байт для malloc_usable_size равно 8 + 16*n. 

Делаем выводы:

-Память выделятся с добиванием до кратности 16 (со сдвигом на 8). То есть аллокатор оперирует маленькими блоками по 16 байт. Непонятно, зачем нужно 8 еще, почему что эти 8 байт не могут использоваться для хранения чего-либо, ибо могут быть перезаписаны пользователем(например для чисел 24, 56 и 1000 и всех чисел, меньших их вплоть до 16, 48 и 992).

-В первых 8-ми байтах перед выданным пользователю блоком действительно лежит реальный размер выделенной памяти + 1. Возможно эта единичка - не часть размера, а какой-то внутренний флаг, который о чем-то говорит аллокатору. Нет смысла выделять для этого флага свое место, учитывая, что 4 бита из ячейки с размером блока всегда будут нулевыми (8+8+16n = 16(n+1), количество выделенных байт будет кратно 16, что освобождает 4 бита). И эти первые 8 байт формируют так называемых header, то бишь метаинформацию о выделенном блоке. 

Безусловно, нам не все стало ясно на 100%. Но кое-какие интересные и полезные выводы мы сделать смогли и это уже круто!

P.S.

Я попробовал предыдущий код для довольно больших размеров блоков. Чисто из интереса и для полноты эксперимента. Взял вот такие значения размеров:
```C++
std::vector<uint64_t> block_sizes = {10000, 100000, 1000000, 10000000, 100000000};
```

И получил вот такой вывод:

Requested block size: 10000. Actual block size: 10017. malloc_usable_size: 10008
Requested block size: 100000. Actual block size: 100017. malloc_usable_size: 100008
Requested block size: 1000000. Actual block size: 1003522. malloc_usable_size: 1003504
Requested block size: 10000000. Actual block size: 10002434. malloc_usable_size: 10002416
Requested block size: 100000000. Actual block size: 100003842. malloc_usable_size: 100003824

В какой-то момент что-то пошло не так и мы видим совершенно другую картину. И если честно, то я абсолютно хз, как это объяснить. 

Еще более странные результаты я получаю, когда пытаюсь выяснить, когда в какой именно момент все начинает ломаться. Для этого я написал следующий код:
![[Pasted image 20231112123224.png]]


```C++
#include <iostream>
#include <cstring>
#include <memory>
#include <malloc.h>

int main() {
	for (int size = 100000; size < 600000; ++size)
	{
		uint64_t * ptr = (uint64_t *)malloc(size);
		std::cout << "Requested block size: " << size <<
					 ". Actual block size: " << *(ptr - 1) <<
					 ". malloc_usable_size: " << malloc_usable_size(ptr) << std::endl;
		free(ptr);
	}
	return 0;
}
```
И получил вот такой результат:

Requested block size: 100000. Actual block size: 100017. malloc_usable_size: 100008
Requested block size: 100001. Actual block size: 100017. malloc_usable_size: 100008
Requested block size: 100002. Actual block size: 100017. malloc_usable_size: 100008
Requested block size: 100003. Actual block size: 100017. malloc_usable_size: 100008
Requested block size: 100004. Actual block size: 100017. malloc_usable_size: 100008
Requested block size: 100005. Actual block size: 100017. malloc_usable_size: 100008
Requested block size: 100006. Actual block size: 100017. malloc_usable_size: 100008
Requested block size: 100007. Actual block size: 100017. malloc_usable_size: 100008
Requested block size: 100008. Actual block size: 100017. malloc_usable_size: 100008
Requested block size: 100009. Actual block size: 100033. malloc_usable_size: 100024
...
Requested block size: 599992. Actual block size: 600001. malloc_usable_size: 599992
Requested block size: 599993. Actual block size: 600017. malloc_usable_size: 600008
Requested block size: 599994. Actual block size: 600017. malloc_usable_size: 600008
Requested block size: 599995. Actual block size: 600017. malloc_usable_size: 600008
Requested block size: 599996. Actual block size: 600017. malloc_usable_size: 600008
Requested block size: 599997. Actual block size: 600017. malloc_usable_size: 600008
Requested block size: 599998. Actual block size: 600017. malloc_usable_size: 600008
Requested block size: 599999. Actual block size: 600017. malloc_usable_size: 600008

Чем я это объясню? Деталями реализации)
Никто мне не обещал, что поведение аллокатора должно быть постоянным и предсказуемым. Все зависит от реализации. На самом деле, маллок может и не хранить размер выделенного блока. У него могут быть несколько отдельных списков, которые хранят блоки фиксированной длины и эта длина "привязана" к списку. Тогда маллок может просто брать блоки из нужного списка, который более-менее подходит по размеру, и выдавать его наружу.

Касательно последнего моего примера. Единственное, могу предположить, что в этом случае аллокатор имеет какие-то структуры для выделенных и освобожденных блоков и как-то может переиспользовать ранее выделенные блоки. И это упрощает ему жизнь. Однако вот эту строку я объяснить ничем не могу Requested block size: 100000000. Actual block size: 100003842. malloc_usable_size: 100003824. Скорее всего это материал для будущих изысканий. А может быть вы знаете? Пишите в комментариях свои варианты)

Если вдруг надумаете запускать мой код, то помните, что ваши циферки могут быть не такими, как у меня

Dig deeper. Stay cool.







