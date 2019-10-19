---
title: 使用XSLT 格式化 XML
excerpt: 在平常工作中经常遇到要求对 XML 进行格式化的情况， XSLT相比较于写纯java代码操作 org.w3c.dom.Document 更加方便，利于维护。下面会用一个java例子来介绍XSLT的基本用法。
category: Java
---

> 在平常工作中经常遇到要求对 XML 进行格式化的情况， XSLT相比较于写纯java代码操作 org.w3c.dom.Document 更加方便，利于维护。
下面会用一个java例子来介绍XSLT的基本用法。

## 场景
原始的XML 
```xml
<?xml version="1.0" encoding="UTF-8"?>
<?xml version="1.0" encoding="UTF-8"?>
<school>
	<students>
		<student>
			<name>Lucy</name>
			<gender>Female</gender>
		</student>
		<student>
			<name>Lily</name>
			<gender>Female</gender>
		</student>
		<student>
			<name>Tom</name>
			<gender>Male</gender>
		</student>
	</students>
</school>
```
想要把所有的女性选出来 放入下面的xml结构中
```xml
<?xml version="1.0" encoding="UTF-8"?>
<cityMall>
	<store>
	    <name>Lucy</name>
	    <name>Lily</name>
	</store>
</cityMall>
```
怎么用XSLT 实现这种转化呢。
如下 XSLT - formater.xml
```xml
<xsl:stylesheet version="1.0"
	xmlns:xsl="http://www.w3.org/1999/XSL/Transform">
	<xsl:output method="xml" omit-xml-declaration="yes"
		encoding="UTF-8" indent="yes" />
	<!-- GLOBAL VARIABLES -->
	<xsl:variable name="target" select="/school/students" />

	<!-- transform template -->
	<xsl:template match="/">
		<cityMall>
			<store>
				<xsl:for-each select="$target/student">
					<xsl:if test="(boolean(gender ='Female'))">
						<xsl:copy-of select="name"></xsl:copy-of>
					</xsl:if>
				</xsl:for-each>
			</store>
		</cityMall>
	</xsl:template>
</xsl:stylesheet>
```
遍历所有的 student element， 发现geneder为Female时，选择它。

怎么使用JAVA 调用它呢。


Formater.java
```java
import java.io.File;
import java.io.IOException;
import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import javax.xml.parsers.ParserConfigurationException;
import javax.xml.transform.Transformer;
import javax.xml.transform.TransformerConfigurationException;
import javax.xml.transform.TransformerException;
import javax.xml.transform.TransformerFactory;
import javax.xml.transform.dom.DOMResult;
import javax.xml.transform.dom.DOMSource;
import javax.xml.transform.stream.StreamResult;
import org.apache.commons.io.output.ByteArrayOutputStream;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.w3c.dom.Document;
import org.xml.sax.SAXException;
import com.ericsson.dve.jdv.m2m.core.util.M2MTransformerFactory;

public class Formater {
	private static final Logger LOGGER = LoggerFactory.getLogger(M2MTransformerFactory.class);

	public static void main(String[] args) {
		Formater formater = new Formater();

		try {
			Document document = formater.getSrcDocument("test/META-INF/source.xml");

			Document transformedResponse = formater.transformDoc(document);

			System.out.println(formater.xml2Str(transformedResponse));

		} catch (Exception e) {
			LOGGER.error("Exception", e);
		}

	}

	private Document getSrcDocument(String srcFilePath) throws SAXException, IOException, ParserConfigurationException {
		Document document = null;
		File srcDocFile = new File("test/META-INF/source.xml");
		DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
		DocumentBuilder builder;
		builder = factory.newDocumentBuilder();
		document = builder.parse(srcDocFile);
		return document;
	}

	private Document transformDoc(Document srcDoc) throws TransformerConfigurationException, TransformerException {
		DOMResult tResult = new DOMResult();

		M2MTransformerFactory.getTransformer("/META-INF/xslt/NewFile2.xml").transform(new DOMSource(srcDoc), tResult);

		Document transformedResponse = (Document) tResult.getNode();
		return transformedResponse;
	}

	private String xml2Str(Document document) throws TransformerException {
		TransformerFactory factory = TransformerFactory.newInstance();
		Transformer transformer = factory.newTransformer();

		ByteArrayOutputStream byteArrayOutputStream = new ByteArrayOutputStream();

		transformer.transform(new DOMSource(document), new StreamResult(byteArrayOutputStream));

		return byteArrayOutputStream.toString();
	}
}
```


TransformerFactory.java
```java
import java.io.InputStream;
import javax.xml.transform.Source;
import javax.xml.transform.Transformer;
import javax.xml.transform.TransformerConfigurationException;
import javax.xml.transform.TransformerException;
import javax.xml.transform.URIResolver;
import javax.xml.transform.stream.StreamSource;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
public class TransformerFactory {
	private static final Logger LOGGER = LoggerFactory.getLogger(TransformerFactory.class);

	private static javax.xml.transform.TransformerFactory factory;
	private static Transformer transformer = null;

	static {
		factory = javax.xml.transform.TransformerFactory.newInstance();
		factory.setURIResolver(new XsltURIResolver());
	}

	private TransformerFactory() {

	}

	/**
	 * Creates Transformer from the file passed as argument.
	 *
	 */
	public static Transformer getTransformer(String xsltFileName) throws TransformerConfigurationException {

		if (xsltFileName == null || xsltFileName.isEmpty()) {
			throw new IllegalArgumentException("xslFileName argument must not be null");
		}

		try {
			if (transformer == null) {
				return factory.newTransformer(
						new StreamSource(TransformerFactory.class.getResourceAsStream(xsltFileName)));
			}

			return transformer;
		} catch (TransformerConfigurationException e) {
			LOGGER.error("Error creating xsl Transformer for file: {}", xsltFileName);
			throw e;
		}

	}

	/**
	 * Transformer factory URI resolver
	 *
	 */
	static class XsltURIResolver implements URIResolver {

		public Source resolve(String href, String base) throws TransformerException {

			InputStream is = TransformerFactory.class.getResourceAsStream("/META-INF/xslt/" + href);
			if (is == null) {
				throw new TransformerException("Transformation file " + href + " not found");
			}
			return new StreamSource(is);
		}
	}
}
```
**Note: formater.xml 应该放在Transformer.java 的相对路径 /META-INF/xslt/ 下面**

之后如果需要不同格式的document，都可以直接修改formater.xml 来完成。 不仅直观，后续维护也更加方便。

##XSLT 其他用法
-  chose, when ,otherwise. 进行逻辑判断。
```xml
<xsl:choose>
	<xsl:when test="ResultData">
		<xsl:copy-of select="ResultData/*"></xsl:copy-of>
	</xsl:when>
	<xsl:otherwise>
		<xsl:copy-of select="*"></xsl:copy-of>
	</xsl:otherwise>
</xsl:choose>
```
-  xls:if xsl:element 动态的生成xml的element. 
```xml
<xsl:if test="not(boolean(/school/student/name =''))">
	<xsl:element name="new_name">
		<xsl:value-of select="/school/student/name"></xsl:value-of>
	</xsl:element>
</xsl:if>
```

优点：XSLT 还有很多其他用法方便xml的处理。特别适合于业务逻辑复杂的转换，
劣势：t比单纯的java 处理 org.w3c.dom.Document 处理速度慢。 但我并没有进行大量实验就行分析对比。只是在使用过程中发现 performance略有不足。