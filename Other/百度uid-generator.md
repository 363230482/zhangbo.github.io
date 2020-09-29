# 百度uid-generator分布式ID生成

## Snowflake算法
Snowflake方案：1位(固定为0) + 28位时间戳(最大支持8.7年) + 22位workerId(420w次启动) + 13位(序列号8192)
每次启动，相当于在MySQL中自动注册一条记录，以该记录的自增ID位当前服务的workerId

双RingBuffer：一个buffer存放当前的id，一个存放buffer对应index的是否可take/put的标记位
// RingBuffer的size
private final int bufferSize; 
private final long indexMask;
// 存放已分配的id
private final long[] slots;
// 存放slots对应index的take/put标记位
private final PaddedAtomicLong[] flags;

// 当前的put指针
private final AtomicLong tail = new PaddedAtomicLong(START_POINT);
// 当前take指针，及当前获取id到的index
private final AtomicLong cursor = new PaddedAtomicLong(START_POINT);
// 当take指针与put指针之间的距离(剩余缓存id个数)小于该值时，启动后台线程生成id put到slots中
private final int paddingThreshold; 
// put拒绝策略，默认大于warn日志
private RejectedPutBufferHandler rejectedPutHandler = this::discardPutBuffer;
// take拒绝策略，默认抛异常
private RejectedTakeBufferHandler rejectedTakeHandler = this::exceptionRejectedTakeBuffer; 
// 后台生成id的线程
private BufferPaddingExecutor bufferPaddingExecutor;


