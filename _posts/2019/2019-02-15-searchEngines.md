---
layout: post
title: 搜索引擎技术
tags:
- tool
categories: java
description: 搜索引擎技术
---
## 搜索引擎
类似于百度，谷歌的搜索引擎技术，Java也提供了相关的搜索引擎工具包Lucene，而Solr和 ElasticSearch则是对Lucene进行封装后的搜索引擎服务器

<!-- more -->

## Lucene
1，准备中文分词器  
2，将所有数据添加到索引里  
3，创建查询器  
4，搜索  
### 什么是索引  
一个索引就相当于数据库中的一张表，表里可以有多个字段，同样索引也是如此，搜索的时候就可以搜索相应的字段  
### 代码  
```
	package com.tantao;

	import java.io.IOException;
	import java.io.StringReader;
	import java.util.ArrayList;
	import java.util.List;

	import org.apache.lucene.analysis.TokenStream;
	import org.apache.lucene.document.Document;
	import org.apache.lucene.document.Field;
	import org.apache.lucene.document.TextField;
	import org.apache.lucene.index.*;
	import org.apache.lucene.queryparser.classic.QueryParser;
	import org.apache.lucene.search.IndexSearcher;
	import org.apache.lucene.search.Query;
	import org.apache.lucene.search.ScoreDoc;
	import org.apache.lucene.search.TopDocs;
	import org.apache.lucene.search.highlight.Highlighter;
	import org.apache.lucene.search.highlight.InvalidTokenOffsetsException;
	import org.apache.lucene.search.highlight.QueryScorer;
	import org.apache.lucene.search.highlight.SimpleHTMLFormatter;
	import org.apache.lucene.store.Directory;
	import org.apache.lucene.store.RAMDirectory;
	import org.wltea.analyzer.lucene.IKAnalyzer;

	public class TestLucene {

		public static void main(String[] args) throws Exception {
			// 1. 准备中文分词器
			IKAnalyzer analyzer = new IKAnalyzer();

			//2.添加索引
			List<String> productList = new ArrayList();
			productList.add("飞利浦led灯泡e27螺口暖白球泡灯家用照明超亮节能灯泡转色温灯泡");
			productList.add("飞利浦led灯泡e14螺口蜡烛灯泡3W尖泡拉尾节能灯泡暖黄光源Lamp");
			productList.add("雷士照明 LED灯泡 e27大螺口节能灯3W球泡灯 Lamp led节能灯泡");
			productList.add("飞利浦 led灯泡 e27螺口家用3w暖白球泡灯节能灯5W灯泡LED单灯7w");
			productList.add("飞利浦led小球泡e14螺口4.5w透明款led节能灯泡照明光源lamp单灯");
			productList.add("飞利浦蒲公英护眼台灯工作学习阅读节能灯具30508带光源");
			productList.add("欧普照明led灯泡蜡烛节能灯泡e14螺口球泡灯超亮照明单灯光源");
			productList.add("欧普照明led灯泡节能灯泡超亮光源e14e27螺旋螺口小球泡暖黄家用");
			productList.add("聚欧普照明led灯泡节能灯泡e27螺口球泡家用led照明单灯超亮光源");
			Directory index = createIndex(analyzer, productList);

			//删除序号为5索引
	//		IndexWriterConfig config = new IndexWriterConfig(analyzer);
	//		IndexWriter indexWriter = new IndexWriter(index, config);
	//		indexWriter.deleteDocuments(new Term("id","5"));
	//		indexWriter.commit();
	//		indexWriter.close();

			//3.创建查询器
			String keyword = "护眼带光源";
			//根据每个document的name,中文分词器以及关键字创建查询器
			Query query = new QueryParser("name",analyzer).parse(keyword);

			//4.搜索
			//创建reader
			IndexReader reader = DirectoryReader.open(index);
			//基于reader创建搜索器
			IndexSearcher searcher = new IndexSearcher(reader);
			//制定每页要显示多少条数据
			int numberPerPage = 3;
			System.out.printf("当前一共有%d条数据%n",productList.size());
			System.out.printf("查询关键字是：\"%s\"%n",keyword);
			//执行搜索，返回搜索结果
			TopDocs topDocs = searcher.search(query, numberPerPage);
			System.out.println("查询到的总条数为" + topDocs.totalHits);
			ScoreDoc[] hits = topDocs.scoreDocs;

			//5.显示搜索结果
			showSearchResults(searcher, hits, query, analyzer);
		}

		private static void showSearchResults(IndexSearcher searcher, ScoreDoc[] hits, Query query, IKAnalyzer analyzer) throws IOException, InvalidTokenOffsetsException {
			System.out.println("找到" + hits.length + "个命中");
			System.out.println("序号\t匹配度得分\t结果");
			//高亮格式
			SimpleHTMLFormatter formatter = new SimpleHTMLFormatter("<span style='color:red'>", "</span>");
			Highlighter highlighter = new Highlighter(formatter, new QueryScorer(query));
			//遍历搜索结果
			for(int i = 0; i < hits.length; i++){
				ScoreDoc scoreDoc = hits[i];
				//获取每个结果的主键
				int docId = scoreDoc.doc;
				//根据主键获得每个document
				Document document = searcher.doc(docId);
				List<IndexableField> fields = document.getFields();
				System.out.print((i + 1));
				//得到每个document的匹配度得分
				System.out.print("\t" + scoreDoc.score);
				//根据Field字段获得相应的值
				for (IndexableField f: fields){
					if ("name".equals(f.name())) {
						TokenStream tokenStream = analyzer.tokenStream(f.name(), new StringReader(document.get(f.name())));
						String fildContent = highlighter.getBestFragment(tokenStream,document.get(f.name()));
						System.out.print("\t" + fildContent);
					}
					else {
						System.out.print("\t" + document.get(f.name()));
					}
				}
				System.out.println();
			}
		}

		private static Directory createIndex(IKAnalyzer analyzer, List<String> productList) throws IOException {
			//创建内存索引
			Directory index = new RAMDirectory();
			//根据中文分词器创建配置对象
			IndexWriterConfig config = new IndexWriterConfig(analyzer);
			//创建索引write
			IndexWriter writer = new IndexWriter(index, config);
			//依次写入内存索引
			int id = 0;
			for(String name: productList){
				addDoc(writer, id, name);
				id++;
			}
			writer.close();
			return index;
		}

		private static void addDoc(IndexWriter writer, int id,  String name) throws IOException {
			//每条数据创建一个document
			Document doc = new Document();
			//将每个产品映射到name字段
			doc.add(new TextField("id", String.valueOf(id), Field.Store.YES));
			doc.add(new TextField("name", name, Field.Store.YES));
			writer.addDocument(doc);
		}
	}
```

## 结果
![结果](\assets\img\searchEngines_1.jpg)  
## Solr
Solr被做成了 webapp形式，以tomcat的应用的方式启动，提供了可视化的配置界面。将Solr下载解压，在相应的bin目录下打开cmd,输入solr.cmd start即可启动服务器。  
**Solr需要配置中文分词插件**  
输入地址http://127.0.0.1:8983/solr/#/即可访问服务器界面  
### 相关概念  
**core**：一个core就相当于一个索引  
### 导入索引数据  
1，可以在网页上引入（但是不可以批量引入）  
2，使用solr提供的java客户端工具包引入(网上搜索教程)  
### 查询索引数据  
使用solr提供的java客户端工具包查询(网上搜索教程)  
## ElasticSearch  
ElasticSearch也是基于Lucene封装的服务器，同样要安装中文分词插件  
### Kibana  
Kibana是分析ElasticSearch服务器存储的索引数据以及导入数据的工具，其本质也是一个web服务器，并且提供的是基于restful风格的服务  
### java工具包
同样ElasticSearch提供java客户端工具包对其进行操作  
