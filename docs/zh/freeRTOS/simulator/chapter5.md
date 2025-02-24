# 第五章 队列(queue)

队列(queue)可以用于"任务到任务"、"任务到中断"、"中断到任务"直接传输信息。

本章涉及如下内容：

* 怎么创建、清除、删除队列
* 队列中消息如何保存
* 怎么向队列发送数据、怎么从队列读取数据、怎么覆盖队列的数据
* 在队列上阻塞是什么意思
* 怎么在多个队列上阻塞
* 读写队列时如何影响任务的优先级

## 5.1 队列的特性

### 5.1.1 常规操作

队列的简化操如入下图所示，从此图可知：

* 队列可以包含若干个数据：队列中有若干项，这被称为"长度"(length)
* 每个数据大小固定
* 创建队列时就要指定长度、数据大小
* 数据的操作采用先进先出的方法(FIFO，First In First Out)：写数据时放到尾部，读数据时从头部读
* 也可以强制写队列头部：覆盖头部数据

![image-20210802235434145](http://photos.100ask.net/rtos-docs/freeRTOS/simulator/chapter-5/02_queue.png)



更详细的操作入下图所示：

![image-20210803001909785](http://photos.100ask.net/rtos-docs/freeRTOS/simulator/chapter-5/03_queue_detail.png)



### 5.1.2 传输数据的两种方法

使用队列传输数据时有两种方法：

* 拷贝：把数据、把变量的值复制进队列里
* 引用：把数据、把变量的地址复制进队列里

FreeRTOS使用拷贝值的方法，这更简单：

* 局部变量的值可以发送到队列中，后续即使函数退出、局部变量被回收，也不会影响队列中的数据

* 无需分配buffer来保存数据，队列中有buffer

* 局部变量可以马上再次使用

* 发送任务、接收任务解耦：接收任务不需要知道这数据是谁的、也不需要发送任务来释放数据

* 如果数据实在太大，你还是可以使用队列传输它的地址

* 队列的空间有FreeRTOS内核分配，无需任务操心

* 对于有内存保护功能的系统，如果队列使用引用方法，也就是使用地址，必须确保双方任务对这个地址都有访问权限。使用拷贝方法时，则无此限制：内核有足够的权限，把数据复制进队列、再把数据复制出队列。



### 5.1.3 队列的阻塞访问

只要知道队列的句柄，谁都可以读、写该队列。任务、ISR都可读、写队列。可以多个任务读写队列。

任务读写队列时，简单地说：如果读写不成功，则阻塞；可以指定超时时间。口语化地说，就是可以定个闹钟：如果能读写了就马上进入就绪态，否则就阻塞直到超时。

某个任务读队列时，如果队列没有数据，则该任务可以进入阻塞状态：还可以指定阻塞的时间。如果队列有数据了，则该阻塞的任务会变为就绪态。如果一直都没有数据，则时间到之后它也会进入就绪态。

既然读取队列的任务个数没有限制，那么当多个任务读取空队列时，这些任务都会进入阻塞状态：有多个任务在等待同一个队列的数据。当队列中有数据时，哪个任务会进入就绪态？

* 优先级最高的任务
* 如果大家的优先级相同，那等待时间最久的任务会进入就绪态



跟读队列类似，一个任务要写队列时，如果队列满了，该任务也可以进入阻塞状态：还可以指定阻塞的时间。如果队列有空间了，则该阻塞的任务会变为就绪态。如果一直都没有空间，则时间到之后它也会进入就绪态。

既然写队列的任务个数没有限制，那么当多个任务写"满队列"时，这些任务都会进入阻塞状态：有多个任务在等待同一个队列的空间。当队列中有空间时，哪个任务会进入就绪态？

* 优先级最高的任务
* 如果大家的优先级相同，那等待时间最久的任务会进入就绪态



## 5.2 队列函数

使用队列的流程：创建队列、写队列、读队列、删除队列。

### 5.2.1 创建

队列的创建有两种方法：动态分配内存、静态分配内存，

* 动态分配内存：xQueueCreate，队列的内存在函数内部动态分配

函数原型如下：

```c
QueueHandle_t xQueueCreate( UBaseType_t uxQueueLength, UBaseType_t uxItemSize );
```

| 参数          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| uxQueueLength | 队列长度，最多能存放多少个数据(item)                         |
| uxItemSize    | 每个数据(item)的大小：以字节为单位                           |
| 返回值        | 非0：成功，返回句柄，以后使用句柄来操作队列<br />NULL：失败，因为内存不足 |



* 静态分配内存：xQueueCreateStatic，队列的内存要事先分配好

函数原型如下：

```c
QueueHandle_t xQueueCreateStatic(
                           UBaseType_t uxQueueLength,
                           UBaseType_t uxItemSize,
                           uint8_t *pucQueueStorageBuffer,
                           StaticQueue_t *pxQueueBuffer
                       );
```

| 参数                  | 说明                                                         |
| --------------------- | ------------------------------------------------------------ |
| uxQueueLength         | 队列长度，最多能存放多少个数据(item)                         |
| uxItemSize            | 每个数据(item)的大小：以字节为单位                           |
| pucQueueStorageBuffer | 如果uxItemSize非0，pucQueueStorageBuffer必须指向一个uint8_t数组，<br />此数组大小至少为"uxQueueLength * uxItemSize" |
| pxQueueBuffer         | 必须执行一个StaticQueue_t结构体，用来保存队列的数据结构      |
| 返回值                | 非0：成功，返回句柄，以后使用句柄来操作队列<br />NULL：失败，因为pxQueueBuffer为NULL |



示例代码：

```c
// 示例代码
 #define QUEUE_LENGTH 10
 #define ITEM_SIZE sizeof( uint32_t )
 
 // xQueueBuffer用来保存队列结构体
 StaticQueue_t xQueueBuffer;
 
 // ucQueueStorage 用来保存队列的数据
 // 大小为：队列长度 * 数据大小
 uint8_t ucQueueStorage[ QUEUE_LENGTH * ITEM_SIZE ];
 
 void vATask( void *pvParameters )
 {
	QueueHandle_t xQueue1;
 
	// 创建队列: 可以容纳QUEUE_LENGTH个数据，每个数据大小是ITEM_SIZE
	xQueue1 = xQueueCreateStatic( QUEUE_LENGTH,
						  ITEM_SIZE,
						  ucQueueStorage,
						  &xQueueBuffer ); 
 }
```



### 5.2.2 复位

队列刚被创建时，里面没有数据；使用过程中可以调用`xQueueReset()`把队列恢复为初始状态，此函数原型为：

```c
/* pxQueue : 复位哪个队列;
 * 返回值: pdPASS(必定成功)
 */
BaseType_t xQueueReset( QueueHandle_t pxQueue);
```



### 5.2.3 删除

删除队列的函数为`vQueueDelete()`，只能删除使用动态方法创建的队列，它会释放内存。原型如下：

```c
void vQueueDelete( QueueHandle_t xQueue );
```



### 5.2.4 写队列

可以把数据写到队列头部，也可以写到尾部，这些函数有两个版本：在任务中使用、在ISR中使用。函数原型如下：

```c
/* 等同于xQueueSendToBack
 * 往队列尾部写入数据，如果没有空间，阻塞时间为xTicksToWait
 */
BaseType_t xQueueSend(
                                QueueHandle_t    xQueue,
                                const void       *pvItemToQueue,
                                TickType_t       xTicksToWait
                            );

/* 
 * 往队列尾部写入数据，如果没有空间，阻塞时间为xTicksToWait
 */
BaseType_t xQueueSendToBack(
                                QueueHandle_t    xQueue,
                                const void       *pvItemToQueue,
                                TickType_t       xTicksToWait
                            );


/* 
 * 往队列尾部写入数据，此函数可以在中断函数中使用，不可阻塞
 */
BaseType_t xQueueSendToBackFromISR(
                                      QueueHandle_t xQueue,
                                      const void *pvItemToQueue,
                                      BaseType_t *pxHigherPriorityTaskWoken
                                   );

/* 
 * 往队列头部写入数据，如果没有空间，阻塞时间为xTicksToWait
 */
BaseType_t xQueueSendToFront(
                                QueueHandle_t    xQueue,
                                const void       *pvItemToQueue,
                                TickType_t       xTicksToWait
                            );

/* 
 * 往队列头部写入数据，此函数可以在中断函数中使用，不可阻塞
 */
BaseType_t xQueueSendToFrontFromISR(
                                      QueueHandle_t xQueue,
                                      const void *pvItemToQueue,
                                      BaseType_t *pxHigherPriorityTaskWoken
                                   );
```



这些函数用到的参数是类似的，统一说明如下：

| 参数          | 说明                                                         |
| ------------- | ------------------------------------------------------------ |
| xQueue        | 队列句柄，要写哪个队列                                       |
| pvItemToQueue | 数据指针，这个数据的值会被复制进队列，<br />复制多大的数据？在创建队列时已经指定了数据大小 |
| xTicksToWait  | 如果队列满则无法写入新数据，可以让任务进入阻塞状态，<br />xTicksToWait表示阻塞的最大时间(Tick Count)。<br />如果被设为0，无法写入数据时函数会立刻返回；<br />如果被设为portMAX_DELAY，则会一直阻塞直到有空间可写 |
| 返回值        | pdPASS：数据成功写入了队列<br />errQUEUE_FULL：写入失败，因为队列满了。 |



### 5.2.5 读队列

使用`xQueueReceive()`函数读队列，读到一个数据后，队列中该数据会被移除。这个函数有两个版本：在任务中使用、在ISR中使用。函数原型如下：

```c
BaseType_t xQueueReceive( QueueHandle_t xQueue,
                          void * const pvBuffer,
                          TickType_t xTicksToWait );

BaseType_t xQueueReceiveFromISR(
                                    QueueHandle_t    xQueue,
                                    void             *pvBuffer,
                                    BaseType_t       *pxTaskWoken
                                );
```

参数说明如下：

| 参数         | 说明                                                         |
| ------------ | ------------------------------------------------------------ |
| xQueue       | 队列句柄，要读哪个队列                                       |
| pvBuffer     | bufer指针，队列的数据会被复制到这个buffer<br />复制多大的数据？在创建队列时已经指定了数据大小 |
| xTicksToWait | 果队列空则无法读出数据，可以让任务进入阻塞状态，<br />xTicksToWait表示阻塞的最大时间(Tick Count)。<br />如果被设为0，无法读出数据时函数会立刻返回；<br />如果被设为portMAX_DELAY，则会一直阻塞直到有数据可写 |
| 返回值       | pdPASS：从队列读出数据入<br />errQUEUE_EMPTY：读取失败，因为队列空了。 |



### 5.2.6 查询

可以查询队列中有多少个数据、有多少空余空间。函数原型如下：

```c
/*
 * 返回队列中可用数据的个数
 */
UBaseType_t uxQueueMessagesWaiting( const QueueHandle_t xQueue );

/*
 * 返回队列中可用空间的个数
 */
UBaseType_t uxQueueSpacesAvailable( const QueueHandle_t xQueue );
```



### 5.2.7 覆盖/偷看

当队列长度为1时，可以使用`xQueueOverwrite()`或`xQueueOverwriteFromISR()`来覆盖数据。
注意，队列长度必须为1。当队列满时，这些函数会覆盖里面的数据，这也以为着这些函数不会被阻塞。
函数原型如下：

```c
/* 覆盖队列
 * xQueue: 写哪个队列
 * pvItemToQueue: 数据地址
 * 返回值: pdTRUE表示成功, pdFALSE表示失败
 */
BaseType_t xQueueOverwrite(
                           QueueHandle_t xQueue,
                           const void * pvItemToQueue
                      );

BaseType_t xQueueOverwriteFromISR(
                           QueueHandle_t xQueue,
                           const void * pvItemToQueue,
                           BaseType_t *pxHigherPriorityTaskWoken
                      );
```



如果想让队列中的数据供多方读取，也就是说读取时不要移除数据，要留给后来人。那么可以使用"窥视"，也就是`xQueuePeek()`或`xQueuePeekFromISR()`。这些函数会从队列中复制出数据，但是不移除数据。这也意味着，如果队列中没有数据，那么"偷看"时会导致阻塞；一旦队列中有数据，以后每次"偷看"都会成功。
函数原型如下：

```c
/* 偷看队列
 * xQueue: 偷看哪个队列
 * pvItemToQueue: 数据地址, 用来保存复制出来的数据
 * xTicksToWait: 没有数据的话阻塞一会
 * 返回值: pdTRUE表示成功, pdFALSE表示失败
 */
BaseType_t xQueuePeek(
                          QueueHandle_t xQueue,
                          void * const pvBuffer,
                          TickType_t xTicksToWait
                      );

BaseType_t xQueuePeekFromISR(
                                 QueueHandle_t xQueue,
                                 void *pvBuffer,
                             );
```









## 5.3 示例8: 队列的基本使用

本节代码为：`FreeRTOS_08_queue`。

本程序会创建一个队列，然后创建2个发送任务、1个接收任务：

* 发送任务优先级为1，分别往队列中写入100、200
* 接收任务优先级为2，读队列、打印数值



main函数中创建的队列、创建了发送任务、接收任务，代码如下：

```c
/* 队列句柄, 创建队列时会设置这个变量 */
QueueHandle_t xQueue;

int main( void )
{
	prvSetupHardware();
	
    /* 创建队列: 长度为5，数据大小为4字节(存放一个整数) */
    xQueue = xQueueCreate( 5, sizeof( int32_t ) );

	if( xQueue != NULL )
	{
		/* 创建2个任务用于写队列, 传入的参数分别是100、200
		 * 任务函数会连续执行，向队列发送数值100、200
		 * 优先级为1
		 */
		xTaskCreate( vSenderTask, "Sender1", 1000, ( void * ) 100, 1, NULL );
		xTaskCreate( vSenderTask, "Sender2", 1000, ( void * ) 200, 1, NULL );

		/* 创建1个任务用于读队列
		 * 优先级为2, 高于上面的两个任务
		 * 这意味着队列一有数据就会被读走
		 */
		xTaskCreate( vReceiverTask, "Receiver", 1000, NULL, 2, NULL );

		/* 启动调度器 */
		vTaskStartScheduler();
	}
	else
	{
		/* 无法创建队列 */
	}

	/* 如果程序运行到了这里就表示出错了, 一般是内存不足 */
	return 0;
}
```



发送任务的函数中，不断往队列中写入数值，代码如下：

```c
static void vSenderTask( void *pvParameters )
{
	int32_t lValueToSend;
	BaseType_t xStatus;

	/* 我们会使用这个函数创建2个任务
	 * 这些任务的pvParameters不一样
 	 */
	lValueToSend = ( int32_t ) pvParameters;

	/* 无限循环 */
	for( ;; )
	{
		/* 写队列
		 * xQueue: 写哪个队列
		 * &lValueToSend: 写什么数据? 传入数据的地址, 会从这个地址把数据复制进队列
		 * 0: 不阻塞, 如果队列满的话, 写入失败, 立刻返回
		 */
		xStatus = xQueueSendToBack( xQueue, &lValueToSend, 0 );

		if( xStatus != pdPASS )
		{
			printf( "Could not send to the queue.\r\n" );
		}
	}
}
```



接收任务的函数中，读取队列、判断返回值、打印，代码如下：

```c
static void vReceiverTask( void *pvParameters )
{
	/* 读取队列时, 用这个变量来存放数据 */
	int32_t lReceivedValue;
	BaseType_t xStatus;
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 100UL );

	/* 无限循环 */
	for( ;; )
	{
		/* 读队列
		 * xQueue: 读哪个队列
		 * &lReceivedValue: 读到的数据复制到这个地址
		 * xTicksToWait: 如果队列为空, 阻塞一会
		 */
		xStatus = xQueueReceive( xQueue, &lReceivedValue, xTicksToWait );

		if( xStatus == pdPASS )
		{
			/* 读到了数据 */
			printf( "Received = %d\r\n", lReceivedValue );
		}
		else
		{
			/* 没读到数据 */
			printf( "Could not receive from the queue.\r\n" );
		}
	}
}
```



程序运行结果如下：

![image-20210803145100540](http://photos.100ask.net/rtos-docs/freeRTOS/simulator/chapter-5/04_queue_read_example.png)

任务调度情况如下图所示：

![image-20210803150702562](http://photos.100ask.net/rtos-docs/freeRTOS/simulator/chapter-5/05_queue_read_schedule.png)



## 5.4 示例9: 分辨数据源

本节代码为：`FreeRTOS_09_queue_datasource`。

当有多个发送任务，通过同一个队列发出数据，接收任务如何分辨数据来源？数据本身带有"来源"信息，比如写入队列的数据是一个结构体，结构体中的lDataSouceID用来表示数据来源：

```c
typedef struct {
    ID_t eDataID;
    int32_t lDataValue;
}Data_t;
```

不同的发送任务，先构造好结构体，填入自己的`eDataID`，再写队列；接收任务读出数据后，根据`eDataID`就可以知道数据来源了，如下图所示：

* CAN任务发送的数据：eDataID=eMotorSpeed
* HMI任务发送的数据：eDataID=eSpeedSetPoint

![image-20210803163949972](http://photos.100ask.net/rtos-docs/freeRTOS/simulator/chapter-5/06_data_contain_source.png)



`FreeRTOS_09_queue_datasource`程序会创建一个队列，然后创建2个发送任务、1个接收任务：

* 创建的队列，用来发送结构体：数据大小是结构体的大小
* 发送任务优先级为2，分别往队列中写入自己的结构体，结构体中会标明数据来源
* 接收任务优先级为1，读队列、根据数据来源打印信息



main函数中创建了队列、创建了发送任务、接收任务，代码如下：

```c
/* 定义2种数据来源(ID) */
typedef enum
{
	eMotorSpeed,
	eSpeedSetPoint
} ID_t;

/* 定义在队列中传输的数据的格式 */
typedef struct {
    ID_t eDataID;
    int32_t lDataValue;
}Data_t;

/* 定义2个结构体 */
static const Data_t xStructsToSend[ 2 ] =
{
	{ eMotorSpeed,    10 }, /* CAN任务发送的数据 */
	{ eSpeedSetPoint, 5 }   /* HMI任务发送的数据 */
};

/* vSenderTask被用来创建2个任务，用于写队列
 * vReceiverTask被用来创建1个任务，用于读队列
 */
static void vSenderTask( void *pvParameters );
static void vReceiverTask( void *pvParameters );

/*-----------------------------------------------------------*/

/* 队列句柄, 创建队列时会设置这个变量 */
QueueHandle_t xQueue;

int main( void )
{
	prvSetupHardware();
	
    /* 创建队列: 长度为5，数据大小为4字节(存放一个整数) */
    xQueue = xQueueCreate( 5, sizeof( Data_t ) );

	if( xQueue != NULL )
	{
		/* 创建2个任务用于写队列, 传入的参数是不同的结构体地址
		 * 任务函数会连续执行，向队列发送结构体
		 * 优先级为2
		 */
		xTaskCreate(vSenderTask, "CAN Task", 1000, (void *) &(xStructsToSend[0]), 2, NULL);
		xTaskCreate(vSenderTask, "HMI Task", 1000, (void *) &( xStructsToSend[1]), 2, NULL);

		/* 创建1个任务用于读队列
		 * 优先级为1, 低于上面的两个任务
		 * 这意味着发送任务优先写队列，队列常常是满的状态
		 */
		xTaskCreate( vReceiverTask, "Receiver", 1000, NULL, 1, NULL );

		/* 启动调度器 */
		vTaskStartScheduler();
	}
	else
	{
		/* 无法创建队列 */
	}

	/* 如果程序运行到了这里就表示出错了, 一般是内存不足 */
	return 0;
}
```



发送任务的函数中，不断往队列中写入数值，代码如下：

```c
static void vSenderTask( void *pvParameters )
{
	BaseType_t xStatus;
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 100UL );

	/* 无限循环 */
	for( ;; )
	{
		/* 写队列
		 * xQueue: 写哪个队列
		 * pvParameters: 写什么数据? 传入数据的地址, 会从这个地址把数据复制进队列
		 * xTicksToWait: 如果队列满的话, 阻塞一会
		 */
		xStatus = xQueueSendToBack( xQueue, pvParameters, xTicksToWait );

		if( xStatus != pdPASS )
		{
			printf( "Could not send to the queue.\r\n" );
		}
	}
}
```



接收任务的函数中，读取队列、判断返回值、打印，代码如下：

```c
static void vReceiverTask( void *pvParameters )
{
	/* 读取队列时, 用这个变量来存放数据 */
	Data_t xReceivedStructure;
	BaseType_t xStatus;

	/* 无限循环 */
	for( ;; )
	{
		/* 读队列
		 * xQueue: 读哪个队列
		 * &xReceivedStructure: 读到的数据复制到这个地址
		 * 0: 没有数据就即刻返回，不阻塞
		 */
		xStatus = xQueueReceive( xQueue, &xReceivedStructure, 0 );

		if( xStatus == pdPASS )
		{
			/* 读到了数据 */
			if( xReceivedStructure.eDataID == eMotorSpeed )
			{
				printf( "From CAN, MotorSpeed = %d\r\n", xReceivedStructure.lDataValue );
			}
			else if( xReceivedStructure.eDataID == eSpeedSetPoint )
			{
				printf( "From HMI, SpeedSetPoint = %d\r\n", xReceivedStructure.lDataValue );
			}
		}
		else
		{
			/* 没读到数据 */
			printf( "Could not receive from the queue.\r\n" );
		}
	}
}
```



运行结果如下：

![image-20210803170521040](http://photos.100ask.net/rtos-docs/freeRTOS/simulator/chapter-5/07_queue_data_with_struct.png)



任务调度情况如下图所示：

* t1：HMI是最后创建的最高优先级任务，它先执行，一下子向队列写入5个数据，把队列都写满了
* t2：队列已经满了，HMI任务再发起第6次写操作时，进入阻塞状态。这时CAN任务是最高优先级的就绪态任务，它开始执行
* t3：CAN任务发现队列已经满了，进入阻塞状态；接收任务变为最高优先级的就绪态任务，它开始运行
* t4：现在，HMI任务、CAN任务的优先级都比接收任务高，它们都在等待队列有空闲的空间；一旦接收任务读出1个数据，会马上被抢占。被谁抢占？谁等待最久？HMI任务！所以在t4时刻，切换到HMI任务。
* t5：HMI任务向队列写入第6个数据，然后再次阻塞，这是CAN任务已经阻塞很久了。接收任务变为最高优先级的就绪态任务，开始执行。
* t6：现在，HMI任务、CAN任务的优先级都比接收任务高，它们都在等待队列有空闲的空间；一旦接收任务读出1个数据，会马上被抢占。被谁抢占？谁等待最久？CAN任务！所以在t6时刻，切换到CAN任务。
* t7：CAN任务向队列写入数据，因为仅仅有一个空间供写入，所以它马上再次进入阻塞状态。这时HMI任务、CAN任务都在等待空闲空间，只有接收任务可以继续执行。

![image-20210803170859100](http://photos.100ask.net/rtos-docs/freeRTOS/simulator/chapter-5/08_queue_data_with_struct_schedule.png)

## 5.5 示例10: 传输大块数据

本节代码为：`FreeRTOS_10_queue_bigtransfer`。

FreeRTOS的队列使用拷贝传输，也就是要传输uint32_t时，把4字节的数据拷贝进队列；要传输一个8字节的结构体时，把8字节的数据拷贝进队列。

如果要传输1000字节的结构体呢？写队列时拷贝1000字节，读队列时再拷贝1000字节？不建议这么做，影响效率！

这时候，我们要传输的是这个巨大结构体的地址：把它的地址写入队列，对方从队列得到这个地址，使用地址去访问那1000字节的数据。

使用地址来间接传输数据时，这些数据放在RAM里，对于这块RAM，要保证这几点：

* RAM的所有者、操作者，必须清晰明了
  这块内存，就被称为"共享内存"。要确保不能同时修改RAM。比如，在写队列之前只有由发送者修改这块RAM，在读队列之后只能由接收者访问这块RAM。
* RAM要保持可用
  这块RAM应该是全局变量，或者是动态分配的内存。对于动然分配的内存，要确保它不能提前释放：要等到接收者用完后再释放。另外，不能是局部变量。



`FreeRTOS_10_queue_bigtransfer`程序会创建一个队列，然后创建1个发送任务、1个接收任务：

* 创建的队列：长度为1，用来传输"char *"指针
* 发送任务优先级为1，在字符数组中写好数据后，把它的地址写入队列
* 接收任务优先级为2，读队列得到"char *"值，把它打印出来

这个程序故意设置接收任务的优先级更高，在它访问数组的过程中，接收任务无法执行、无法写这个数组。



main函数中创建了队列、创建了发送任务、接收任务，代码如下：

```c
/* 定义一个字符数组 */
static char pcBuffer[100];


/* vSenderTask被用来创建2个任务，用于写队列
 * vReceiverTask被用来创建1个任务，用于读队列
 */
static void vSenderTask( void *pvParameters );
static void vReceiverTask( void *pvParameters );

/*-----------------------------------------------------------*/

/* 队列句柄, 创建队列时会设置这个变量 */
QueueHandle_t xQueue;

int main( void )
{
	prvSetupHardware();
	
    /* 创建队列: 长度为1，数据大小为4字节(存放一个char指针) */
    xQueue = xQueueCreate( 1, sizeof(char *) );

	if( xQueue != NULL )
	{
		/* 创建1个任务用于写队列
		 * 任务函数会连续执行，构造buffer数据，把buffer地址写入队列
		 * 优先级为1
		 */
		xTaskCreate( vSenderTask, "Sender", 1000, NULL, 1, NULL );

		/* 创建1个任务用于读队列
		 * 优先级为2, 高于上面的两个任务
		 * 这意味着读队列得到buffer地址后，本任务使用buffer时不会被打断
		 */
		xTaskCreate( vReceiverTask, "Receiver", 1000, NULL, 2, NULL );

		/* 启动调度器 */
		vTaskStartScheduler();
	}
	else
	{
		/* 无法创建队列 */
	}

	/* 如果程序运行到了这里就表示出错了, 一般是内存不足 */
	return 0;
}
```



发送任务的函数中，现在全局大数组pcBuffer中构造数据，然后把它的地址写入队列，代码如下：

```c
static void vSenderTask( void *pvParameters )
{
	BaseType_t xStatus;
	static int cnt = 0;
	
	char *buffer;

	/* 无限循环 */
	for( ;; )
	{
		sprintf(pcBuffer, "www.100ask.net Msg %d\r\n", cnt++);
		buffer = pcBuffer; // buffer变量等于数组的地址, 下面要把这个地址写入队列
		
		/* 写队列
		 * xQueue: 写哪个队列
		 * pvParameters: 写什么数据? 传入数据的地址, 会从这个地址把数据复制进队列
		 * 0: 如果队列满的话, 即刻返回
		 */
		xStatus = xQueueSendToBack( xQueue, &buffer, 0 ); /* 只需要写入4字节, 无需写入整个buffer */

		if( xStatus != pdPASS )
		{
			printf( "Could not send to the queue.\r\n" );
		}
	}
}
```



接收任务的函数中，读取队列、得到buffer的地址、打印，代码如下：

```c
static void vReceiverTask( void *pvParameters )
{
	/* 读取队列时, 用这个变量来存放数据 */
	char *buffer;
	const TickType_t xTicksToWait = pdMS_TO_TICKS( 100UL );	
	BaseType_t xStatus;

	/* 无限循环 */
	for( ;; )
	{
		/* 读队列
		 * xQueue: 读哪个队列
		 * &xReceivedStructure: 读到的数据复制到这个地址
		 * xTicksToWait: 没有数据就阻塞一会
		 */
		xStatus = xQueueReceive( xQueue, &buffer, xTicksToWait); /* 得到buffer地址，只是4字节 */

		if( xStatus == pdPASS )
		{
			/* 读到了数据 */
			printf("Get: %s", buffer);
		}
		else
		{
			/* 没读到数据 */
			printf( "Could not receive from the queue.\r\n" );
		}
	}
}
```



运行结果如下图所示：

![image-20210803181502048](http://photos.100ask.net/rtos-docs/freeRTOS/simulator/chapter-5/09_queue_transfer_big_data.png)



## 5.6 示例11: 邮箱(Mailbox)

本节代码为：`FreeRTOS_11_queue_mailbox`。

FreeRTOS的邮箱概念跟别的RTOS不一样，这里的邮箱称为"橱窗"也许更恰当：

* 它是一个队列，队列长度只有1
* 写邮箱：新数据覆盖旧数据，在任务中使用`xQueueOverwrite()`，在中断中使用`xQueueOverwriteFromISR()`。
  既然是覆盖，那么无论邮箱中是否有数据，这些函数总能成功写入数据。
* 读邮箱：读数据时，数据不会被移除；在任务中使用`xQueuePeek()`，在中断中使用`xQueuePeekFromISR()`。
  这意味着，第一次调用时会因为无数据而阻塞，一旦曾经写入数据，以后读邮箱时总能成功。



main函数中创建了队列(队列长度为1)、创建了发送任务、接收任务：

* 发送任务的优先级为2，它先执行
* 接收任务的优先级为1

代码如下：

```c
/* 队列句柄, 创建队列时会设置这个变量 */
QueueHandle_t xQueue;

int main( void )
{
	prvSetupHardware();
	
    /* 创建队列: 长度为1，数据大小为4字节(存放一个char指针) */
    xQueue = xQueueCreate( 1, sizeof(uint32_t) );

	if( xQueue != NULL )
	{
		/* 创建1个任务用于写队列
		 * 任务函数会连续执行，构造buffer数据，把buffer地址写入队列
		 * 优先级为2
		 */
		xTaskCreate( vSenderTask, "Sender", 1000, NULL, 2, NULL );

		/* 创建1个任务用于读队列
		 * 优先级为1
		 */
		xTaskCreate( vReceiverTask, "Receiver", 1000, NULL, 1, NULL );

		/* 启动调度器 */
		vTaskStartScheduler();
	}
	else
	{
		/* 无法创建队列 */
	}

	/* 如果程序运行到了这里就表示出错了, 一般是内存不足 */
	return 0;
}
```



发送任务、接收任务的代码和执行流程如下：

* A：发送任务先执行，马上阻塞
* BC：接收任务执行，这是邮箱无数据，打印"Could not ..."。在发送任务阻塞过程中，接收任务多次执行、多次打印。
* D：发送任务从阻塞状态退出，立刻执行、写队列
* E：发送任务再次阻塞
* FG、HI、……：接收任务不断"偷看"邮箱，得到同一个数据，打印出多个"Get: 0"
* J：发送任务从阻塞状态退出，立刻执行、覆盖队列，写入1
* K：发送任务再次阻塞
* LM、……：接收任务不断"偷看"邮箱，得到同一个数据，打印出多个"Get: 1"

![image-20210803191526300](http://photos.100ask.net/rtos-docs/freeRTOS/simulator/chapter-5/11_queue_mailbox_code.png)



运行结果如下图所示：

![image-20210803190540351](http://photos.100ask.net/rtos-docs/freeRTOS/simulator/chapter-5/10_queue_mailbox_result.png)


## 技术答疑交流

在学习中遇到任何问题，请前往我们的技术交流社区留言： [https://forums.100ask.net](https://forums.100ask.net)


---
<center>本章完</center>


