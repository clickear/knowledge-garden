---
title: html转pdf
date created: 2023-07-17
date modified: 2023-07-17
---

通过freemark将模板转换成html，后将html转pdf，方便扩展。支持水印

## 方案选择

itext,早期是可以商业的，后面itext5.0,7.0之后不可商用。itext不可商业后，基于itext4.x版本的代码，有人fork了分支。openpdf

html转pdf方案参考

flying-saucer-pdf: 基于Itext支持html，但是样式只支持css2.0，不建议使用  
xhtmlrenderer: 基于itext的开源实现，样式支持比较丰富。  
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
