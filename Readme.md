# In memory LongLongMap.
Требуется написать LongLongMap который по произвольному long ключу хранить произвольное long значение
Важно: все данные (в том числе дополнительные, если их размер зависит от числа элементов) требуется хранить в выделенном заранее блоке
в разделяемой памяти, адрес и размер которого передается в конструкторе
для доступа к памяти напрямую необходимо (и достаточно) использовать следующие два метода:
sun.misc.Unsafe.getLong(long), sun.misc.Unsafe.putLong(long, long)

# Решение
При первой попытке решить задачу, у меня была идея реализовать что-то схожее с HashMap, то есть, сделать 20 байтовый **Entry**, в котором хранится <key, value, metainfo>.  
При коллизии, помечаем в метаинформации что текущий entry имеет соседние записи, и идем шагом на 20 байт пока не найдем свободную ячейку памяти, выходит что-то схожее со связным списком.  
У меня было предположение что операции **put** и **get** будут работать с наихудшей сложность O(n), поэтому я решил найти оптимальный алгоритм хэширования (получения адреса памяти из ключа), который будет иметь равномерное распределение.  

В ходе чтения 11 главы (Хеширование и хеш таблицы) книги "Алгоритмы построение и анализ", я заметил, что задача хорошо подходит под метод **Открытой адресации**.  
При использовании этого метода все элементы хранятся в хеш-таблице (в нашем случае это выделенная память) и каждая запись таблицы содержит либо элемент диманического множества (у нас это 16 байтовый **Entry<key, value>**), либо значение NIL (в нашем случае 0).  
В отличии от метода цепочек, нам не нужны ни списки, ни доп указатели. Так же, мы можем сохранить больше данных при том же количестве памяти, так как уменьшили размер **Entry** с 20 до 16 байтов, потенциально приводя к меньшему количеству коллизий и более быстрой выборке, так как мы вместо того чтобы следовать по указателям, вычисляем последовательность проверяемых ячеек.  

## Вставка
Для выполнения вставки при открытой адресации мы последовательно проверяем, или исследуем (probe), ячейки хеш-таблицы до тех пор, пока не находим пустую ячейку, в которую помещаем вставляемый ключ.  
Вместо фиксированного порядка исследования ячеек 0, 1, ..., m - 1 (для чего нам требуется O(n) времени), последовательность зависит от вставляемого в него ключа.  
Для определения исследуемых ячеек мы добавим в хеш-функцию аргумент - номер исследования (который начинается с 0).  

## Получение
Алгоритм поиска ключа **k** исследует ту же последовательность ячеек, что и алгоритм вставки ключа **k**.  
Таким образом, если при поиске встречается пусткая ячейка, поиск завершается неудачей, посколькку ключ **k** должен был бы быть вставлен в эту ячейку в последовательности исследований, и никак не позже нее.

## Выбор хеш-функции
Важнейшей частью реализации таблицы является хеш-функция. Есть три основных метода вычисления последовательности исследований при открытой адресации:
1. Линейное исследование
2. Квадратичное исследование
3. Двойное хеширование

**Двойное хеширование** требует взаимную простоту одной из хеш функции с размером таблицы. Отбрасываем, так как размер таблицы зависит от выделяемой памяти, которая задается извне, и мы не можем всегда удовлетворять это условие.  

**Линейное исследование** имеет вид: **h(k,i) = (haux(k) + i) mod m**, где m - размер таблицы.  
В нашем случае, ```haux(ключ) = адрес_начала_памяти_таблицы + (ключ % вместимость) * ENTRY_SIZE```.  
С помощью этой функции мы можем получить адрес первой исследуемой ячейки T[haux(ключ)], далее, при необходимости по условию, мы можем перейти к ячейке T[haux(ключ) + 1] до T[размер_таблицы - 1], после чего переходит в начало таблицы и последовательно исследуем ячейки Т[0], T[1] до T[haux(ключ) - 1].  

**Квадратичное исследование** можно представить как улучшенную версию линейного исследования, с помощью правильно подобранных констант с1, с2, так же, размер таблицы тоже должен быть правильно подобран. Снова, так как размер таблицы зависит от выделяемой памяти, которая задается извне, мы не можем всегда удовлетворять это условие.  

**Методом исключения мы выбираем Линейное исследование в качестве нашей хэш функции**.

## Анализ
Математическое ожидание количества исследований при неудачном поиске в хеш-таблице с открытой адресацией и коэффициентом заполнения a = n/m < 1 в преположении равномерного хеширования не превышает 1/(1-a).  
Где n - количество записанных элементов в таблице, m - размер таблицы.  
Из этого мы можем увидить, что неудачный поиск выполняется за время О(1)

Примеры:  
```a = 0.2 -> 1/0.8 = 1.25 Нам понадобиться 1.25 исследований для вставки в нашу таблицу при коэфф. заполнения 0.2 ```  
```a = 0.6 -> 1/0.4 = 2.5 Нам понадобиться 2.5 исследований для вставки в нашу таблицу при коэфф. заполнения 0.6```   
Из примеров понятно что, чем больше у нас элементов в таблице, тем больше исследований нам понадобится для одной вставки.  

