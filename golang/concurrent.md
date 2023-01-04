*  waitgroup add 必须要在 go func() 的外面, 保证 happen before waitgroup wait().
* break in select means break out in one loop, 如果外面还有 for loop, 并不能跳出, 需要一个标志位. 


# sync map
sync map 实际是以空间换时间, 适合读多写少的场景, 有read 和dirty 两个map 结构. 读取不需要加锁. 
