# 背景

kafka没有重试机制不支持消息重试，也没有死信队列，因此使用kafka做消息队列时，如果遇到了消息在业务处理时出现异常，就会很难进行下一步处理。应对这种场景，需要自己实现消息重试的功能。



# 方案

申请一个新的kafka topic作为重试队列，步骤如下：

1. 创建一个topic作为重试topic用于接受等待重试的消息
2. 普通topic消费者给待重试的消息设置下一次的消费事件后发送到重试topic
3. 从重试topic获取待重试消息储存到redis的zset中，并以下一次消费时间排序
4. 定时任务从redis获取到达消费事件的消息，并把消息发送到对应的topic
5. 同一个消息重试次数过多则不再重试
   

# 代码实现

## 重试消息的javaBean

```java
public class KafkaRetryRecord {

    public static final String KEY_RETRY_TIMES = "retryTimes";

    private String key;
    private String value;

    private Integer retryTimes;
    private String topic;
    private Long nextTime;

    public KafkaRetryRecord(){
    }

    public String getKey() {
        return key;
    }
    public void setKey(String key) {
        this.key = key;
    }
    public String getValue() {
        return value;
    }
    public void setValue(String value) {
        this.value = value;
    }
    public Integer getRetryTimes() {
        return retryTimes;
    }
    public void setRetryTimes(Integer retryTimes) {
        this.retryTimes = retryTimes;
    }
    public String getTopic() {
        return topic;
    }
    public void setTopic(String topic) {
        this.topic = topic;
    }
    public Long getNextTime() {
        return nextTime;
    }
    public void setNextTime(Long nextTime) {
        this.nextTime = nextTime;
    }

    public ProducerRecord parse(){
        Integer partition = null;
        Long timestamp = System.currentTimeMillis();
        List<Header> headers = new ArrayList<>();
        ByteBuffer retryTimesBuffer = ByteBuffer.allocate(4);
        retryTimesBuffer.putInt(retryTimes);
        retryTimesBuffer.flip();
        headers.add(new RecordHeader(KafkaRetryRecord.KEY_RETRY_TIMES, retryTimesBuffer));

        ProducerRecord sendRecord = new ProducerRecord(
                topic, partition, timestamp, key, value, headers);
        return sendRecord;
    }
}
```



## 消费端的消息发送到重试队列

```java
public class KafkaRetryService {

	private static final Logger log = LoggerFactory.getLogger(KafkaRetryService.class);

	/**
	 * 消息消费失败后下一次消费的延迟时间(秒)
	 * 第一次重试延迟10秒;第二次延迟30秒,第三次延迟1分钟...
	 */
	private static final int[] RETRY_INTERVAL_SECONDS = {10, 30, 1*60, 2*60, 5*60, 10*60, 30*60, 1*60*60, 2*60*60};

	/**
	 * 重试topic
	 */
	@Value("${spring.kafka.topics.retry}")
	private String retryTopic;
	@Autowired
	private KafkaTemplate<String, String> template;

	public void consumerLater(ConsumerRecord<String, String> record){
		// 获取消息的已重试次数
		int retryTimes = getRetryTimes(record);
		Date nextConsumerTime = getNextConsumerTime(retryTimes);
		if(nextConsumerTime == null) {
			return;
		}

		KafkaRetryRecord retryRecord = new KafkaRetryRecord();
		retryRecord.setNextTime(nextConsumerTime.getTime());
		retryRecord.setTopic(record.topic());
		retryRecord.setRetryTimes(retryTimes);
		retryRecord.setKey(record.key());
		retryRecord.setValue(record.value());

		String value = JSON.toJSONString(retryRecord);
		template.send(retryTopic, null, value);
	}

	/**
	 * 获取消息的已重试次数
	 */
	private int getRetryTimes(ConsumerRecord record){
		int retryTimes = -1;
		for(Header header : record.headers()){
			if(KafkaRetryRecord.KEY_RETRY_TIMES.equals(header.key())){
				ByteBuffer buffer = ByteBuffer.wrap(header.value());
				retryTimes = buffer.getInt();
			}
		}
		retryTimes++;
		return retryTimes;
	}

	/**
	 * 获取待重试消息的下一次消费时间
	 */
	private Date getNextConsumerTime(int retryTimes){
		// 重试次数超过上限,不再重试
		if(RETRY_INTERVAL_SECONDS.length < retryTimes) {
			return null;
		}

		Calendar calendar = Calendar.getInstance();
		calendar.add(Calendar.SECOND, RETRY_INTERVAL_SECONDS[retryTimes]);
		return calendar.getTime();
	}

}
```



## 处理待消费的消息

```java
public class RetryListener {
	private static final Logger log = LoggerFactory.getLogger(RetryListener.class);

	private static final String RETRY_KEY_ZSET = "_retry_key";
	private static final String RETRY_VALUE_MAP = "_retry_value";
	@Autowired
	private RedisTemplate<String,Object> redisTemplate;
	@Autowired
	private KafkaTemplate<String, String> kafkaTemplate;

	@KafkaListener(topics = "${spring.kafka.topics.retry}")
	public void consume(List<ConsumerRecord<String, String>> list) {
		for(ConsumerRecord<String, String> record : list){
			KafkaRetryRecord retryRecord = JSON.parseObject(record.value(), KafkaRetryRecord.class);

			/**
			 * TODO 防止待重试消息太多撑爆redis,可以将待重试消息按下一次重试时间分开存储放到不同介质
			 * 例如下一次重试时间在半小时以后的消息储存到mysql,并定时从mysql读取即将重试的消息储储存到redis
			 */

			// 通过redis的zset进行时间排序
			String key = UUID.randomUUID().toString();
			redisTemplate.opsForHash().put(RETRY_VALUE_MAP, key, record.value());
			redisTemplate.opsForZSet().add(RETRY_KEY_ZSET, key, retryRecord.getNextTime());
		}
	}

	/**
	 * 定时任务从redis读取到达重试时间的消息,发送到对应的topic
	 */
	@Scheduled(cron="0/2 * * * * *")
	public void retryFormRedis() {
		long currentTime = System.currentTimeMillis();
		Set<ZSetOperations.TypedTuple<Object>> typedTuples =
				redisTemplate.opsForZSet().reverseRangeByScoreWithScores(RETRY_KEY_ZSET, 0, currentTime);
		redisTemplate.opsForZSet().removeRangeByScore(RETRY_KEY_ZSET, 0, currentTime);
		for(ZSetOperations.TypedTuple<Object> tuple : typedTuples){
			String key = tuple.getValue().toString();
			String value = redisTemplate.opsForHash().get(RETRY_VALUE_MAP, key).toString();
			redisTemplate.opsForHash().delete(RETRY_VALUE_MAP, key);
			KafkaRetryRecord retryRecord = JSON.parseObject(value, KafkaRetryRecord.class);
			ProducerRecord record = retryRecord.parse();
			kafkaTemplate.send(record);
		}
		// TODO 发生异常将发送失败的消息重新扔回redis
	}
}
```



## 消息重试

```java
public class ConsumeListener {
	private static final Logger log = LoggerFactory.getLogger(ConsumeListener.class);

	@Autowired
	private KafkaRetryService kafkaRetryService;

	@KafkaListener(topics = "${spring.kafka.topics.test}")
	public void consume(List<ConsumerRecord<String, String>> list) {
		for(ConsumerRecord record : list){
			try {
				// 业务处理
			} catch (Exception e){
				log.error(e.getMessage());
				// 消息重试
				kafkaRetryService.consumerLater(record);
			}
		}
	}

}
```

