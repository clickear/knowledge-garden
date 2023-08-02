---
title: html转pdf
date created: 2023-07-17
date modified: 2023-08-01
---

通过freemark将模板转换成html，后将html转pdf，方便扩展。支持水印

## 方案选择

itext,早期是可以商业的，后面itext5.0,7.0之后不可商用。itext不可商业后，基于itext4.x版本的代码，有人fork了分支。openpdf

html转pdf方案参考

flying-saucer-pdf: 基于Itext支持html，样式只支持css2.0  
xhtmlrenderer: 基于itext的开源实现，样式有些支持比flying多。如倾斜等  
itext7.0： 全新重构的itext版本，不支持商业  
wikihtml： 需要宿主机按照程序，未调研

## 代码展示

### 开源版本，可商用 参考[xhtmlrenderer 将html转换成pdf，完美css，带图片，手动分页，解决内容断开的问题 - 煮过的花朵 - 博客园](https://www.cnblogs.com/trisolaris2018/p/10754914.html)

1. 引入pom

```java
<!-- freemarker 读取html模板文件 -->  
<dependency>  
    <groupId>org.freemarker</groupId>  
    <artifactId>freemarker</artifactId>  
    <version>2.3.30</version>  
</dependency>  
<!-- xml 将html模板文件转换成pdf -->  
<dependency>  
    <groupId>org.xhtmlrenderer</groupId>  
    <artifactId>core-renderer</artifactId>  
    <version>R8</version>  
</dependency>
```

2. 工具类

```java
package com.lucky.attendance.common.utils;  
  
import com.lowagie.text.BadElementException;  
import com.lowagie.text.DocumentException;  
import com.lowagie.text.Element;  
import com.lowagie.text.Rectangle;  
import com.lowagie.text.pdf.*;  
import com.lowagie.text.pdf.codec.Base64;  
import freemarker.template.Configuration;  
import freemarker.template.Template;  
import freemarker.template.TemplateException;  
import lombok.extern.slf4j.Slf4j;  
import org.apache.commons.lang.StringUtils;  
import org.springframework.core.io.ClassPathResource;  
import org.xhtmlrenderer.extend.FSImage;  
import org.xhtmlrenderer.extend.ReplacedElement;  
import org.xhtmlrenderer.extend.ReplacedElementFactory;  
import org.xhtmlrenderer.extend.UserAgentCallback;  
import org.xhtmlrenderer.layout.LayoutContext;  
import org.xhtmlrenderer.pdf.ITextFSImage;  
import org.xhtmlrenderer.pdf.ITextFontResolver;  
import org.xhtmlrenderer.pdf.ITextImageElement;  
import org.xhtmlrenderer.pdf.ITextRenderer;  
import org.xhtmlrenderer.render.BlockBox;  
import org.xhtmlrenderer.simple.extend.FormSubmissionListener;  
  
import javax.servlet.http.HttpServletResponse;  
import javax.swing.*;  
import java.awt.*;  
import java.io.ByteArrayOutputStream;  
import java.io.IOException;  
import java.io.StringWriter;  
import java.net.URLEncoder;  
import java.util.Locale;  
  
/**  
 * @author  
 * @description  
 * @date 2022/3/3 18:37  
 * @since 1.0  
 */@Slf4j  
public class PdfTemplateUtil {  
  
    /**  
     * 水印旋转角度  
     */  
    public static final Integer DEFAULT_WATER_MARK_ANGEL = 45;  
  
    /**  
     * 水印文字透明度  
     */  
    private final static float DEFAULT_WATER_MARK_OPACITY = 0.1f;  
    /**  
     * 水印字体大小  
     */  
    private final static int DEFAULT_WATER_MARK_FONT_SIZE = 12;  
  
    /***  
     * 通过模板导出pdf文件  
     * @param data  
     * @param templateFileName  
     * @param waterMarker 水印文字内容。  
     * @throws Exception  
     */    public static ByteArrayOutputStream createPDF(Object data, String templateFileName, String waterMarker) throws Exception {  
        // 生成html  
        String html = generateHtml(data, templateFileName);  
  
        // 渲染成pdf  
        return convertToPdf(html, waterMarker);  
    }  
  
    /**  
     * 创建pdf，并输出到response  
     *     * @param data  
     * @param templateFileName  
     * @param fileName  
     * @param watermark 水印  
     * @param response  
     */  
    public static void createPDFAndFlush(Object data, String templateFileName, String fileName, String watermark, HttpServletResponse response) {  
        try (ByteArrayOutputStream outputStream = createPDF(data, templateFileName, watermark)) {  
            response.setContentType("application/x-msdownload");  
            // 告诉浏览器，当前响应数据要求用户干预保存到文件中，以及文件名是什么 如果文件名有中文，必须URL编码  
            fileName = URLEncoder.encode(fileName, "UTF-8");  
            response.setHeader("Content-Disposition", "attachment;filename=" + fileName);  
            outputStream.writeTo(response.getOutputStream());  
        } catch (Exception e) {  
            log.error("exportDetailData2Pdf exception:", e);  
        }  
    }  
  
    /**  
     * 使用freemarker生成html  
     *     * @param data  
     * @param templateFileName  
     * @return  
     * @throws IOException  
     * @throws TemplateException  
     */    private static String generateHtml(Object data, String templateFileName) throws IOException, TemplateException {  
        try (StringWriter writer = new StringWriter()) {  
            // 创建一个FreeMarker实例, 负责管理FreeMarker模板的Configuration实例  
            Configuration configuration = new Configuration(Configuration.DEFAULT_INCOMPATIBLE_IMPROVEMENTS);  
            // 指定FreeMarker模板文件的位置  
            configuration.setClassForTemplateLoading(PdfTemplateUtil.class, "/templates");  
            // 设置模板的编码格式  
            configuration.setEncoding(Locale.CHINA, "UTF-8");  
            //设置兼容${xxx}方式取值为空的情况  
            configuration.setClassicCompatible(true);  
  
            // 获取模板文件  
            Template template = configuration.getTemplate(templateFileName, "UTF-8");  
            // 将数据输出到html中  
            template.process(data, writer);  
            writer.flush();  
            return writer.toString();  
        }  
    }  
  
    public static ByteArrayOutputStream convertToPdf(String htmlStr, String watermark) throws Exception {  
        if (StringUtils.isBlank(htmlStr)) {  
            return null;  
        }  
        // itext开源版本实现  
        ITextRenderer renderer = new ITextRenderer();  
        ByteArrayOutputStream out = new ByteArrayOutputStream();  
        renderer.setPDFVersion(PdfWriter.VERSION_1_7);  
        renderer.setDocumentFromString(htmlStr);  
        // 解决base64图片支持问题  
        renderer.getSharedContext().setReplacedElementFactory(new B64ImgReplacedElementFactory());  
        ITextFontResolver fontResolver = renderer.getFontResolver();  
        // 解决中文支持问题，参数为字体的路径，html页面也必须引入字体  
        fontResolver.addFont(getResourceFontPath(), BaseFont.IDENTITY_H, BaseFont.NOT_EMBEDDED);  
        renderer.layout();  
        renderer.createPDF(out, false);  
        out.flush();  
        renderer.finishPDF();  
        if (StringUtils.isNotEmpty(watermark)) {  
            byte[] data = out.toByteArray();  
            out.close();  
            return watermark(data, watermark);  
        } else {  
            return out;  
        }  
    }  
  
    /**  
     * 增加水印，水印默认铺满全page  
     * 不使用onEndPage事件进行处理，此事件仅处理最后一页数据  
     * @param data pdf数据  
     * @param watermark 水印文字  
     * @return  
     * @throws IOException  
     * @throws DocumentException  
     */    private static ByteArrayOutputStream watermark(byte[] data, String watermark) throws IOException, DocumentException {  
  
        PdfReader reader = new PdfReader(data);  
  
        ByteArrayOutputStream outputStream = new ByteArrayOutputStream();  
        PdfStamper stamper = new PdfStamper(reader, outputStream, PdfWriter.VERSION_1_7);  
        PdfContentByte under;  
        // 字体  
        BaseFont font = BaseFont.createFont(getResourceFontPath() + ",1", BaseFont.IDENTITY_H, BaseFont.NOT_EMBEDDED);  
        // 原pdf文件的总页数  
        int pageSize = reader.getNumberOfPages();  
        PdfGState gs = new PdfGState();  
        // 设置填充字体不透明度为0.1f  
        gs.setFillOpacity(DEFAULT_WATER_MARK_OPACITY);  
  
        JLabel label = new JLabel();  
        label.setText(watermark);  
        FontMetrics metrics = label.getFontMetrics(label.getFont());  
        // 字符串的高, 只和字体有关  
        int textH = metrics.getHeight();  
        // 字符串的宽  
        int textW = metrics.stringWidth(label.getText());  
  
        for (int i = 1; i <= pageSize; i++) {  
            Rectangle rectangle = reader.getPageSize(i);  
            // 水印在之前文本下  
            under = stamper.getUnderContent(i);  
            under.beginText();  
            // 文字水印 颜色  
            under.setColorFill(Color.darkGray);  
            // 文字水印 字体及字号  
            under.setFontAndSize(font, DEFAULT_WATER_MARK_FONT_SIZE);  
            under.setGState(gs);  
            // 文字水印 起始位置 (0,0)            under.setTextMatrix(CommonUtil.ZERO, CommonUtil.ZERO);  
  
            if (StringUtils.isNotBlank(watermark)) {  
                // 高度间距，多少个文字高度  
                int heightDensity = CommonUtil.SIX;  
                // 宽度间距，多少个文字宽度  
                int widthDensity = CommonUtil.ONE;  
                for (int height = CommonUtil.TWO * textH; height < rectangle.getHeight(); height = height + textH * heightDensity) {  
                    for (int width = CommonUtil.TWO * textW; width < rectangle.getWidth(); width = width + textW * widthDensity) {  
                        // 显示文本  
                        under.showTextAligned(Element.ALIGN_CENTER, watermark, width - textW, height - textH, DEFAULT_WATER_MARK_ANGEL);  
  
                    }  
                }  
            }  
            under.endText();  
        }  
        stamper.close();  
        outputStream.flush();  
        return outputStream;  
    }  
  
    private static String getResourceFontPath() throws IOException {  
        final ClassPathResource regular = new ClassPathResource("fonts/simsun.ttc");  
        return regular.getURI() + "";  
    }  
  
  
    public static class B64ImgReplacedElementFactory implements ReplacedElementFactory {  
  
        /**  
         * 实现createReplacedElement 替换html中的Img标签  
         *  
         * @param c         上下文  
         * @param box       盒子  
         * @param uac       回调  
         * @param cssWidth  css宽  
         * @param cssHeight css高  
         * @return ReplacedElement  
         */        public ReplacedElement createReplacedElement(LayoutContext c, BlockBox box, UserAgentCallback uac,  
                                                     int cssWidth, int cssHeight) {  
            org.w3c.dom.Element e = box.getElement();  
            if (e == null) {  
                return null;  
            }  
            String nodeName = e.getNodeName();  
            // 找到img标签  
            if (nodeName.equals("img")) {  
                String attribute = e.getAttribute("src");  
                FSImage fsImage;  
                try {  
                    // 生成itext图像  
                    fsImage = buildImage(attribute, uac);  
                } catch (BadElementException | IOException e1) {  
                    fsImage = null;  
                }  
                if (fsImage != null) {  
                    // 对图像进行缩放  
                    if (cssWidth != -1 || cssHeight != -1) {  
                        fsImage.scale(cssWidth, cssHeight);  
                    }  
                    return new ITextImageElement(fsImage);  
                }  
            }  
  
            return null;  
        }  
  
        /**  
         * 将base64编码解码并生成itext图像  
         *  
         * @param srcAttr 属性  
         * @param uac     回调  
         * @return FSImage  
         * @throws IOException         io异常  
         * @throws BadElementException BadElementException  
         */        protected FSImage buildImage(String srcAttr, UserAgentCallback uac) throws IOException, BadElementException {  
            FSImage fsImage;  
            if (srcAttr.startsWith("data:image/")) {  
                String b64encoded = srcAttr.substring(srcAttr.indexOf("base64,") + "base64,".length());  
                // 解码  
                byte[] decodedBytes = Base64.decode(b64encoded);  
  
                fsImage = new ITextFSImage(com.lowagie.text.Image.getInstance(decodedBytes));  
            } else {  
                fsImage = uac.getImageResource(srcAttr).getImage();  
            }  
            return fsImage;  
        }  
  
  
        /**  
         * 实现reset  
         */        public void reset() {  
        }  
  
  
        @Override  
        public void remove(org.w3c.dom.Element arg0) {  
        }  
  
        @Override  
        public void setFormSubmissionListener(FormSubmissionListener arg0) {  
        }  
    }  
}
```

### itext7.0。不可商用

1. pom文件引入

```java
<!-- freemarker 读取html模板文件 -->  
<dependency>  
    <groupId>org.freemarker</groupId>  
    <artifactId>freemarker</artifactId>  
    <version>2.3.30</version>  
</dependency>  
<!-- xml 将html模板文件转换成pdf -->  
<dependency>  
    <groupId>com.itextpdf</groupId>  
    <artifactId>html2pdf</artifactId>  
    <version>3.0.3</version>  
</dependency>
```

1. 核心代码

```java  
import com.alibaba.metrics.StringUtils;  
import com.itextpdf.html2pdf.ConverterProperties;  
import com.itextpdf.html2pdf.HtmlConverter;  
import com.itextpdf.html2pdf.resolver.font.DefaultFontProvider;  
import com.itextpdf.io.font.FontProgram;  
import com.itextpdf.io.font.FontProgramFactory;  
import com.itextpdf.io.font.PdfEncodings;  
import com.itextpdf.kernel.colors.DeviceRgb;  
import com.itextpdf.kernel.events.Event;  
import com.itextpdf.kernel.events.IEventHandler;  
import com.itextpdf.kernel.events.PdfDocumentEvent;  
import com.itextpdf.kernel.font.PdfFont;  
import com.itextpdf.kernel.font.PdfFontFactory;  
import com.itextpdf.kernel.geom.Rectangle;  
import com.itextpdf.kernel.pdf.DocumentProperties;  
import com.itextpdf.kernel.pdf.PdfDocument;  
import com.itextpdf.kernel.pdf.PdfPage;  
import com.itextpdf.kernel.pdf.PdfWriter;  
import com.itextpdf.kernel.pdf.canvas.PdfCanvas;  
import com.itextpdf.kernel.pdf.extgstate.PdfExtGState;  
import com.itextpdf.layout.Canvas;  
import com.itextpdf.layout.element.Paragraph;  
import com.itextpdf.layout.font.FontProvider;  
import com.itextpdf.layout.property.TextAlignment;  
import com.itextpdf.layout.property.VerticalAlignment;  
import freemarker.template.Configuration;  
import freemarker.template.Template;  
import freemarker.template.TemplateException;  
import lombok.extern.slf4j.Slf4j;  
import org.springframework.core.io.ClassPathResource;  
  
import javax.servlet.http.HttpServletResponse;  
import javax.swing.*;  
import java.awt.*;  
import java.io.ByteArrayOutputStream;  
import java.io.IOException;  
import java.io.StringWriter;  
import java.net.URLEncoder;  
import java.util.Locale;  
  
/**  
 * @author  
 * @description  
 * @date 2022/3/3 18:37  
 * @since 1.0  
 */@Slf4j  
public class PdfTemplateUtil {  
  
    /***  
     * 通过模板导出pdf文件  
     * @param data  
     * @param templateFileName  
     * @param waterMarker 水印文字内容。  
     * @throws Exception  
     */    public static ByteArrayOutputStream createPDF(Object data, String templateFileName, String waterMarker) throws Exception {  
        // 生成html  
        String html = generateHtml(data, templateFileName);  
  
        // 渲染成pdf  
        return convertToPdf(html, waterMarker);  
    }  
  
    /**  
     * 创建pdf，并输出到response  
     *     * @param data  
     * @param templateFileName  
     * @param fileName  
     * @param watermark 水印  
     * @param response  
     */  
    public static void createPDFAndFlush(Object data, String templateFileName, String fileName, String watermark, HttpServletResponse response) {  
        try (ByteArrayOutputStream outputStream = createPDF(data, templateFileName, watermark)) {  
            response.setContentType("application/x-msdownload");  
            // 告诉浏览器，当前响应数据要求用户干预保存到文件中，以及文件名是什么 如果文件名有中文，必须URL编码  
            fileName = URLEncoder.encode(fileName, "UTF-8");  
            response.setHeader("Content-Disposition", "attachment;filename=" + fileName);  
            outputStream.writeTo(response.getOutputStream());  
        } catch (Exception e) {  
            log.error("exportDetailData2Pdf exception:", e);  
        }  
    }  
  
    /**  
     * 使用freemarker生成html  
     *     * @param data  
     * @param templateFileName  
     * @return  
     * @throws IOException  
     * @throws TemplateException  
     */    private static String generateHtml(Object data, String templateFileName) throws IOException, TemplateException {  
        try (StringWriter writer = new StringWriter()) {  
            // 创建一个FreeMarker实例, 负责管理FreeMarker模板的Configuration实例  
            Configuration configuration = new Configuration(Configuration.DEFAULT_INCOMPATIBLE_IMPROVEMENTS);  
            // 指定FreeMarker模板文件的位置  
            configuration.setClassForTemplateLoading(PdfTemplateUtil.class, "/templates");  
            // 设置模板的编码格式  
            configuration.setEncoding(Locale.CHINA, "UTF-8");  
            //设置兼容${xxx}方式取值为空的情况  
            configuration.setClassicCompatible(true);  
  
            // 获取模板文件  
            Template template = configuration.getTemplate(templateFileName, "UTF-8");  
            // 将数据输出到html中  
            template.process(data, writer);  
            writer.flush();  
            return writer.toString();  
        }  
    }  
  
    private static ByteArrayOutputStream convertToPdf(String html, String watermark) throws IOException {  
        ByteArrayOutputStream out = new ByteArrayOutputStream();  
  
        PdfDocument pdfDocument = new PdfDocument(new PdfWriter(out), (new DocumentProperties()));  
  
        // 添加水印，  
        if (StringUtils.isNotBlank(watermark)) {  
            pdfDocument.addEventHandler(PdfDocumentEvent.INSERT_PAGE, new WatermarkingEventHandler(watermark));  
        }  
  
        HtmlConverter.convertToPdf(html, pdfDocument, config());  
        return out;  
    }  
  
    private static ConverterProperties config() throws IOException {  
        // 设置字体  
        FontProgram fontProgram = FontProgramFactory.createFont(getResourceFontPath());  
  
        // 设置 css中 的字体样式（暂时仅支持宋体和黑体） 必须，不然中文不显示  
        FontProvider dfp = new DefaultFontProvider();  
        dfp.addFont(fontProgram);  
  
        ConverterProperties converterProperties = new ConverterProperties();  
        converterProperties.setFontProvider(dfp);  
        return converterProperties;  
    }  
  
    private static String getResourceFontPath() throws IOException {  
        final ClassPathResource regular = new ClassPathResource("fonts/simsun.ttc");  
        return regular.getURI() + ",1";  
    }  
  
  
    protected static class WatermarkingEventHandler implements IEventHandler {  
  
        /**  
         * 水印数据  
         */  
        private final String waterMark;  
  
        /**  
         * 水印旋转角度  
         */  
        private final static float DEFAULT_WATER_MARK_ANGEL = 0.45f;  
  
        /**  
         * 水印文字透明度  
         */  
        private final static float DEFAULT_OPACITY = 0.1f;  
        /**  
         * 水印字体大小  
         */  
        private final static int DEFAULT_WATER_MARK_FONT_SIZE = 12;  
  
        public WatermarkingEventHandler(String waterMark) {  
            this.waterMark = waterMark;  
        }  
  
        @Override  
        public void handleEvent(Event event) {  
            PdfDocumentEvent docEvent = (PdfDocumentEvent) event;  
            PdfDocument pdfDoc = docEvent.getDocument();  
            PdfPage page = docEvent.getPage();  
            PdfFont font = null;  
            try {  
                font = PdfFontFactory.createFont(getResourceFontPath(), PdfEncodings.IDENTITY_H, true);  
            } catch (Exception e) {  
                throw new RuntimeException(e);  
            }  
  
            JLabel label = new JLabel();  
            label.setText(waterMark);  
            FontMetrics metrics = label.getFontMetrics(label.getFont());  
            int textH = metrics.getHeight(); // 字符串的高, 只和字体有关  
            int textW = metrics.stringWidth(label.getText()); // 字符串的宽  
  
            PdfCanvas pdfCanvas = new PdfCanvas(page.newContentStreamBefore(), page.getResources(), pdfDoc);  
            // 保存配置，避免设置的透明度全局生效  
            pdfCanvas.saveState();  
  
            // 设置透明度  
            PdfExtGState gs = new PdfExtGState();  
            gs.setFillOpacity(DEFAULT_OPACITY);  
            pdfCanvas.setExtGState(gs);  
  
            try (Canvas canvas = new Canvas(pdfCanvas, page.getPageSize())) {  
                canvas.setFontSize(DEFAULT_WATER_MARK_FONT_SIZE)  
                        .setFont(font)  
                        .setFontColor(DeviceRgb.makeDarker(new DeviceRgb(192, 192, 192)));  
                Rectangle pageRect = page.getPageSizeWithRotation();  
  
                // 高度间距，多少个文字高度  
                int heightDensity = 6;  
                // 宽度间距，多少个文字宽度  
                int widthDensity = 1;  
                for (int height = 2 * textH; height < pageRect.getHeight(); height = height + textH * heightDensity) {  
                    for (int width = 2 * textW; width < pageRect.getWidth(); width = width + textW * widthDensity) {  
                        // 显示文本  
                        canvas.showTextAligned(new Paragraph(waterMark), width - textW, height - textH,  
                                pdfDoc.getPageNumber(page), TextAlignment.LEFT, VerticalAlignment.MIDDLE, DEFAULT_WATER_MARK_ANGEL);  
                    }  
                }  
            }  
  
            pdfCanvas.restoreState();  
        }  
    }  
}
```

### flying-saucer-pdf（基本和xhtmlrender 一样）

1. 引入pom

```
<!-- freemarker 读取html模板文件 -->  
<dependency>  
    <groupId>org.freemarker</groupId>  
    <artifactId>freemarker</artifactId>  
    <version>2.3.30</version>  
</dependency>  
<!-- xml 将html模板文件转换成pdf -->  
<dependency>  
    <groupId>org.xhtmlrenderer</groupId>  
    <artifactId>flying-saucer-pdf</artifactId>  
    <version>9.0.9</version>  
</dependency>
```

2. 代码

```
  
/**  
 * 水印旋转角度  
 */  
public static final Integer DEFAULT_WATER_MARK_ANGEL = 45;  
  
/**  
 * 水印文字透明度  
 */  
private final static float DEFAULT_WATER_MARK_OPACITY = 0.1f;  
/**  
 * 水印字体大小  
 */  
private final static int DEFAULT_WATER_MARK_FONT_SIZE = 12;  
  
/***  
 * 通过模板导出pdf文件  
 * @param data  
 * @param templateFileName  
 * @param waterMarker 水印文字内容。  
 * @throws Exception  
 */public static ByteArrayOutputStream createPDF(Object data, String templateFileName, String waterMarker) throws Exception {  
    // 生成html  
    String html = generateHtml(data, templateFileName);  
  
    // 渲染成pdf  
    return convertToPdf(html, waterMarker);  
}  
  
/**  
 * 创建pdf，并输出到response  
 * * @param data  
 * @param templateFileName  
 * @param fileName  
 * @param watermark 水印  
 * @param response  
 */  
public static void createPDFAndFlush(Object data, String templateFileName, String fileName, String watermark, HttpServletResponse response) {  
    try (ByteArrayOutputStream outputStream = createPDF(data, templateFileName, watermark)) {  
        response.setContentType("application/x-msdownload");  
        // 告诉浏览器，当前响应数据要求用户干预保存到文件中，以及文件名是什么 如果文件名有中文，必须URL编码  
        fileName = URLEncoder.encode(fileName, "UTF-8");  
        response.setHeader("Content-Disposition", "attachment;filename=" + fileName);  
        outputStream.writeTo(response.getOutputStream());  
    } catch (Exception e) {  
        log.error("exportDetailData2Pdf exception:", e);  
    }  
}  
  
/**  
 * 使用freemarker生成html  
 * * @param data 模板数据  
 * @param templateFileName 模板名称  
 * @return html  
 * @throws IOException  
 * @throws TemplateException  
 */private static String generateHtml(Object data, String templateFileName) throws IOException, TemplateException {  
    try (StringWriter writer = new StringWriter()) {  
        // 创建一个FreeMarker实例, 负责管理FreeMarker模板的Configuration实例  
        Configuration configuration = new Configuration(Configuration.DEFAULT_INCOMPATIBLE_IMPROVEMENTS);  
        // 指定FreeMarker模板文件的位置  
        configuration.setClassForTemplateLoading(PdfTemplateUtil.class, "/templates");  
        // 设置模板的编码格式  
        configuration.setEncoding(Locale.CHINA, "UTF-8");  
        //设置兼容${xxx}方式取值为空的情况  
        configuration.setClassicCompatible(true);  
  
        // 获取模板文件  
        Template template = configuration.getTemplate(templateFileName, "UTF-8");  
        // 将数据输出到html中  
        template.process(data, writer);  
        writer.flush();  
        return writer.toString();  
    }  
}  
  
public static ByteArrayOutputStream convertToPdf(String htmlStr, String watermark) throws Exception {  
    if (StringUtils.isBlank(htmlStr)) {  
        return null;  
    }  
    // itext开源版本实现  
    ITextRenderer renderer = new ITextRenderer();  
    ByteArrayOutputStream out = new ByteArrayOutputStream();  
    renderer.setPDFVersion(PdfWriter.VERSION_1_7);  
    renderer.setDocumentFromString(htmlStr);  
    renderer.getSharedContext().getTextRenderer().setSmoothingThreshold(1);  
    ITextFontResolver fontResolver = renderer.getFontResolver();  
    // 解决中文支持问题，参数为字体的路径，html页面也必须引入字体  
    fontResolver.addFont(getResourceFontPath(), BaseFont.IDENTITY_H, BaseFont.NOT_EMBEDDED);  
    renderer.layout();  
    renderer.createPDF(out, false);  
    out.flush();  
    renderer.finishPDF();  
    if (StringUtils.isNotEmpty(watermark)) {  
        byte[] data = out.toByteArray();  
        out.close();  
        return watermark(data, watermark);  
    } else {  
        return out;  
    }  
}  
  
/**  
 * 增加水印，水印默认铺满全page  
 * 不使用onEndPage事件进行处理，此事件仅处理最后一页数据  
 * @param data pdf数据  
 * @param watermark 水印文字  
 * @return 返回水印  
 * @throws IOException  
 * @throws DocumentException  
 */private static ByteArrayOutputStream watermark(byte[] data, String watermark) throws IOException, DocumentException {  
  
    PdfReader reader = new PdfReader(data);  
  
    ByteArrayOutputStream outputStream = new ByteArrayOutputStream();  
    PdfStamper stamper = new PdfStamper(reader, outputStream, PdfWriter.VERSION_1_7);  
    PdfContentByte under;  
    // 字体  
    BaseFont font = BaseFont.createFont(getResourceFontPath() + ",1", BaseFont.IDENTITY_H, BaseFont.NOT_EMBEDDED);  
    // 原pdf文件的总页数  
    int pageSize = reader.getNumberOfPages();  
    PdfGState gs = new PdfGState();  
    // 设置填充字体不透明度为0.1f  
    gs.setFillOpacity(DEFAULT_WATER_MARK_OPACITY);  
  
    JLabel label = new JLabel();  
    label.setText(watermark);  
    FontMetrics metrics = label.getFontMetrics(label.getFont());  
    // 字符串的高, 只和字体有关  
    int textH = metrics.getHeight();  
    // 字符串的宽  
    int textW = metrics.stringWidth(label.getText());  
  
    for (int i = 1; i <= pageSize; i++) {  
        Rectangle rectangle = reader.getPageSize(i);  
        // 水印在之前文本下  
        under = stamper.getUnderContent(i);  
        under.beginText();  
        // 文字水印 颜色  
        under.setColorFill(Color.darkGray);  
        // 文字水印 字体及字号  
        under.setFontAndSize(font, DEFAULT_WATER_MARK_FONT_SIZE);  
        under.setGState(gs);  
        // 文字水印 起始位置 (0,0)        under.setTextMatrix(CommonUtil.ZERO, CommonUtil.ZERO);  
  
        if (StringUtils.isNotBlank(watermark)) {  
            // 高度间距，多少个文字高度  
            int heightDensity = CommonUtil.SIX;  
            // 宽度间距，多少个文字宽度  
            int widthDensity = CommonUtil.ONE;  
            for (int height = CommonUtil.TWO * textH; height < rectangle.getHeight(); height = height + textH * heightDensity) {  
                for (int width = CommonUtil.TWO * textW; width < rectangle.getWidth(); width = width + textW * widthDensity) {  
                    // 显示文本  
                    under.showTextAligned(Element.ALIGN_CENTER, watermark, width - textW, height - textH, DEFAULT_WATER_MARK_ANGEL);  
  
                }  
            }  
        }  
        under.endText();  
    }  
    stamper.close();  
    outputStream.flush();  
    return outputStream;  
}  
  
private static String getResourceFontPath() throws IOException {  
    final ClassPathResource regular = new ClassPathResource("fonts/simsun.ttc");  
    return regular.getURI() + "";  
}
```

### 模板示例

```html
<!DOCTYPE html>  
<html lang="en">  
<head>  
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8"/>  
    <style type="text/css">  
        /*解决html转pdf文件中文不显示的问题*/  
        body {  
            font-family: SimSun;  
        }  
  
        /*设定纸张大小*/  
        @page {  
            size: A3  
        }  
  
        table {  
            width: 100%;  
            margin-bottom: 10px;  
        }  
  
        th.th-cell {  
            width: 25%;  
            text-align: right;  
        }  
  
        .th-cell-label {  
            float: left;  
            width: 160px;  
            font-weight: 400;  
            text-align: right;  
            margin-bottom: 10px;  
        }  
  
        .th-cell-tip {  
            float: left;  
            width: 75%;  
            font-weight: 400;  
            text-align: left;  
        }  
  
        .th-cell-span {  
            float: left;  
            width: 170px;  
            font-weight: 400;  
            text-align: left;  
            margin-bottom: 10px;  
        }  
  
        .th-cell-text {  
            float: left;  
            text-align: left;  
            border: 1px solid black;  
            margin-bottom: 10px;  
            width: auto;  
            min-width: 800px;  
            max-width: 800px;  
            height: auto;  
            min-height: 50px;  
            font-weight: normal;  
        }  
  
        .comment-tr {  
            height: auto;  
        }  
  
        .th-cell-div {  
            text-align: left;  
            margin-bottom: 10px;  
        }  
  
        .block {  
            float: none;  
            height: auto;  
        }  
  
        .watermark {  
            position: absolute;  
            z-index: 9627;  
            top: 0px;  
            left: 700px;  
            height:0px;  
            overflow:visible;  
            pointer-events:none;  
            background:none !important;  
        }  
    </style>  
</head>  
<body>  
<#if status = 4>  
    <div id="waterMark" class="watermark">  
        <img width="300" height="300" src="data:image/png;base64,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cbEQSiQLv/rxBQVnB7w0itjWXieAAkKvjxC2zxcEYKYhnPqPiBZXw7xmRkw608CKltxpe36vweRF6n+AOp3iCKcILF8qRkWFYfqUNx89p6Iw4LXRxPZ6jQAKCF4bMWyvrwg88MvbxOVLs/KSqsicaVIDASJTwwJZJ5bsBSuAdgwGzAYI1IzTJKwmSqymmLwicFJJLA4q6EiqHYSxDCTQGgEUEFrjhrmQgOsEwIeA8tGfZKIUZIUzJMkgInUeJNCgQyKYEzIsIxBC9wnd69E/ePmD6SX4ceB1VjdEVpMIr2kxQYszEe02Zjd1BjVl9GjzsVlIINAEevbBEuhRwc4jAQcEQJDIX35fKC1lqEdIXYgzppArEEFkLY46JGKFEsNRh0ScwK5sRZ5hDbrPUxdMJsNwhIoaosmwOk9f5tSbAku3UD29aOk8zQjHKj2G6/Rfp+cl6vWAxpswCPW1QD1AmqYIx4JhhAVzOCOb/TduNEYu3Wei90EHA4lJkQASQAJIAAkgASSABJAAEkACSAAJIAEkgASQABJAAkgACSABJIAEkAASQAJIAAkgASSABJAAEkACSAAJIAEkgASQABJAAkgACSABJIAEkAASQAJIAAkgASSABJAAEkACSAAJIAEkgASQABJAAkgACSABJIAEkAASQAJIAAkgASSABJAAEkACSAAJIAEkgASQABJAAkgACSABJIAEkAASQAJIAAkgASSABJAAEkACSAAJIAEkgASQABJAAkgACSABJIAEkAASQAJIAAkgASSABJAAEkACSAAJIAEkgASQABJAAkgACSABJIAEkAASQAJIAAkgASSABJAAEkACSAAJIAEkgASQABJAAkgACSABJIAEkAASQAJIAAkgASSABJAAEkACSAAJIAEkgASQABJAAkgACSABJIAEkAASQAI2CPx/+wUEot6YvdQAAAAASUVORK5CYII="/>  
    </div></#if>  
<span>资质认证详情</span><br/>  
<br/>  
<span>运营系统 &gt; 考勤管理 &gt; 基础配置 &gt; 资质认证 &gt;资质认证详情</span><br/>  
<br/>  
<!--基本信息-->  
<div>  
    <span style="border-bottom: 1px solid black">基本信息</span>  
</div>  
<br/>  
<div>  
    <div class="block">  
        <table>            <tr>                <th class="th-cell">  
                    <label class="th-cell-label">资质认证单号：</label>  
                    <span class="th-cell-span">${qualityCertNo}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">认证状态：</label>  
                    <span class="th-cell-span">${statusStr}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">新建人：</label>  
                    <span class="th-cell-span">${createEmpName}</span>  
                </th>            </tr>            <tr>                <th class="th-cell">  
                    <label class="th-cell-label">新建人岗位：</label>  
                    <span class="th-cell-span">${createEmpPostName}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">资质认证类型：</label>  
                    <span class="th-cell-span">${qualityCertTemplateName}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">认证员工工号：</label>  
                    <span class="th-cell-span">${certEmpNo}</span>  
                </th>            </tr>            <tr>                <th class="th-cell">  
                    <label class="th-cell-label">认证员工姓名：</label>  
                    <span class="th-cell-span">${certEmpName}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">岗位名称：</label>  
                    <span class="th-cell-span">${certEmpPostName}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">所属部门：</label>  
                    <span class="th-cell-span">${certEmpDeptName}</span>  
                </th>            </tr>            <tr>                <th class="th-cell">  
                    <label class="th-cell-label">运营模式：</label>  
                    <span class="th-cell-span">${cooperationName}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">所属城市：</label>  
                    <span class="th-cell-span">${cityName}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">新建时间：</label>  
                    <span class="th-cell-span">${createTimeStr}</span>  
                </th>            </tr>            <tr>                <th class="th-cell">  
                    <label class="th-cell-label">通过认证时间：</label>  
                    <span class="th-cell-span">${passTimeStr}</span>  
                </th>            </tr>            <tr class="comment-tr">  
                <th rowspan="1" colspan="3" class="th-cell">  
                    <label class="th-cell-label">备注：</label>  
                    <span class="th-cell-text">${remark}</span>  
                </th>            </tr>        </table>    </div>    <!--分隔线-->  
    <HR align="center" width="100%" color="black" size="1px"/>  
    <!--店长评估信息-->  
    <div>  
        <span style="border-bottom: 1px solid black">认证评估信息</span>  
    </div>    <br/>    <div class="block">  
        <table>            <tr>                <th class="th-cell">  
                    <label class="th-cell-label">认证人姓名：</label>  
                    <span class="th-cell-span">${shopownerEmpName}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">认证人最终评价结果：</label>  
                    <span class="th-cell-span">${shopownerEvaluateStatusStr}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">认证人评价时间：</label>  
                    <span class="th-cell-span">${shopownerEvaluateDateStr}</span>  
                </th>            </tr>            <tr class="comment-tr">  
                <th rowspan="1" colspan="3" class="th-cell">  
                    <label class="th-cell-label">认证人评语：</label>  
                    <span class="th-cell-text">${shopownerComment.comment}</span>  
                </th>            </tr>        </table>    </div>    <br/>    <div>        <span style="border-bottom: 1px solid black">认证访谈信息</span>  
    </div>    <br/>    <div class="block">  
        <table>            <tr>                <th class="th-cell">  
                    <label class="th-cell-label">访谈人姓名：</label>  
                    <span class="th-cell-span">${areaManagerEmpName}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">访谈人评价结果：</label>  
                    <span class="th-cell-span">${areaManagerEvaluateStatusStr}</span>  
                </th>                <th class="th-cell">  
                    <label class="th-cell-label">访谈人评价日期：</label>  
                    <span class="th-cell-span">${areaManagerEvaluateDateStr}</span>  
                </th>            </tr>            <#list areaManagerComments as areaManagerComment>  
                <tr class="comment-tr">  
                    <th rowspan="1" colspan="3" class="th-cell">  
                        <label class="th-cell-label"><#if areaManagerComment_index = 0>访谈人评语：<#else>&#160;</#if></label>  
                        <label class="th-cell-tip">${areaManagerComment_index + 1}.${areaManagerComment.commentTitle}</label>  
                    </th>                </tr>                <tr class="comment-tr">  
                    <th rowspan="1" colspan="3" class="th-cell">  
                        <label class="th-cell-label">&#160;</label>  
                        <span class="th-cell-text">  
                                 ${areaManagerComment.comment}  
                        </span>  
                    </th>                </tr>            </#list>  
        </table>  
    </div>    <br/>    <div>        <span style="border-bottom: 1px solid black">资质认证明细</span>  
    </div>    <br/>    <div class="block">  
        <table border="1" cellspacing="0">  
            <tr>                <td width="10%">任务序号</td>  
                <td width="10%">任务项</td>  
                <td width="10%">任务内容序号</td>  
                <td width="30%">任务内容</td>  
                <td width="10%">标签</td>  
                <td width="10%">是否合格</td>  
                <td width="10%">操作人</td>  
                <td width="10%">操作时间</td>  
            </tr>            <#list qualityCertDetailList as detail>  
                <tr>  
                    <td>${detail.taskNumber}</td>  
                    <td>${detail.taskName}</td>  
                    <td>${detail.taskContentNumber}</td>  
                    <td>${detail.taskContent}</td>  
                    <td>${detail.label}</td>  
                    <td>${detail.qualifiedStatusStr}</td>  
                    <td>${detail.createEmpName}</td>  
                    <td>${detail.createTimeStr}</td>  
                </tr>            </#list>  
        </table>  
  
    </div></div>  
</body>  
</html>
```

## 使用踩坑

1. itext7.0之前的onEndPage 方法，仅仅在最后一页才执行，而不是每页执行。
2. 分页大小，横竖屏转换。在css中加 [@page](https://docs.w3cub.com/css/@page/size)
3. 中文不显示问题，需要引入字体。

## 参考资料

[java实现HTML转PDF -腾讯云开发者社区-腾讯云](https://cloud.tencent.com/developer/article/1534712)  
[xhtmlrenderer +itext2.0.8 将html转换成pdf，完美css，带图片，自动分页 - 台部落](https://www.twblogs.net/a/5cc14797bd9eee397113e839/?lang=zh-cn)
