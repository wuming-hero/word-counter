如题：现在电子图书馆中有10亿本电子书，这都是英文书。这些英文书有书名和出版日期两个属性，请实现以下功能：

1. 我们想要计算出来整个电子图书馆中每个英文单词出现的次数，比如老人与海这本书里，有my home这样的，我需要计算出my和home这些单词出现的次数。
2. 并且在大屏幕上"实时"展示计算结果。也就是说我在计算过程中的任何时间，都有可能需要看当前每个英文单词出现的次数。
3. 现在由于一些原因，电子图书馆中的书还在增加。

请设计你的程序去实现这个功能，可以不用做屏幕展示的能力。在设计过程中，请考虑分区(分库分表)，多线程，性能，线程安全，业务一致性等非功能性因素。可以在注释中加上您自己认为需要关注的点，以及你的设计思路。

## 设计思路：
1. 全局文件队列及增加文件监听放入列表
通过`FileVisitor`获取全部文件路径放入线程安全的无界队列`ConcurrentLinkedQueue<String> filePathQueue`中；对于增量数据，通过`java.nio.file.WatchService`监听目标文件夹，如果有新增的有效的目标文件，通过`filePathQueue.offer(T)`追回到队列尾部。使用多线程通过`filePathQueue.poll()`方法获取文件路径，构建`BigFileReader`对象，调用`BigFileReader.countWord()`进行统计。
2. 大文件读取类`BigFileReader`
`BigFileReader`类的主要方法分为切片和切片读取统计两个方法。文件大小和线程数量对文件进切片，则每个切片大小约等于`file.length/thread.size`。
使用多线程对每个切片读取，每个切片读取的内容单独放入`ConcurrentHashMap<String, AtomicLong> sliceWordCountMap`，key为单词，value为线程安全的次数。每个切片统计成功，放入全局统计队列`ConcurrentLinkedQueue<ConcurrentHashMap<String, AtomicLong>> wordCountQueue`中；如果统计失败，则放入`failSliceSet`集合中，每个文件多线程操作通过`CountDownLatch`进行统一管理，当前文件的全部线程结束后，关闭文件注，处理队列。将失败的切片信息放入失败队列`ConcurrentLinkedQueue<BigFileReader> fileFailSliceQueue`。
3. 失败的切片队列`ConcurrentLinkedQueue<BigFileReader> fileFailSliceQueue`处理  
别起线程对失败的切片队列`ConcurrentLinkedQueue<BigFileReader> fileFailSliceQueue`进行消费，清理费的主要功能还是调用`BigFileReader.countWordForSlice()`进行统计。
4. 全局统计队列`ConcurrentLinkedQueue<ConcurrentHashMap<String, AtomicLong>> wordCountQueue` 处理
`wordCountQueue`每个元素为文件切片的统计信息，对之进行遍历，与全局统一数据统计集合`ConcurrentHashMap<String, AtomicLong> wordCountMap`进行合并，则`wordCountMap`中的数据即为我们的目标数据。

## 具体主要类及功能说明
#### 1. 通用全局文件处理类`FileQueueConsumer`及文件监听器
1. 基于 `FileVisitor` 查找指定文件路径下所有有效文件，将文件路径放入无界链表队列`ConcurrentLinkedQueue<String> filePathQueue`
2. 声名实现`Runnable`接口的`FileQueueConsumer`类，该消费类的主要功能是根据文件路径构建`BigFileReader`,调用该类的`countWord`方法进行统计单词
3. 声明一个固定10个线程的线程池，多线程方式提交`FileQueueConsumer`进行消费
4. 主线程基于`WatchService`注册文件夹新增文件监听类，如果监听到新增有效文件，放入`filePathQueue`

#### 2. 大文件多线程分片读取类`BigFileReader` 
1. 根据文件大小和线程数量对文件进行切片，默认使用10个线程。则每个切片大小约等于`file.length/thread.size`,为了防止单词被分开读取，每个切片的`end`往后读取到换行为止，剩下的大小全部给最后一个切片。
2. 每个切片按指定的`bufferSize`进行读取，这样按指定size大小读，读出来数据也可能是半个单词，所以对读取出来的内容循环判断，如果里面有换行符，则直接通过`ByteArrayOutputStream`输出，如果没有，判断完后一起输出。
3. 将`ByteArrayOutputStream`输出的字节转为String字符串，通过正则表达式`Pattern.compile("\\b\\w+\\b").matcher(line)`进行单词分隔,使用线程安全的`ConcurrentHashMap`进行单词次数统计。
4. 每个文件切片如上读取，切片读取成功，则将该切片的统计数据放入全局统计队列`WordCountQueue`;若切片读取失败，则使用Set记录下来，使用`CountDownLatch`对线程执行失败的结果统一处理，放入`FileFailSliceQueueConsumer`;
5. 当前文件操作完成，文件流关闭；线程池关闭；

#### 3. 处理失败切片队列`WordCountQueue`及对应消费类`FileFailSliceQueueConsumer`及
1. `WordCountQueue`每个记录为统计异常的文件信息和失败的切片Set
2. `FileFailSliceQueueConsumer` 异步循环消费`WordCountQueue`的信息
3. 获取到文件信息后，构建`BigFileReader` 并且设置指定要统计的失败切片信息，直接调用`BigFileReader.countWordForSlice()`对失败的切片进行统计。统计结果按正常读取处理。

#### 4. 全局数据统计队列`WordCountQueue`及对应的消费类 `WordCountQueueConsumer`
1. `WordCountQueue`队列中的每个元素为`ConcurrentHashMap<String, AtomicLong>`，数据为单个文件切片的单词统计数据, key 为单词内容，value为原子安全的次数。
2. `WordCountQueueConsumer`中循环遍历单个切片的单词统计数据，总统计数据`ConcurrentHashMap<String, AtomicLong> wordCountMap`通过key进行判断，如果该单词已存在，则使用线程安全的`addAndGet()`进行增加次数；如果单词不存在，调用`putIfAbsent`进行初始化。如此，则`wordCountMap`中的数据则为目标数据。直接获取展示即可。