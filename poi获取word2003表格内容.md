上代码：

maven依赖：
```
        <!-- https://mvnrepository.com/artifact/org.apache.poi/poi -->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi</artifactId>
            <version>4.1.0</version>
        </dependency>

        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-ooxml</artifactId>
            <version>4.1.0</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.poi/poi-scratchpad -->
        <dependency>
            <groupId>org.apache.poi</groupId>
            <artifactId>poi-scratchpad</artifactId>
            <version>4.1.0</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/junit/junit -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>

        <!-- https://mvnrepository.com/artifact/org.apache.commons/commons-io -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-io</artifactId>
            <version>1.3.2</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.apache.commons/commons-lang3 -->
        <dependency>
            <groupId>org.apache.commons</groupId>
            <artifactId>commons-lang3</artifactId>
            <version>3.9</version>
        </dependency>

```


word工具类
```
package com.fly.word;

import org.apache.poi.hwpf.HWPFDocument;
import org.apache.poi.hwpf.usermodel.*;
import org.apache.poi.xwpf.usermodel.XWPFDocument;
import org.apache.poi.xwpf.usermodel.XWPFTable;
import org.apache.poi.xwpf.usermodel.XWPFTableCell;
import org.apache.poi.xwpf.usermodel.XWPFTableRow;

import java.io.FileInputStream;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

public class WordUtils {

    public static List<String> readDoc(String path) {
        HWPFDocument document;
        try {
            document = new HWPFDocument(new FileInputStream(path));
        } catch (IOException e) {
            e.printStackTrace();
            throw new RuntimeException("read word error");
        }
        // 获取所有表格
        Range range = document.getRange();// 得到文档的读取范围
        TableIterator it = new TableIterator(range);

        List<String> contentList = new ArrayList<>();
        while (it.hasNext()) {
            Table tb = it.next();
            //迭代行，默认从0开始
            for (int i = 0; i < tb.numRows(); i++) {
                TableRow tr = tb.getRow(i);
                //迭代列，默认从0开始
                for (int j = 0; j < tr.numCells(); j++) {
                    TableCell td = tr.getCell(j);//取得单元格
                    //取得单元格的内容
                    //取得单元格的内容
                    for(int k = 0; k < td.numParagraphs(); k++){
                        Paragraph para = td.getParagraph(k);
                        String s = para.text();
                        //去除后面的特殊符号
                        if(null != s && !"".equals(s)){
                            s = s.substring(0, s.length()-1);
                            contentList.add(s);
                        } else {
                            contentList.add(null);
                        }
                    }
                }
            }
        }
        return contentList;
    }

}

@Data
public class DstData {

    private String number;

    private String name;

    private String time;

    private String receiver;

    private String describe;

    public DstData(List<String> contentList) {

        for (int i = 0; i < contentList.size(); i++) {
            String content = contentList.get(i).trim();
            if ("发文（邮件）编号".equals(content)) {
                String s = contentList.get(i + 1);
                if (StringUtils.isNotBlank(s) && number == null) {
                    number = s.replaceAll("发", "");
                }
            }

            if ("发文（邮件）日期".equals(content)) {
                String s = contentList.get(i + 1);
                if (StringUtils.isNotBlank(s) && time == null) {
                    time = s;
                }
            }

            if ("发文（邮件）名称".equals(content)) {
                String s = contentList.get(i + 1);
                if (StringUtils.isNotBlank(s) && name == null) {
                    name = s;
                }
            }

            if ("发送：".equals(content)) {
                String s = contentList.get(i + 1);
                if (StringUtils.isNotBlank(s) && receiver == null) {
                    receiver = s;
                }
            }


        }
    }
}

public static void main(String[] args) {

        String path = args[0];
        String dstPath;
        if (args.length > 1) {
            dstPath = args[1];
        } else {
            dstPath = "D:/";
        }

        File file = new File(path);
        File[] files = file.listFiles();
        if (files == null || files.length == 0) {
            System.out.println("文件夹为空！");
            return;
        }

        List<DstData> list = new ArrayList<>();
        int i = 1;
        for (File f : files) {
            System.out.println("开始处理第" + i++ + "个文件：【" + f.getName() + "】");
            try {
                List<String> contentList = WordUtils.readDoc(f.getAbsolutePath());
                DstData dstData = new DstData(contentList);
                list.add(dstData);
                System.out.println("处理文件：【" + f.getName() + "】完成");
            } catch (Exception e) {
                System.out.println("文件：【" + f.getName() + "】 转换失败，请手动转换");
            }
        }

        String dstFile = ExcelUtils.writeToExcel(list, dstPath);
        System.out.println("处理完成，文件路径为：" + dstFile);
    }

```


写入excel工具类：
```
package com.fly.word;

import org.apache.commons.collections4.CollectionUtils;
import org.apache.commons.io.FileUtils;
import org.apache.commons.lang3.StringUtils;
import org.apache.poi.hssf.usermodel.*;
import org.apache.poi.ss.usermodel.HorizontalAlignment;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.io.IOException;
import java.lang.reflect.Field;
import java.text.Format;
import java.text.SimpleDateFormat;
import java.time.LocalDateTime;
import java.time.format.DateTimeFormatter;
import java.util.*;

public class ExcelUtils {

    /** 默认sheet名称 */
    private static final String DEFAULT_SHEET_NAME = "sheet";
    /** 单个Sheet页最大行数 */
    private static final int SINGLE_SHEET_MAX_ROWS = 65535;
    /** 默认单元格宽度 */
    private static final int DEFAULT_CELL_WIDTH = 5000;
    /** 默认日期格式 */
    private static final String DEFAULT_DATE_PATTERN = "yyyy-MM-dd HH:mm:ss";
    /** 默认日期格式 */
    private static final ThreadLocal<SimpleDateFormat> DEFAULT_DATE_FORMAT = new ThreadLocal<SimpleDateFormat>(){
        @Override
        protected SimpleDateFormat initialValue() {
            return new SimpleDateFormat(DEFAULT_DATE_PATTERN);
        }
    };



    public static String writeToExcel(List<DstData> list, String dstPath) {
        String format = LocalDateTime.now().format(DateTimeFormatter.ofPattern("yyyyMMddHHmmss"));
        String path = dstPath + "data" + format + ".xls";
        File file = new File(path);
        List<String> titleList = Arrays.asList("序号", "发文名称", "发送时间", "收件人");
        List<String> fieldList = Arrays.asList("number", "name", "time", "receiver");
        try {
            byte[] excel = createExcel(titleList, list, fieldList);
            if(file.exists()) {
                file.delete();
            }
            file.createNewFile();
            FileUtils.writeByteArrayToFile(file, excel);
            return path;
        } catch (Exception e) {
            System.out.println("创建excel失败！");
        }
        return null;
    }




    /**
     * 根据列标题和列数据生成excel表格文件
     * excel默认一个sheet且sheet名称默认为sheet1
     * 日期格式：yyyy-MM-dd HH:mm:ss
     * list格式：XXX,XXXX,XXXX
     * map格式：a:1,b:2
     *
     * @param columnTitles
     *            列标题
     * @param datas
     *            列数据，只支持对象的属性使用string、date、Number数字类型和list、map，有需要请扩展
     * @param columnFields
     *            列标题对应对象的属性名（依次与列标题对应）
     * @return
     * @throws Exception
     */
    public static byte[] createExcel(List<String> columnTitles, List<? extends Object> datas, List<String> columnFields) throws Exception {
        return createExcel(null, columnTitles, datas, columnFields,null);
    }

    /**
     * 根据列标题和列数据生成excel表格文件，生成excel默认一个sheet且sheet名称默认为sheet1
     * 日期格式：yyyy-MM-dd HH:mm:ss
     * list格式：XXX,XXXX,XXXX
     * map格式：a:1,b:2
     *
     * @param columnTitles
     *            列标题
     * @param datas
     *            列数据，只支持对象的属性使用string、date、Number数字类型和list、map，有需要请扩展
     * @param columnFields
     *            列标题对应对象的属性名（依次与列标题对应）
     * @param customFieldFormats
     *            每列数据自定义format,没有为空，list类型属性:格式化其子元素，map类型属性: 格式化其子元素的value
     * @return
     * @throws Exception
     */
    public static byte[] createExcel( List<String> columnTitles, List<? extends Object> datas, List<String> columnFields,
                                      Map<String,? extends Format> customFieldFormats) throws Exception {
        return createExcel(null, columnTitles, datas, columnFields, customFieldFormats);
    }

    /**
     * 根据列标题和列数据生成excel表格文件
     * excel默认一个sheet且sheet名称默认为sheet1
     * 日期格式：yyyy-MM-dd HH:mm:ss
     * list格式：XXX,XXXX,XXXX
     * map格式：a:1,b:2
     *
     * @param fileName
     *              文件名称
     * @param columnTitles
     *            列标题
     * @param datas
     *            列数据，只支持对象的属性使用string、date、Number数字类型和list、map，有需要请扩展
     * @param columnFields
     *            列标题对应对象的属性名（依次与列标题对应）
     * @return
     * @throws Exception
     */
    public static File createExcel(String fileName, List<String> columnTitles, List<? extends Object> datas, List<String> columnFields) throws Exception {
        byte[] fileBytes = createExcel(null, columnTitles, datas, columnFields,null);
        File file = createTmpFile(fileName);
        FileUtils.writeByteArrayToFile(file, fileBytes);
        return file;
    }

    /**
     * 根据列标题和列数据生成excel表格文件，excel文件默认一个sheet
     * 日期格式：yyyy-MM-dd HH:mm:ss
     * list格式：XXX,XXXX,XXXX
     * map格式：a:1,b:2
     *
     * @param sheetName
     *            生成的excel文件的sheet名
     * @param columnTitles
     *            列标题
     * @param datas
     *            列数据，只支持对象的属性使用string、date、Number数字类型和list、map，有需要请扩展
     * @param columnFields
     *            列标题对应对象的属性名（依次与列标题对应）
     * @param customFieldFormats
     *              对象属性自定义format,没有为空，list类型属性:格式化其子元素，map类型属性: 格式化其子元素的value
     * @return 文件byte
     * @throws Exception
     */
    public static byte[] createExcel(String sheetName, List<String> columnTitles, List<? extends Object> datas,
                                     List<String> columnFields, Map<String,? extends Format> customFieldFormats) throws Exception {
        if (datas == null || columnTitles == null) {
            throw new IllegalArgumentException("illegal data: data or columnTitles is null");
        }
        if (columnTitles.size() != columnFields.size()) {
            throw new IllegalArgumentException("every column title should have its mapped column field name");
        }
//        File file = null;
        HSSFWorkbook workbook = new HSSFWorkbook();
        int dataSize = datas.size();
        int part = dataSize / SINGLE_SHEET_MAX_ROWS;
        if(dataSize > SINGLE_SHEET_MAX_ROWS) {
            sheetName = StringUtils.isNotBlank(sheetName) ? sheetName : DEFAULT_SHEET_NAME;
            for(int i = 0; i < part; i++) {
                String tmpSheetName = String.format("%s%d", sheetName, i + 1);
                createSheetAndWriteData(tmpSheetName, columnTitles, datas.subList(0, SINGLE_SHEET_MAX_ROWS), columnFields,
                                        customFieldFormats, workbook);
                datas.subList(0, SINGLE_SHEET_MAX_ROWS).clear();
            }
            if(CollectionUtils.isNotEmpty(datas)) {
                sheetName = String.format("%s%d", sheetName, part + 1);
                createSheetAndWriteData(sheetName, columnTitles, datas, columnFields, customFieldFormats, workbook);
            }
        } else {
            sheetName = StringUtils.isNotBlank(sheetName) ? sheetName : DEFAULT_SHEET_NAME + "1";
            createSheetAndWriteData(sheetName, columnTitles, datas, columnFields,customFieldFormats, workbook);
        }
        ByteArrayOutputStream os = new ByteArrayOutputStream();
        workbook.write(os);
        return os.toByteArray();
    }

    private static void createSheetAndWriteData(String sheetName, List<String> columnTitles, List<? extends Object> datas,
                                                List<String> columnFields, Map<String,? extends Format> customFieldFormats,
                                                HSSFWorkbook workbook) throws Exception {
        HSSFSheet sheet = workbook.createSheet(sheetName);
        initSheetHeaders(workbook, sheet, columnTitles);
        writeData(sheet, datas, columnFields,customFieldFormats);
    }

    private static void writeData(HSSFSheet sheet, List<? extends Object> datas, List<String> columnFields,
                                  Map<String,? extends Format> customFieldFormats) throws Exception {
        int startX = 1;
        int startY = 0;
        for (Object item : datas) {
            HSSFRow row = sheet.createRow(startX);
            for (int i = 0; i < columnFields.size(); i++) {
                String fieldName = columnFields.get(i);
                Field field = item.getClass().getDeclaredField(fieldName);
                field.setAccessible(true);
                Format fmt = null;
                if(customFieldFormats != null && customFieldFormats.containsKey(fieldName)) {
                    fmt = customFieldFormats.get(fieldName);
                }
                HSSFCell cell = row.createCell(startY++);
                String columnData = resolveFieldData(field.get(item),fmt);
                cell.setCellValue(columnData);
            }
            startX++;
            startY = 0;
        }
    }

    @SuppressWarnings("unchecked")
    private static String resolveFieldData(Object orgnColumnData,Format fmt) {
        String columnData = "";
        if (orgnColumnData == null) {
        } else if (orgnColumnData instanceof String) {
            columnData = fmt == null ? (String) orgnColumnData : fmt.format((String) orgnColumnData);
        } else if (orgnColumnData instanceof Date) {
            columnData = fmt ==null ? formatDate((Date) orgnColumnData) : fmt.format((Date) orgnColumnData);
        } else if (orgnColumnData instanceof Number) {
            columnData = fmt == null ? String.valueOf(orgnColumnData) : fmt.format((Number)orgnColumnData);
        } else if (orgnColumnData.getClass().isArray()) {
            List<Object> toList = Arrays.asList((Object[]) orgnColumnData);
            columnData = resolveList(toList,fmt);
        } else if (orgnColumnData instanceof List) {
            List<Object> toList = (List<Object>) orgnColumnData;
            columnData = resolveList(toList,fmt);
        } else if (orgnColumnData instanceof Map) {
            columnData = resolveMap((Map<Object, Object>) orgnColumnData,fmt);
        } else {
            throw new IllegalArgumentException("unsupported field type");
        }
        return columnData;
    }

    private static String resolveList(List<Object> list,Format fmt) {
        StringBuilder builder = new StringBuilder();
        for (Object obj : list) {
            builder.append(resolveFieldData(obj, fmt)).append(",");
        }
        if(builder.length() > 0) {
            return builder.substring(0, builder.length() - 1);
        }
        return builder.toString();
    }

    private static String resolveMap(Map<Object, Object> map,Format fmt) {
        StringBuilder builder = new StringBuilder();
        Iterator<Map.Entry<Object, Object>> iterator = map.entrySet().iterator();
        while(iterator.hasNext()) {
            Map.Entry<Object, Object> entry = iterator.next();
            String mKey = entry.getKey() != null ? entry.getKey().toString() : "";
            builder.append(mKey).append(":").append(resolveFieldData(entry.getValue(),fmt)).append(",");
        }
        if(builder.length() > 0) {
            return builder.substring(0, builder.length() - 1);
        }
        return builder.toString();
    }

    /**
     * 初始化表头
     *
     * @param wb
     * @param sheet
     * @param headers
     */
    private static void initSheetHeaders(HSSFWorkbook wb, HSSFSheet sheet, List<String> headers) {
        // 表头样式
        HSSFCellStyle style = wb.createCellStyle();
        // 创建一个居中格式
        style.setAlignment(HorizontalAlignment.CENTER);
        // 字体样式
        HSSFFont fontStyle = wb.createFont();
        fontStyle.setFontName("微软雅黑");
        fontStyle.setFontHeightInPoints((short) 12);
        fontStyle.setBold(true);
        style.setFont(fontStyle);
        // 生成sheet1内容
        // 第一个sheet的第一行为标题
        HSSFRow rowFirst = sheet.createRow(0);
        // 冻结第一行
        sheet.createFreezePane(0, 1, 0, 1);
        // 写标题
        for (int i = 0; i < headers.size(); i++) {
            // 获取第一行的每个单元格
            HSSFCell cell = rowFirst.createCell(i);
            // 设置每列的列宽
            sheet.setColumnWidth(i, DEFAULT_CELL_WIDTH);
            //加样式
            cell.setCellStyle(style);
            //往单元格里写数据
            cell.setCellValue(headers.get(i));
        }
    }

    private static String formatDate(Date date) {
        return DEFAULT_DATE_FORMAT.get().format(date);
    }

    private static File createTmpFile(String fileName) throws IOException {
        String tmpFileName = StringUtils.isNotBlank(fileName) ? fileName : new SimpleDateFormat("yyyyMMddHHmmss").format(new Date());
        File file = File.createTempFile(tmpFileName + "_", ".xls");
        file.deleteOnExit();
        return file;
    }
}

```
