最近在对消息中间件进行调研，原先项目里使用的是RabbitMQ，理由很简单：**对开发语言支持度是最好的**，没有之一。但是业界的反馈是，其高并发和分布式支持存在不足~

我就打算再了解一下**kafka**，看看它是怎么个用法~~

如RabbitMQ一样，kafka也提供了终端脚本来完成基本功能的测试，可以看一下[这里](http://colobu.com/2014/08/06/kafka-quickstart/#show-last-Point)。但光玩玩脚本或官方提供的例子，是不能满足我的~

作为把消息队列，让其使用在项目中，除了解决和业务代码进行互动以外，还要考虑数据传输的格式问题，也就是说如何解决producer和consumer之间通信的协议问题。

官方推荐的就是Avro，只可惜我找了半天，都没有一个现成的kafka+Avro的demo供我测试，那就只能自己试着写个了~~

	package me.kazaff.mq;
	
	import kafka.consumer.ConsumerConfig;
	import kafka.consumer.ConsumerIterator;
	import kafka.consumer.KafkaStream;
	import kafka.javaapi.consumer.ConsumerConnector;
	import kafka.javaapi.producer.Producer;
	import kafka.producer.KeyedMessage;
	import kafka.producer.ProducerConfig;
	import me.kazaff.mq.avro.Message;
	import org.apache.avro.io.*;
	import org.apache.avro.specific.SpecificDatumReader;
	import org.apache.avro.specific.SpecificDatumWriter;
	
	import java.io.ByteArrayOutputStream;
	import java.io.IOException;
	import java.util.*;
	
	public class Main {
	
	    public static void main(String[] args){
	        try {
	            //消息生产
	            produce();
	
	            //消息消费
	            consume();
	
	        }catch (Exception ex){
	            System.out.println(ex);
	        }
	
	    }
	
	    private static void produce() throws IOException{
	        Properties props = new Properties();
	        props.put("metadata.broker.list", "localhost:9092");
	        props.put("serializer.class", "kafka.serializer.DefaultEncoder");
	        props.put("key.serializer.class", "kafka.serializer.StringEncoder");    //key的类型需要和serializer保持一致，如果key是String，则需要配置为kafka.serializer.StringEncoder，如果不配置，默认为kafka.serializer.DefaultEncoder，即二进制格式
	        props.put("partition.class", "me.kazaff.mq.MyPartitioner");
	        props.put("request.required.acks", "1");
	
	        ProducerConfig config = new ProducerConfig(props);
	        Producer<String, byte[]> producer = new Producer<String, byte[]>(config);
	
	        Random rnd = new Random();
	        for(int index = 0; index <= 10; index++){
	            Message msg = new Message();
	            msg.setUrl("blog.kazaff.me");
	            msg.setIp("192.168.137." + rnd.nextInt(255));
	            msg.setDate(Long.toString(new Date().getTime()));
	
	            DatumWriter<Message> msgDatumWriter = new SpecificDatumWriter<Message>(Message.class);
	            ByteArrayOutputStream os = new ByteArrayOutputStream();
	            try {
	                Encoder e = EncoderFactory.get().binaryEncoder(os, null);
	                msgDatumWriter.write(msg, e);
	                e.flush();
	                byte[] byteData = os.toByteArray();
	
	                KeyedMessage<String, byte[]> data = new KeyedMessage<String, byte[]>("demo", "0", byteData);
	                producer.send(data);
	
	            }catch (IOException ex){
	                System.out.println(ex.getMessage());
	            }finally {
	                os.close();
	            }
	        }
	        producer.close();
	    }
	
	    private static void consume(){
	        Properties props = new Properties();
	        props.put("zookeeper.connect", "localhost:2181");
	        props.put("group.id", "1");
	        props.put("zookeeper.session.timeout.ms", "400");
	        props.put("zookeeper.sync.time.ms", "200");
	        props.put("auto.commit.interval.ms", "1000");
	
	        ConsumerConnector consumer = kafka.consumer.Consumer.createJavaConsumerConnector(new ConsumerConfig(props));
	        Map<String, Integer> topicCountMap = new HashMap<String, Integer>();
	        topicCountMap.put("demo", 1);
	        Map<String, List<KafkaStream<byte[], byte[]>>> consumerMap = consumer.createMessageStreams(topicCountMap);
	        List<KafkaStream<byte[], byte[]>> streams = consumerMap.get("demo");
	        KafkaStream steam = streams.get(0);
	
	        ConsumerIterator<byte[], byte[]> it = steam.iterator();
	        while (it.hasNext()){
	            try {
	                DatumReader<Message> reader = new SpecificDatumReader<Message>(Message.class);
	                Decoder decoder = DecoderFactory.get().binaryDecoder(it.next().message(), null);
	                Message msg = reader.read(null, decoder);
	
	                System.out.println(msg.getDate() + "," + msg.getUrl() + "," + msg.getIp());
	
	            }catch (Exception ex){
	                System.out.println(ex);
	            }
	        }
	
	        if(consumer != null)
	            consumer.shutdown();
	    }
	}

**PS：这只是个测试的例子，存在各种问题，不建议直接使用在项目中。**



### 问题
---
#### 1.

> java.lang.ClassCastException: [B cannot be cast to java.lang.String

解决方法很简单：

	props.put("serializer.class", "kafka.serializer.DefaultEncoder");

这样，kafka就默认使用二进制的序列化方案处理Avro的编码结果了。


#### 2.

> java.lang.ClassCastException: java.lang.String cannot be cast to [B

这个问题是最恶心的，搜了半天都没有找到原因，原因是**问题1**中那么设置后，Kafka所有的数据序列化方式都成了二进制方式，包括我们后面要使用的“key”（用于kafka选择分区的线索）。

所以你还需要再加一条配置：

	props.put("key.serializer.class", "kafka.serializer.StringEncoder");

单独设置一下“key”的序列化方式，这样就可以编译运行了~~


---

初尝Kafka和Avro，就这么点儿要记录的，不要嫌少哟~~