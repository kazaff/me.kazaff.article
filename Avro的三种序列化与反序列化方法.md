知道Avro有一段时间了，但是并没有过多的动手测试其用法，今天花时间好好找了几个例子来练习Avro，特此记录，供以后查阅。

Avro提供了自身的RPC实现，可以更完美的利用Avro的设计思想来完成高性能的跨进程通信，不过我们这里只注重使用Avro的序列化/反序列化的细节。

Avro使用Json来声明Schema，这个预先定义的模式作为通信两端数据协议，在网络传输前后对目标数据进行二进制处理。太多的理论可以从Avro官网上了解到，我们下面就进入代码模式。

测试代码是基于Maven的，你可能会使用下面的pom.xml：

	<dependencies>
        <dependency>
            <groupId>org.apache.avro</groupId>
            <artifactId>avro</artifactId>
            <version>1.7.6-cdh5.2.5</version>
        </dependency>
    </dependencies>

	<repositories>
        <repository>
            <id>cloudera-repo-releases</id>
            <url>https://repository.cloudera.com/artifactory/repo/</url>
        </repository>
    </repositories>

#### 方法1
---
该方法描述的场景是：用指定的json schema对数据进行编码，并用相同/近似的schema对数据进行解码。
	
	import java.io.ByteArrayOutputStream;
	import java.io.IOException;
	
	import org.apache.avro.Schema;
	import org.apache.avro.generic.GenericData;
	import org.apache.avro.generic.GenericDatumReader;
	import org.apache.avro.generic.GenericDatumWriter;
	import org.apache.avro.generic.GenericRecord;
	import org.apache.avro.io.DatumReader;
	import org.apache.avro.io.Decoder;
	import org.apache.avro.io.DecoderFactory;
	import org.apache.avro.io.Encoder;
	import org.apache.avro.io.EncoderFactory;
	
	public class AvroTest {
	    public static void main(String[] args) throws IOException {
	        // Schema
	        String schemaDescription = " {    \n"
	                + " \"name\": \"FacebookUser\", \n"
	                + " \"type\": \"record\",\n" + " \"fields\": [\n"
	                + "   {\"name\": \"name\", \"type\": \"string\"},\n"
	                + "   {\"name\": \"num_likes\", \"type\": \"int\"},\n"
	                + "   {\"name\": \"num_photos\", \"type\": \"int\"},\n"
	                + "   {\"name\": \"num_groups\", \"type\": \"int\"} ]\n" + "}";
	
	        Schema s = Schema.parse(schemaDescription);	//parse方法在当前的Avro版本下已不推荐使用
	        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
	        Encoder e = EncoderFactory.get().binaryEncoder(outputStream, null);
	        GenericDatumWriter w = new GenericDatumWriter(s);
	
	        // Populate data
	        GenericRecord r = new GenericData.Record(s);
	        r.put("name", new org.apache.avro.util.Utf8("kazaff"));
	        r.put("num_likes", 1);
	        r.put("num_groups", 423);
	        r.put("num_photos", 0);
	
	        // Encode
	        w.write(r, e);
	        e.flush();
	       
	        byte[] encodedByteArray = outputStream.toByteArray();
	        String encodedString = outputStream.toString();
	       
	        System.out.println("encodedString: "+encodedString);
	       
	        // Decode using same schema
	        DatumReader<GenericRecord> reader = new GenericDatumReader<GenericRecord>(s);
	        Decoder decoder = DecoderFactory.get().binaryDecoder(encodedByteArray, null);
	        GenericRecord result = reader.read(null, decoder);
	        System.out.println(result.get("name").toString());
	        System.out.println(result.get("num_likes").toString());
	        System.out.println(result.get("num_groups").toString());
	        System.out.println(result.get("num_photos").toString());
	       
	    }
	}

可以看出，编码解码都需要指定schema，这相当于要求通信两端必须提前通过某种交互来完成schema的同步工作（貌似Avro自带的RPC就实现了这个细节），通过代码中的打印语句，我们可以留意到，这种方式下，编码后的数据是不包含schema的（**我不确定对，但相比第二种方式确实有这种差异性**）。


#### 方法2
---
该方法采用的方式是通过指定的json schema把数据进行编码，并同时把schema作为metadata放在编码后的数据头部。解码则使用内嵌在数据中的schema来完成。

	import java.io.ByteArrayInputStream;
	import java.io.ByteArrayOutputStream;
	import java.io.IOException;
	
	import org.apache.avro.Schema;
	import org.apache.avro.file.DataFileStream;
	import org.apache.avro.file.DataFileWriter;
	import org.apache.avro.generic.GenericData;
	import org.apache.avro.generic.GenericDatumReader;
	import org.apache.avro.generic.GenericDatumWriter;
	import org.apache.avro.generic.GenericRecord;
	import org.apache.avro.io.DatumReader;
	import org.apache.avro.io.DatumWriter;
	
	public class AvroDataFile {
	
	    /**
	     * @param args
	     * @throws IOException
	     */
	    public static void main(String[] args) throws IOException {
	        // Schema
	        String schemaDescription = " {    \n"
	                + " \"name\": \"FacebookUser\", \n"
	                + " \"type\": \"record\",\n" + " \"fields\": [\n"
	                + "   {\"name\": \"name\", \"type\": \"string\"},\n"
	                + "   {\"name\": \"num_likes\", \"type\": \"int\"},\n"
	                + "   {\"name\": \"num_photos\", \"type\": \"int\"},\n"
	                + "   {\"name\": \"num_groups\", \"type\": \"int\"} ]\n" + "}";
	
	        Schema schema = Schema.parse(schemaDescription);	//parse方法在当前的Avro版本下已不推荐使用
	        ByteArrayOutputStream os = new ByteArrayOutputStream();
	        DatumWriter<GenericRecord> writer = new GenericDatumWriter<GenericRecord>(schema);
	        DataFileWriter<GenericRecord> dataFileWriter = new DataFileWriter<GenericRecord>(writer);
	        dataFileWriter.create(schema, os);
	
	        // Populate data
	        GenericRecord datum = new GenericData.Record(schema);
	        datum.put("name", new org.apache.avro.util.Utf8("kazaff"));
	        datum.put("num_likes", 1);
	        datum.put("num_groups", 423);
	        datum.put("num_photos", 0);
	
	        dataFileWriter.append(datum);
	        dataFileWriter.close();
	
	        System.out.println("encoded string: " + os.toString());	//可以看到，数据是头部携带了schema metadata
	
	        // Decode
	        DatumReader<GenericRecord> reader = new GenericDatumReader<GenericRecord>();
	        ByteArrayInputStream is = new ByteArrayInputStream(os.toByteArray());
	        DataFileStream<GenericRecord> dataFileReader = new DataFileStream<GenericRecord>(is, reader);
	
	        GenericRecord record = null;
	        while (dataFileReader.hasNext()) {
	            record = dataFileReader.next(record);
				System.out.println(record.getSchema());	//可以直接获取该数据的json schema定义
	            System.out.println(record.get("name").toString());
	            System.out.println(record.get("num_likes").toString());
	            System.out.println(record.get("num_groups").toString());
	            System.out.println(record.get("num_photos").toString());
	        }
	    }
	}

该方法最符合我的预期，这样若我们使用第三方RPC框架（like dubbo），就不需要为schema的数据同步问题花太多精力，但是能想到的问题就是通信成本，毕竟每次数据通信都要携带json schema metadata感觉总是不太好接受，尤其是针对单一长连接场景，似乎更应该去实现类似Avro提供的RPC那样的机制，可以获得更好的性能。


#### 方法3
---
利用Avro提供的反射机制来完成数据的编码解码，**该方法我没有进行实测**。

	import java.io.ByteArrayOutputStream;
	import java.io.IOException;
	
	import org.apache.avro.Schema;
	import org.apache.avro.io.DatumWriter;
	import org.apache.avro.io.Decoder;
	import org.apache.avro.io.DecoderFactory;
	import org.apache.avro.io.Encoder;
	import org.apache.avro.io.EncoderFactory;
	import org.apache.avro.reflect.ReflectData;
	import org.apache.avro.reflect.ReflectDatumReader;
	import org.apache.avro.reflect.ReflectDatumWriter;
	
	import dto.AnotherEmployee;
	import dto.Employee;
	
	public class AvroReflect {
	    final static ReflectData reflectData = ReflectData.get();
	    final static Schema schema = reflectData.getSchema(Employee.class);
	
	    public static void main(String[] args) throws IOException {
	        ByteArrayOutputStream os = new ByteArrayOutputStream();
	        Encoder e = EncoderFactory.get().binaryEncoder(os, null);
	
	        DatumWriter<Employee> writer = new ReflectDatumWriter<Employee>(schema);
	        Employee employee = new Employee();
	        employee.setName("Kamal");
	        employee.setSsn("000-00-0000");
	        employee.setAge(29);
	
	        writer.write(employee, e);
	        e.flush();
	       
	        System.out.println(os.toString());
	       
	        ReflectData reflectData = ReflectData.get();
	        Schema schm = reflectData.getSchema(AnotherEmployee.class);
	        ReflectDatumReader<AnotherEmployee> reader = new ReflectDatumReader<AnotherEmployee>(schm);
	        Decoder decoder = DecoderFactory.get().binaryDecoder(os.toByteArray(), null);
	        AnotherEmployee decodedEmployee = reader.read(null, decoder);
	       
	        System.out.println("Name: "+decodedEmployee.getName());
	        System.out.println("Age: "+decodedEmployee.getAge());
	        System.out.println("SSN: "+decodedEmployee.getSsn());
	    }
	}


以上代码，均参考自：[Apache Avro "HelloWorld" Examples](http://bigdatazone.blogspot.com/2012/04/apache-avro-helloworld-examples.html)