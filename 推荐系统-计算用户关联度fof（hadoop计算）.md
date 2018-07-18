---
title: 推荐系统-计算用户关联度fof（hadoop计算） 
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
>场景：无论是qq，还是微博、头条等带有社交属性的平台，为了黏住用户，往往会给用户推荐好友，这种好友一般都是更具自己的兴趣或者自己好友的好友得来，比如qq中“可能认识的人”。

fof关系：对于任何一个用户A，用户A的好友集合为B，B中的任何两个用户之间的关系如果不是好友关系，则就为fof关系

推荐系数需要排序，排序的依据就是整个用户组的fof关系的多少，同样的fof关系越多，表明两个用户的关联度越大，推荐系数就越高

数据：
![enter description here][1]

定义一个fof关系类

``` stylus
package com.sound.mr.friend;

import org.apache.hadoop.io.Text;

public class Fof extends Text{
	public Fof() {
		super();
	}
	
	public Fof(String a,String b){
		super(getFof(a,b));
	}

	// 对fof关系进行排序，即：小明-小红，小红-小明统一为一个
	private static String getFof(String a, String b) {
		int r0 = a.compareTo(b);
		if(r0<0){
			return a+"\t"+b;
		}else {
			return b+"\t"+a;
		}
	}
	
	
}

```
* 主函数

``` stylus
package com.sound.mr.friend;

import java.io.IOException;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.mapreduce.Reducer;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.input.KeyValueTextInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import org.apache.hadoop.util.StringUtils;


public class MainJob {
	public static void main(String[] args) {
		Configuration config = new Configuration();
		// 本地调试测试
		config.set("fs.defaultFS", "hdfs://node1:8020");
		config.set("yarn.resourcemanager.hostname", "node1");

		// 连接服务器测试
		// config.set("mapred.jar", "C:\\Users\\53033\\Desktop\\wc.jar");
		
		
		// 之所以有两个是因为，第一个用于找到所有的fof关系
		// 第二个是对fof关系进行排序
		if(runFind(config)){
			runSort(config);
		}
	}

	// 对所有的fof关系进行大小排序
	private static void runSort(Configuration config) {
		try {
			Job job = Job.getInstance(config);
			job.setJarByClass(MainJob.class);
			job.setJobName("sort fof of friends");

			job.setMapOutputKeyClass(Fofer.class);
			job.setMapOutputValueClass(NullWritable.class);

			job.setMapperClass(SortMapper.class);
			job.setReducerClass(SortReducer.class);
			
			job.setSortComparatorClass(MySort.class);
			
			job.setInputFormatClass(KeyValueTextInputFormat.class);
			
			FileSystem fs = FileSystem.get(config);
//			Path inPath = new Path("/usr/input/");
//			if (!fs.exists(inPath)) {
//				fs.mkdirs(inPath);
//			}
			FileInputFormat.addInputPath(job, new Path("/usr/output/friend"));

			Path outPath = new Path("/usr/output/friendSort");
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

	//	用于查找所有Fof关系	
	public static boolean runFind(Configuration config) {
		boolean finished = false;
		try {
			Job job = Job.getInstance(config);
			job.setJarByClass(MainJob.class);
			job.setJobName("find fof of friends");

			job.setMapOutputKeyClass(Fof.class);
			job.setMapOutputValueClass(IntWritable.class);

			job.setMapperClass(FofMapper.class);
			job.setReducerClass(FofReducer.class);
			
			job.setInputFormatClass(KeyValueTextInputFormat.class);
			
			FileSystem fs = FileSystem.get(config);
			Path inPath = new Path("/usr/input/");
			if (!fs.exists(inPath)) {
				fs.mkdirs(inPath);
			}
			FileInputFormat.addInputPath(job, new Path("/usr/input/"));

			Path outPath = new Path("/usr/output/friend");
			if (fs.exists(outPath)) {
				fs.delete(outPath, true);
			}
			FileOutputFormat.setOutputPath(job, outPath);

			finished = job.waitForCompletion(true);

			if (finished) {
				System.out.println("finished success!");
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
		return finished;
	}

	static class FofMapper extends Mapper<Text, Text, Fof, IntWritable> {

		@Override
		protected void map(Text key, Text value, Context context)
				throws IOException, InterruptedException {
			String user = key.toString();
			String[] friends = StringUtils.split(value.toString(), '\t');
			for (int i = 0; i < friends.length; i++) {
				Fof ofof = new Fof(user, friends[i]);
				
				// 朋友关系
				context.write(ofof, new IntWritable(0));
				for(int j=i+1;j<friends.length;j++) {
					// fof关系，但是也可能是朋友关系
					Fof fof = new Fof(friends[i],friends[j]);
					context.write(fof, new IntWritable(1));
				}
			}
		}

	}
	
	static class FofReducer extends Reducer<Fof, IntWritable, Fof, IntWritable>{

		@Override
		protected void reduce(Fof arg0, Iterable<IntWritable> arg1,
				Context arg2) throws IOException, InterruptedException {
			int sum =0;
			boolean isFriend = false;
			for(IntWritable iw: arg1) {
				if(iw.get()==0) {
					isFriend = true;
					break; // 等于0，表示已经是朋友关系，就不用累计计算了
				}else {
					sum+=iw.get();
//					sum+=1;
				}
			}
			if(!isFriend){
				arg2.write(arg0, new IntWritable(sum));
			}
		}
		
	}
	
	
	static class SortMapper extends Mapper<Text, Text, Fofer, NullWritable> {
		@Override
		protected void map(Text key, Text value, Context context)
				throws IOException, InterruptedException {
			// 这里注意一个情况，第一步run输出的文件，第一个间隔符在fof中间，
			// 所以key值为一个单个人名：“老王”；value为：“老宋  3”
			String[] fofs = StringUtils.split(value.toString(), '\t');
			int num = Integer.valueOf(fofs[1]);
			Fofer fofer = new Fofer();
			fofer.setFof(key.toString()+'\t'+fofs[0]);
			fofer.setNum(num);
			context.write(fofer, NullWritable.get());
		}

	}
	
	static class SortReducer extends Reducer<Fofer, NullWritable, Text, NullWritable>{

		@Override
		protected void reduce(Fofer arg0, Iterable<NullWritable> arg1,
				Context arg2) throws IOException, InterruptedException {
			// 直接写入
			arg2.write(new Text(arg0.getFof()+arg0.getNum()), NullWritable.get());
		}
		
	}
}

```
* 其中第二步mapreduce需要进行排序：相应的key和sort如下，可参考上一篇文章

``` stylus
package com.sound.mr.friend;

import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

import org.apache.hadoop.io.WritableComparable;

/**
 * @author 53033
 * 用于排序
 */
public class Fofer implements WritableComparable<Fofer>{
	private String fof;
	private int num;
	
	
	public String getFof() {
		return fof;
	}

	public void setFof(String fof) {
		this.fof = fof;
	}

	public int getNum() {
		return num;
	}

	public void setNum(int num) {
		this.num = num;
	}

	@Override
	public int compareTo(Fofer o) {
		int r0 = -Integer.compare(this.getNum(), o.getNum());
		
		
		// 这里必须要比较字符串，否则reduce时，对于num相同的，只取其中一个
		if(r0==0) {
			return -this.getFof().compareTo(o.getFof());
		}else {
			return r0;
		}
	}

	@Override
	public void write(DataOutput out) throws IOException {
		out.writeUTF(this.getFof());
		out.writeInt(this.getNum());
	}

	@Override
	public void readFields(DataInput in) throws IOException {
		this.setFof(in.readUTF());
		this.setNum(in.readInt());
	}

}



package com.sound.mr.friend;

import org.apache.hadoop.io.WritableComparable;
import org.apache.hadoop.io.WritableComparator;
/**
 * 
 * @author 53033
 * 排序规则
 */
public class MySort extends WritableComparator {
	// 指明比较的对象，true表明创建对象mykey实例
	public MySort() {
		super(Fofer.class, true);
	}

	// 大小排序
	@Override
	public int compare(WritableComparable a, WritableComparable b) {
		Fofer akey = (Fofer) a;
		Fofer bkey = (Fofer) b;
		return akey.compareTo(bkey);
	}

}

```

* 结果
![enter description here][2]


错误记录：
	* 工程clean
	* eclipse没有DFS文件夹
	![enter description here][3]
	* 访问http://node1:50070，Live Nodes少一个
	![enter description here][4]
		* 原因：storage id出错了，不一致
		* 解决办法：找到datanode的data文件价，移去current文件夹，重新启动datanode:`hadoop-daemon.sh start datanode`
		* 注意：请找准出错节点；另外data文件夹为datanode所在目录，勿找错


参考链接：
[https://my.oschina.net/u/3264690/blog/1377199][5]


  [1]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525362715504.jpg
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525403786519.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525404943628.jpg
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1525405156727.jpg
  [5]: https://my.oschina.net/u/3264690/blog/1377199
