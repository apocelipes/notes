open code就是指不需要runtime介入，在每个函数的退出点都自动执行defer指定的函数。（panic的时候还是需要runtime的）这样开销很小，就像自己在每个return前写函数调用语句一样。

open code defer的数据都分配在栈上。

for-loop以及迭代器里的defer分配在堆上，且会加入runtime维护的defer链表，由runtime负责在函数退出前执行。

如果函数里出现分配在堆的defer，则其他open code defer不再open，调用也由runtime负责，但仍然分配在堆上。

defer是否在if/switch里不会影响它open。
