---
title: hadoop集群-单词计数wordcount 
tags: 新建,模板,小书匠
grammar_cjkRuby: true
---
* 开发环境
系统：CentOS release 6.5
jdk：jdk1.7.0_45
hadoop：2.5.2

* hadoop集群搭建
参照：[https://blog.csdn.net/soundslow/article/details/80101146][1]

* eclipse插件配置
由于需要使用跨平台文件传递，多以需要一个hadoop插件hadoop-eclipse-plugin-2.5.2.jar,
将其放入eclipse插件文件中，重启eclipse，在new中将会出现map/reduce选项
![map/reduce][2]

* 配置mapreduce=
![location][3]
![文件显示][4]


* 编写计算文件
计算流程：
![两个函数map和reduce][5]
1、分割类map：WordCountMapper

``` stylus
package com.sound.mr.wc;

import java.io.IOException;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import org.apache.hadoop.util.StringUtils;


/**
 * 
 * @author 53033 <KEYIN, VALUEIN, KEYOUT, VALUEOUT>
 */
public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable> {

	// 对每一行句子进行map-split（相当于一个切分）
	protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, IntWritable>.Context context)
			throws IOException, InterruptedException {
		
		String[] words = StringUtils.split(value.toString(), ' ');
		for (String word : words) {
			context.write(new Text(word), new IntWritable(1));
		}
	}

}

```
2、汇总类：WordCountReducer

``` stylus
package com.sound.mr.wc;

import java.io.IOException;

import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {

	// key唯一，将数字相加
	protected void reduce(Text arg0, Iterable<IntWritable> arg1,
			Reducer<Text, IntWritable, Text, IntWritable>.Context arg2) throws IOException, InterruptedException {
		 int sum =0 ;
		 for(IntWritable iw : arg1) {
			 sum+=iw.get();
		 }
		 arg2.write(arg0, new IntWritable(sum));
	}

}

```
3、实施计算主类：MainJob

``` stylus
package com.sound.mr.wc;


import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.FileSystem;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class MainJob {
	public static void main(String[] args) {
		Configuration conf = new Configuration();
		try {
			Job job = Job.getInstance(conf);
			job.setJarByClass(MainJob.class);
			job.setJobName("WordCount");
			
			job.setMapOutputKeyClass(Text.class);
			job.setMapOutputValueClass(IntWritable.class);
			
			job.setMapperClass(WordCountMapper.class);
			job.setReducerClass(WordCountReducer.class);
			
			FileInputFormat.addInputPath(job, new Path("/usr/input/"));
			
			Path outPath = new Path("/usr/output/wc");
			FileSystem fs = FileSystem.get(conf);
			if(fs.exists(outPath)){
				fs.delete(outPath,true);
			}
			FileOutputFormat.setOutputPath(job, outPath);
			
			boolean finished = job.waitForCompletion(true);
			
			if(finished) {
				System.out.println("finished success!");
			}
		} catch (Exception e) {
			e.printStackTrace();
		}
	}
}

```
4、DFS处，人工创建输入输出文件（这里的输出文件使用代码创建）
![创建文件][6]
可能会存在创建失败的情况，可能是权限失败，需要修改权限。参照另一个博客[错误记录][7]

* 将程序打包
由于windows平台并没有相关hadoop环境，需要复制到CentOS系统执行，这里选择node3
![打包流程][8]

* hadoop运行及结果

``` stylus
hadoop jar wc.jar com.sound.mr.wc.MainJob
```
![结果][9]


  [1]: https://blog.csdn.net/soundslow/article/details/80101146
  [2]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524822445936.jpg
  [3]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524822699237.jpg
  [4]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524822731474.jpg
  [5]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524822842508.jpg
  [6]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524823055468.jpg
  [7]: https://blog.csdn.net/soundslow/article/details/80111713
  [8]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524823321658.jpg
  [9]: http://osiy4s0ad.bkt.clouddn.com/soundblog/1524823604330.jpg
