---
title: hadoop计算框架shuffle-计算每个月最高三个温度出现的时间
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>MapReduce主要由两部分组成，map和reduce，但是这两部分如何连接？比如对于单词计数，原始数据为`java hadoop java`，map的作用是对单条数据进行处理，划分格式便于处理计算，处理后为`java 1 hadoop 1 java 1`,而reduce是对map的类型进行统一计算，输出为`java 2 hadoop 1`。如果只是这样简单地逻辑，shuffle就不用了，shuffle的作用是将map的数据传递到reduce。但是，我们需要分别统计不同字母开头出现最多的单词，这时reduce不仅仅是一个，26个字母就需要26个reduce。但是map如何对应着26个reduce？这是就需要shuffle的处理，这里的动作叫做partition（即分区的意思）。

* 计算框架Shuffler
	* 在mapper和reducer中间的一个步骤
	* 可以把mapper的输出按照某种key值重新切分和组合成n份，把key值符合某种范围的输出送到特定的reducer那里去处理
	* 可以简化reducer过程

![流程：input->Splitting->Mapping->Shuffling（partition，sort，split to disk）->Reducing-> result][1]
1. 分区partition
	* 每个MapTask的输出都会被分割为多个分区，Reducer会根据JobTask维护的映射关系获取自己应该处理的那一份。

	* 有多少个Reducer，Mapper的输出就应该有多少个分区。

	* 这个分区动作叫做partition，具体逻辑是由partitioner类实现（用户可以自定义自己的partitioner），partition的职责就是保证MapTask输出的数据中具有同类Key的数据进入同一个Reducer进行处理。

2. 三次排序
	* Mapper输出阶段，缓冲区溢写时，溢写结果是分区内排序的。
	* Shuffle阶段，合并溢写文件时需要分区内排序（归并排序）。
	* Copy阶段（Reducer输入阶段），从各个Mapper收集过来的数据先入Reducer的缓冲区，溢写(merge)时整体排序（归并排序）。

* 计算每个月最高三个温度出现的时间
>问题描述：对于一个气温数据`1949-10-01 14:21:02	34c`（第一个空格为空格，第二个空格为tab键），统计每年每个月的最高的三个温度出现的时间（比如：1959 6 45.0）

* 流程
input->Splitting->Mapping->Shuffling（partition，sort，split to disk）->Reducing-> result

* 一步一步写程序（需要7个类）
>开发时，是需要什么（类），就去编写什么，而不是提前去编写好，这样没有头绪
![7个类][2]
* 第一步：程序运行主类MainJob.class和两个内部类：WeatherMapper\WeatherReduce

``` stylus
package com.sound.mr.weather;

import java.io.IOException;
import java.text.ParseException;
import java.text.SimpleDateFormat;
import java.util.Calendar;
import java.util.Date;

import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.DoubleWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.KeyValueTextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;



public class MainJob {
	public static void main(String[] args) {
		Configuration config = new Configuration();
		// 本地调试测试
		 config.set("fs.defaultFS", "hdfs://node1:8020");
		 config.set("yarn.resourcemanager.hostname", "node1");

		// 连接服务器测试
//		config.set("mapred.jar", "C:\\Users\\53033\\Desktop\\wc.jar");
		try {
			Job job = Job.getInstance(config); // 加载配置
			job.setJarByClass(MainJob.class); // 运行类 // 第1个类
			job.setJobName("weather"); // 作业名

			// map输出格式
			job.setMapOutputKeyClass(MyKey.class); // 第2个类
			job.setMapOutputValueClass(DoubleWritable.class);

			// map\reduce主类
			job.setMapperClass(WeatherMapper.class); // 第3个类
			job.setReducerClass(WeatherReduce.class); // 第4个类
			
			// 分别加载reduce分区类（多少个reduce）、自定义排序类、自定义reduce分组类（reduce计算业务组，和Partitioner不同）
			job.setPartitionerClass(MyPartitioner.class); // 第5个类
			job.setSortComparatorClass(MySort.class); // 第6个类
			job.setGroupingComparatorClass(MyGroup.class); // 第7个类

			// reduce任务个数
			job.setNumReduceTasks(3);
			
			// 设置输入格式，否则报错
			// 即对于：1949-10-01 14:21:02	34c
			// 以第一个分隔符（34c前面的tab键）作为分解
			// java.lang.ClassCastException: org.apache.hadoop.io.LongWritable cannot be cast to org.apache.hadoop.io.Text
			job.setInputFormatClass(KeyValueTextInputFormat.class);
			
			FileSystem fs = FileSystem.get(config);
//			Path inPath = new Path("/usr/input/weather");
//			if (!fs.exists(inPath)) {
//				fs.mkdirs(inPath);
//			}
			FileInputFormat.addInputPath(job, new Path("/usr/input/weather"));

			Path outPath = new Path("/usr/output/weather");
			if (fs.exists(outPath)) {
				fs.delete(outPath, true);
			}
			
			FileOutputFormat.setOutputPath(job, outPath);

			boolean finished = job.waitForCompletion(true);

			if (finished) {
				System.out.println("finished success!");
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}

	/**
	 * 
	 * @author 53033
	 * mapper内部类
	 * 解析文本数据，便于处理计算
	 */
	static class WeatherMapper extends Mapper<Text, Text, MyKey, DoubleWritable> {
		SimpleDateFormat sdf = new SimpleDateFormat("yy-MM-dd HH:mm:ss");

		@Override
		protected void map(Text key, Text value, Context context)
				throws IOException, InterruptedException {
			try {
				/**
				 * Date和Calendar区别:
				 * 时间和日历的区别，Date是整个时间，便于计算时分秒，但是Calendar不能
				 * Calendar用于计算年月日，此时date需要转换
				 */
				Date date = sdf.parse(key.toString());
				Calendar calendar = Calendar.getInstance();
				calendar.setTime(date);
				int year = calendar.get(Calendar.YEAR);
				int month = calendar.get(Calendar.MONDAY);
				double weather = Double.parseDouble(value.toString().substring(0, value.toString().lastIndexOf('c')));
				MyKey k = new MyKey();
				k.setYear(year);
				k.setMonth(month);
				k.setWeather(weather);
				context.write(k, new DoubleWritable(weather));
			} catch (ParseException e) {
				e.printStackTrace();
			}
		}
	}
	/**
	 * 
	 * @author 53033
	 * reduce内部类
	 * 输出最终结
	 */
	static class WeatherReduce extends Reducer<MyKey, DoubleWritable, Text, NullWritable> {

		@Override
		protected void reduce(MyKey arg0, Iterable<DoubleWritable> arg1,
				Context arg2)
				throws IOException, InterruptedException {
			int i = 0;
			for (DoubleWritable dw : arg1) {
				i++;
				String msg = arg0.getYear() + "\t" + arg0.getMonth() + "\t" + dw.get();
				arg2.write(new Text(msg), NullWritable.get());
				if (i == 3) {
					break;
				}
			}
		}

	}
}

```

* 第二步：MyKey.class

``` stylus
package com.sound.mr.weather;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

import org.apache.hadoop.io.WritableComparable;

import com.sun.org.apache.regexp.internal.recompile;

public class MyKey implements WritableComparable<MyKey>{

	private int year;
	private int month;
	private double weather;
	
	
	public int getYear() {
		return year;
	}

	public void setYear(int year) {
		this.year = year;
	}

	public int getMonth() {
		return month;
	}

	public void setMonth(int month) {
		this.month = month;
	}

	public double getWeather() {
		return weather;
	}

	public void setWeather(double weather) {
		this.weather = weather;
	}

	@Override
	public int compareTo(MyKey o) {
		int r0 = Integer.compare(this.getYear(), o.getYear());
		if(r0==0) {
			int r1 = Integer.compare(this.getMonth(), o.getMonth());
			if(r1==0) {
				return Double.compare(this.getWeather(), o.getWeather());
			}else {
				return r1;
			}
		}else {
			return r0;
		}
	}

	// 反序列化：从文件中读取对象
	@Override
	public void readFields(DataInput arg0) throws IOException {
		this.year=arg0.readInt();
		this.month=arg0.readInt();
		this.weather=arg0.readDouble();
	}

	// 序列化：讲对象写入文件
	@Override
	public void write(DataOutput arg0) throws IOException {
		arg0.writeInt(year);
		arg0.writeInt(month);
		arg0.writeDouble(weather);
	}
	
}

```


* 第三步：MyPartitioner分区类、MySort排序类、MyGroup分组类

``` stylus
package com.sound.mr.weather;

import org.apache.hadoop.io.DoubleWritable;
import org.apache.hadoop.mapreduce.Partitioner;

public class MyPartitioner extends Partitioner{
	
	// 根据年进行分组，即一年对应一个reduce
	@Override
	public int getPartition(MyKey key, DoubleWritable value, int numReduceTasks) {
		// 1949为最小年份，除以reduce数目，取余
		return (key.getYear()-1949)%numReduceTasks;
	}
	
}


package com.sound.mr.weather;

import org.apache.hadoop.io.WritableComparable;
import org.apache.hadoop.io.WritableComparator;

public class MySort extends WritableComparator {
	// 指明比较的对象，true表明创建对象mykey实例
	public MySort() {
		super(MyKey.class, true);
	}

	// 大小排序
	@Override
	public int compare(WritableComparable a, WritableComparable b) {
		MyKey akey = (MyKey) a;
		MyKey bkey = (MyKey) b;
		return akey.compareTo(bkey);
	}

}


package com.sound.mr.weather;

import org.apache.hadoop.io.WritableComparable;
import org.apache.hadoop.io.WritableComparator;

public class MyGroup extends WritableComparator{
	public MyGroup() {
		super(MyKey.class,true);
	}
	
	// 进行分组，即年相同、月相同，分为一组；
	// 只比较是否相等，不比较大小；大小排序有sort完成
	@Override
	public int compare(WritableComparable a, WritableComparable b) {
		MyKey akey = (MyKey) a;
		MyKey bkey = (MyKey) b;
		int r0 = Integer.compare(akey.getYear(), bkey.getYear());
		if(r0==0){
			return Integer.compare(akey.getMonth(),bkey.getMonth());
		}else {
			return r0;
		}
	}
	
}

```

* 运行结果
![enter description here][3]



* 错误注意：`org.apache.hadoop.io.LongWritable cannot be cast to org.apache.hadoop.io.Text`
	* 原因：输入格式未指定，导致无法自动识别隔离符，需要加`job.setInputFormatClass(KeyValueTextInputFormat.class);` 
	* 另外KeyValueTextInputFormat指定为`import org.apache.hadoop.mapreduce.lib.input.KeyValueTextInputFormat;`


参考链接：
[http://roserouge.iteye.com/blog/746391][4]
[http://matt33.com/2016/03/02/hadoop-shuffle/][5]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525264759203.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525314005498.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525314455332.jpg
  [4]: http://roserouge.iteye.com/blog/746391
  [5]: http://matt33.com/2016/03/02/hadoop-shuffle/
