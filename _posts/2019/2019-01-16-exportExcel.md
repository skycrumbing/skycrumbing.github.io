---
layout: post
title: 后台导出excel
tags:
- tool
categories: java
description: 后台导出excel
---
## excel导出
excel导出可以在前端进行，也可以在后端进行，本篇主要针对后台导出

<!-- more -->

## 三种方式  
HSSFWorkbook：针对03前的版本，扩展名为.xls。一般不用这个。  
XSSFWorkbook：针对03前的版本，扩展名.xlsx。当数据过多会发生内存溢出。  
SXSSFWorkbook：通过将过多的数据写入临时文件的方式保证内存不会溢出。  
## 流程  
1, 创建sheet  
2, 创建row    
3, 创建cell    
4, 在cell创建写入内容     
## 创建模板test1.xlsx
在C:\\Users\\Administrator\\Desktop下创建test1.xlsx  
![test1.xlsx](\assets\img\exportExcel_1.jpg)  
![test1.xlsx](\assets\img\exportExcel_2.jpg)  
## 代码  
```
	package com.tantao;

	import org.apache.poi.ss.usermodel.Cell;
	import org.apache.poi.ss.usermodel.RichTextString;
	import org.apache.poi.ss.usermodel.Row;
	import org.apache.poi.ss.usermodel.Sheet;
	import org.apache.poi.xssf.streaming.SXSSFSheet;
	import org.apache.poi.xssf.streaming.SXSSFWorkbook;
	import org.apache.poi.xssf.usermodel.XSSFWorkbook;

	import java.io.File;
	import java.io.FileInputStream;
	import java.io.FileOutputStream;
	import java.io.IOException;
	import java.util.Calendar;
	import java.util.Date;
	import java.util.HashMap;
	import java.util.Map;

	/**
	 * Created by Administrator on 2019/1/16.
	 */
	public class ExcelSXSSFWriter {
		public static void main(String[] args) throws IOException{
			//输入模板文件
			XSSFWorkbook xssfWorkbook = new XSSFWorkbook(new FileInputStream("C:\\Users\\Administrator\\Desktop\\test1.xlsx"));
			SXSSFWorkbook workbook = new SXSSFWorkbook(xssfWorkbook, 10000);

			//导出文件
			File file = new File("C:\\Users\\Administrator\\Desktop\\test.xlsx");

			//读取第一页第一行第一个单元格
			String s = xssfWorkbook.getSheetAt(0).getRow(0).getCell(0).getStringCellValue();
			System.out.println(s);

			//插入数据
			for(int i = 0; i < 2; i++){
				Sheet sheet = workbook.getSheet("Sheet" + (i + 1));
				//获取模板文件最新的一行
				int lastRowNum = xssfWorkbook.getSheetAt(i).getLastRowNum();
				if(lastRowNum > 0){
					lastRowNum++;
				}
				System.out.println(lastRowNum);
				if (sheet == null) {
					sheet = workbook.createSheet("sheet" + (i + 1));
				}
				//生成标题
				Map<Integer, Object> firstTitles = new HashMap<>();
				firstTitles.put(0, "部门");
				firstTitles.put(1, "test12221");
				firstTitles.put(7, "时间:");
				firstTitles.put(8, "2017-09-11");
				genSheetHead(sheet, 0 + lastRowNum, firstTitles);
				Map<Integer, Object> twoTitles = new HashMap<>();
				twoTitles.put(0, "工号：");
				twoTitles.put(1, "test12221");
				twoTitles.put(2, "姓名:");
				twoTitles.put(3, "aaaa");
				genSheetHead(sheet, 1 + lastRowNum, twoTitles);
				//内容
				for (int rownum = 2 + lastRowNum; rownum < 10000; rownum++) {
					Row row = sheet.createRow(rownum);
					int k = -1;
					createCell(row, ++k, "第 " + rownum + " 行");
					createCell(row, ++k, "34343.123456");
					createCell(row, ++k, "23.67%");
					createCell(row, ++k, "12:12:23");
					createCell(row, ++k, "2014-10-<11 12:12:23");
					createCell(row, ++k, "true");
					createCell(row, ++k, "false");
					createCell(row, ++k, "fdsa");
					createCell(row, ++k, "123");
					createCell(row, ++k, "321");
					createCell(row, ++k, "3213");
					createCell(row, ++k, "321");
					createCell(row, ++k, "321");
					createCell(row, ++k, "43432");
					createCell(row, ++k, "54");
					createCell(row, ++k, "fal45se");
					createCell(row, ++k, "fal6se");
					createCell(row, ++k, "fal64321se");
					createCell(row, ++k, "fal43126se");
					createCell(row, ++k, "432432");
					createCell(row, ++k, "432432");
					createCell(row, ++k, "r54");
					createCell(row, ++k, "543");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1a");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
					createCell(row, ++k, "few1");
				}
			}

			FileOutputStream out = new FileOutputStream(file);
			workbook.write(out);
			out.close();
		}
	
   		 /**
    		 * 创建单元格
    		 */
		private static void createCell(Row row, int cellNum, String value) {
			Cell cell = row.createCell(cellNum);
			generateValue(value, cell);
		}
		
   		 /**
    		 * 生成标题
    		 */
		private static void genSheetHead(Sheet sheet, int rowNum, Map<Integer, Object> firstTitles) {
			Row row = sheet.createRow(rowNum);
			for(Integer i: firstTitles.keySet()){
				Cell cell = row.createCell(i);
				Object object = firstTitles.get(i);
				generateValue(object, cell);
			}
		}

   		 /**
    		 * 在单元格中写入内容
    		 */
		private static void generateValue(Object object, Cell cell) {
			if(object instanceof String){
				cell.setCellValue((String) object);
			}
			else if(object instanceof Boolean){
				cell.setCellValue((Boolean) object);
			}
			else if(object instanceof Double){
				cell.setCellValue((Double) object);
			}
			else if(object instanceof Date){
				cell.setCellValue((Date) object);
			}
			else if(object instanceof Date){
				cell.setCellValue((Date) object);
			}
			else if(object instanceof Calendar){
				cell.setCellValue((Calendar) object);
			}
			else if (object instanceof RichTextString) {
				cell.setCellValue((RichTextString) object);
			}
		}
	}
```
## 查看test.xlsx
在C:\\Users\\Administrator\\Desktop下查看test.xlsx  
![test1.xlsx](\assets\img\exportExcel_3.jpg)  
![test1.xlsx](\assets\img\exportExcel_4.jpg)  
