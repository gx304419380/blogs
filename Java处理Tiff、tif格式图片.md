上代码：
maven依赖
```
<dependencies>
        <!-- https://mvnrepository.com/artifact/javax.media/jai-core -->
        <dependency>
            <groupId>javax.media</groupId>
            <artifactId>jai-core</artifactId>
            <version>1.1.3</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/javax.media/jai_codec -->
        <dependency>
            <groupId>javax.media</groupId>
            <artifactId>jai_codec</artifactId>
            <version>1.1.3</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/javax.media/jai_imageio -->
        <dependency>
            <groupId>javax.media</groupId>
            <artifactId>jai_imageio</artifactId>
            <version>1.1</version>
        </dependency>


    </dependencies>
```


读取图片：
```
        File file = new File(path);
        //读取tif图片
        ImageReader reader = ImageIO.getImageReadersByFormatName("tiff").next();
        FileImageInputStream inputStream = new FileImageInputStream(file);

        reader.setInput(inputStream);
        return reader.read(0);
```

绘制图片：
```
        //将结果画出来
        ImageWriter writer = ImageIO.getImageWritersByFormatName("tiff").next();
        writer.setOutput(new FileImageOutputStream(new File("./output.tif")));
        writer.write(originalImage);
```


获取图像属性值：

```
        image.getRGB(x, y);
        int width = originalImage.getWidth();
        int height = originalImage.getHeight();
        int minX = originalImage.getMinX();
        int minY = originalImage.getMinY();

```

注意事项：打jar包后无法运行，需要修改MANIFEST
或者修改maven打包参数
```
Solution
Include the following lines in your MANIFEST.MF file:

Specification-Title: Java Advanced Imaging Image I/O Tools
Specification-Version: 1.1
Specification-Vendor: Sun Microsystems, Inc.
Implementation-Title: com.sun.media.imageio
Implementation-Version: 1.1
Implementation-Vendor: Sun Microsystems, Inc.
The values for each property can be anything (I used my specific application/version/company), as long as all six are defined.

maven打包参数
<plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>2.5.5</version>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>com.fly.image.Processor</mainClass>
                            <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
                            <addDefaultSpecificationEntries>true</addDefaultSpecificationEntries>
                        </manifest>
                        <manifestEntries>
                            <Specification-Vendor>MyCompany</Specification-Vendor>
                            <Implementation-Vendor>MyCompany</Implementation-Vendor>
                        </manifestEntries>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
            </plugin>

```
一个Tif多图按照类型权重取最大值程序：
```
package com.fly.image;

import javax.imageio.ImageIO;
import javax.imageio.ImageReader;
import javax.imageio.ImageWriter;
import javax.imageio.spi.IIORegistry;
import javax.imageio.stream.FileImageInputStream;
import javax.imageio.stream.FileImageOutputStream;
import java.awt.image.BufferedImage;
import java.io.BufferedReader;
import java.io.File;
import java.io.IOException;
import java.io.InputStreamReader;
import java.util.*;
import java.util.concurrent.atomic.AtomicInteger;
import java.util.stream.Collectors;
import java.util.stream.Stream;


public class Processor {

    static ArrayList<PixelInfoList> dataList = new ArrayList<>(1024);
    static ArrayList<ImageWithWeight> imageList = new ArrayList<>();

    public static void main(String[] args) throws IOException {

        BufferedReader scan = new BufferedReader(new InputStreamReader(System.in ));
        System.out.println("请输入要处理的图片数目，按回车确认：");
        int n = Integer.parseInt(scan.readLine());
        for (int i = 0; i < n; i++) {
            System.out.println("输入第 " + (i + 1) + " 张图片的路径，按回车确认：");
            String path = scan.readLine();
            System.out.println("输入第 " + (i + 1) + " 张图片的权重，按回车确认：");
            double weight = Double.parseDouble(scan.readLine());

            BufferedImage image = readTifFile(path);
            imageList.add(new ImageWithWeight(image, weight));
        }
        scan.close();
        if (imageList.isEmpty()) {
            System.out.println("未发现图片，退出...");
            return;
        }

        BufferedImage originalImage = imageList.get(0).image;
        int width = originalImage.getWidth();
        int height = originalImage.getHeight();
        int minX = originalImage.getMinX();
        int minY = originalImage.getMinY();
        int imageSize = width * height;

        System.out.println("图片参数： width=" + width + ",height=" + height + ".");

        //初始化数组
        dataList.ensureCapacity(imageSize);

        //开始读取图片像素，将像素信息放入数组中
        for (int i = minX; i < width; i++) {
            for (int j = minY; j < height; j++) {
                PixelInfoList pixelInfoList = new PixelInfoList(i, j);
                for (ImageWithWeight iw : imageList) {
                    //获取像素颜色值
                    int pixel = iw.getPixel(i, j);
                    PixelType pixelType = new PixelType(pixel, iw.weight);
                    pixelInfoList.list.add(pixelType);
                }
                dataList.add(pixelInfoList);
            }
        }


        HashMap<Integer, Double> map = new HashMap<>();
        //开始计算各个像素的加权最优值
        dataList.forEach(pixelInfo -> {
            map.clear();
            pixelInfo.list.forEach(pixel -> map.compute(pixel.type, (k, v) -> v == null ? pixel.weight : v + pixel.weight));
            List<Map.Entry<Integer, Double>> collect = map.entrySet()
                    .stream()
                    .sorted((a, b) -> a.getValue() - b.getValue() > 0 ? -1 : 1)
                    .collect(Collectors.toList());

            originalImage.setRGB(pixelInfo.x, pixelInfo.y, collect.get(0).getKey());
        });


        //将结果画出来
        ImageWriter writer = ImageIO.getImageWritersByFormatName("tiff").next();
        writer.setOutput(new FileImageOutputStream(new File("./output.tif")));
        writer.write(originalImage);
    }


    //读取tif文件
    private static BufferedImage readTifFile(String path) throws IOException {
        File file = new File(path);
        //读取tif图片
        ImageReader reader = ImageIO.getImageReadersByFormatName("tiff").next();
        FileImageInputStream inputStream = new FileImageInputStream(file);

        reader.setInput(inputStream);
        return reader.read(0);
    }

}




/************************************************************/
package com.fly.image;

public class PixelType {

    int type;
    double weight;

    public PixelType(int type, double weight) {
        this.type = type;
        this.weight = weight;
    }

    @Override
    public String toString() {
        return "PixelType{" +
                "type=" + type +
                ", weight=" + weight +
                '}';
    }
}


/********************************************************************/
package com.fly.image;

import java.util.ArrayList;

public class PixelInfoList {

    ArrayList<PixelType> list = new ArrayList<>();
    int x;
    int y;

    public PixelInfoList(int x, int y) {
        this.x = x;
        this.y = y;
    }

    public PixelInfoList() {
    }
}

/*********************************************************/
package com.fly.image;

import java.awt.image.BufferedImage;

public class ImageWithWeight {

    BufferedImage image;
    double weight;

    public ImageWithWeight(BufferedImage image, double weight) {
        this.image = image;
        this.weight = weight;
    }

    public int getPixel(int x, int y) {
        return this.image.getRGB(x, y);
    }

    @Override
    public String toString() {
        return "ImageWithWeight{" +
                "image=" + image +
                ", weight=" + weight +
                '}';
    }
}

```

maven配置
```
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.fly</groupId>
    <artifactId>image</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>jar</packaging>

    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-assembly-plugin</artifactId>
                <version>2.5.5</version>
                <configuration>
                    <archive>
                        <manifest>
                            <mainClass>com.fly.image.Processor</mainClass>
                            <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
                            <addDefaultSpecificationEntries>true</addDefaultSpecificationEntries>
                        </manifest>
                        <manifestEntries>
                            <Specification-Vendor>MyCompany</Specification-Vendor>
                            <Implementation-Vendor>MyCompany</Implementation-Vendor>
                        </manifestEntries>
                    </archive>
                    <descriptorRefs>
                        <descriptorRef>jar-with-dependencies</descriptorRef>
                    </descriptorRefs>
                </configuration>
            </plugin>
        </plugins>
    </build>

    <dependencies>
        <!-- https://mvnrepository.com/artifact/javax.media/jai-core -->
        <dependency>
            <groupId>javax.media</groupId>
            <artifactId>jai-core</artifactId>
            <version>1.1.3</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/javax.media/jai_codec -->
        <dependency>
            <groupId>javax.media</groupId>
            <artifactId>jai_codec</artifactId>
            <version>1.1.3</version>
        </dependency>

        <!-- https://mvnrepository.com/artifact/javax.media/jai_imageio -->
        <dependency>
            <groupId>javax.media</groupId>
            <artifactId>jai_imageio</artifactId>
            <version>1.1</version>
        </dependency>


    </dependencies>

</project>
```
