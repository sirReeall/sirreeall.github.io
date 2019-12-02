---
title:  "Creating an Alfresco custom transformer for html2pdf transformation"
categories: 
    - blog
    - alfresco
    - howto
    - customisations
tags: 
    - alfresco
    - acs
    - transformers
    - customisations
---

While working with Alfresco Content Services there always comes a time where the out of the box solution just doesn't cut it. As is the way with Open Source, products are built using other open source products. This is the case with Alfresco and how it transforms content from one mime type to another.

In this post, I'll be looking at an issue with the out of the box jodconverter used to transform html files to pdf files. This process is actually critical for Share previews (since 4.2 I think). For the most part jodconverter does a good job of transforming documents to pdf. 

However, if you are using `<ul type="desc">` something odd happens. 

Here is what this element looks like in Google Chrome:

![Working](/assets/files/2019-11-28-custom-trans-html2pdf/test-chrome.png)

And here is what this looks like in Alfresco Share's OOTB preview:

![Not Working](/assets/files/2019-11-28-custom-trans-html2pdf/test-share.png)

These squiggly lines are probably caused by a few factors:

1. There is no direct html to pdf transform. In fact, Alfresco defines a pipeline which actually is html -> odt - pdf.
2. The intermediate html -> odt does not render the html element correctly
    You can find this out by enabling debugging on the TransformationDebug class, finding the intermediate node, and downloading this.

## The Solution

Let's create a transformer using a 3rd party tool. In this instance we will use [wkhtmltopdf](https://wkhtmltopdf.org/).

There are a few parts to this:

1. Install and test [wkhtmltopdf](https://wkhtmltopdf.org/)
2. Configure a new transformer in Alfresco
3. Set the new transformer to transform html files to pdf

Let's get started!

### Install and test [wkhtmltopdf](https://wkhtmltopdf.org/)

In my lab I have ACS 5.2.6 on Centos 7. wkhtmltopdf only supports a [limited number of OS](https://wkhtmltopdf.org/downloads.html), so you may need to find another transformer if you are using a different os. Either way, the principles out lined here are the same.

Run the install, I used rpm:

```
wget https://downloads.wkhtmltopdf.org\
/0.12/0.12.5/wkhtmltox-0.12.5-1.centos7.x86_64.rpm
sudo rpm -i wkhtmltox-0.12.5-1.centos7.x86_64.rpm
```

I found there are some missing dependencies as wkhtmltox requires org-x11-fonts-75dpi. That ok, we can install these:

```
sudo yum install xorg-x11-fonts-75dpi
```

We can then attempt the installation again:

```
sudo rpm -i wkhtmltox-0.12.5-1.centos7.x86_64.rpm
```

The wkhtmltox documentation has a pretty simple test to verify the install. 
The following command takes Google's landing page and turns it into a pdf file saved in the current directory:

```
 wkhtmltopdf http://google.com google.pdf
 ```

Open this pdf file up and you'll see something like the following

![Google PDF](/assets/files/2019-11-28-custom-trans-html2pdf/google-pdf.png)

Awesome! 

Let's test with the problem html tag to verify this transformer fits the bill. To do this, we create a test file with the following content:

```html
<html>
  <body>
    <ul type="disc">
      <li>888</li>
    </ul>
  </body>
</html>
```

Run this through wkhtmltox:

```
wkhtmltopdf test.html test.pdf
```

And vola, we have a correctly rendered unsorted list of type disc.

### Configure a new transformer in Alfresco

There are a lot on online resources documenting this procedure. The [Alfresco Documentation](https://docs.alfresco.com/5.2/references/dev-extension-points-content-transformer.html) provides the background information on how to set up custom transformers. It is definitely worth a read as I don't go into too much detail below.

First, lets extend Alfresco with a new bean for our transformer:

1. Create a file called `custom-html2pdf-transformer-context.xml` in the Alfresco extension directory to represent this bean:

```
touch /opt/acs526\
/tomcat/shared/classes/alfresco/extension\
/custom-html2pdf-transformer-context.xml
```

2. Add the following bean definition (there are actually two beans defined):

```xml
<?xml version='1.0' encoding='UTF-8'?>
<!DOCTYPE beans PUBLIC '-//SPRING//DTD BEAN//EN' 'http://www.springframework.org/dtd/spring-beans.dtd'>

<beans>
	<bean id="transformer.worker.html2pdf" class="org.alfresco.repo.content.transform.RuntimeExecutableContentTransformerWorker">
		<property name="mimetypeService">
			<ref bean="mimetypeService" />
		</property>
		<property name="checkCommand">
			<bean class="org.alfresco.util.exec.RuntimeExec">
				<property name="commandsAndArguments">
					<map>
						<entry key="Linux">
							<list>
								<value>sh</value>
								<value>-c</value>
								<value>/usr/local/bin/wkhtmltopdf -V</value>
							</list>
						</entry>
					</map>
				</property>
				<property name="errorCodes">
					<value>1</value>
				</property>
			</bean>
		</property>
		<property name="transformCommand">
			<bean class="org.alfresco.util.exec.RuntimeExec">
				<property name="commandsAndArguments">
					<map>
						<entry key="Linux">
							<list>
								<value>sh</value>
								<value>-c</value>
								<value>/usr/local/bin/wkhtmltopdf ${source} ${target}</value>
							</list>
						</entry>
					</map>
				</property>
				<property name="errorCodes">
					<value>1</value>
				</property>
			</bean>
		</property>
    </bean>
    
	<bean id="transformer.html2pdf" class="org.alfresco.repo.content.transform.ProxyContentTransformer" parent="baseContentTransformer">
		<property name="worker">
			<ref bean="transformer.worker.html2pdf" />
		</property>
	</bean>
</beans>
```

The above is specific to a Linux as you probably guess from the element `<entry key="Linux">`

The above beans will be loaded when Alfresco bootstraps. They contain all the information required to allow Alfresco to run wkhtml2pdf. If you are interested on what the above actually means, I recommended reading the [documentation](https://docs.alfresco.com/5.2/references/dev-extension-points-content-transformer.html). Here's two quote that sums up the important parts:

>The transformer worker does the actual job of executing the transformation. It has two important properties that need to be set, the checkCommand property, which is used to verify that the command line tool/script that is to be used for the transformation is actually available. The other property is called transformCommand and should contain the script/tool path plus the source and target variables, which will resolve to temporary files that will be used during the transformation.

>The transformer bean needs to specify baseContentTransformer as the parent, as it handles registering this new transformer with the Alfresco Content Services system. The transformer implementation class that you use in this case is called ProxyContentTransformer and it is delegating the actual transformation to the worker.

3. Set the new transformer to transform html files to pdf

This is set in alfresco-global.properties via the following settings:

```
content.transformer.html2pdf.priority=30
content.transformer.html2pdf.extensions.html.pdf.supported=true
content.transformer.html2pdf.extensions.html.pdf.priority=30
```

Again this is [documented](https://docs.alfresco.com/5.2/references/dev-extension-points-content-transformer.html) and the important quote is:

>The properties have names that follow a certain convention:
>content.<transformer bean id>.priority=<number> (Default priority for this transformer)
>content.<transformer bean id>.extensions.<source mimetype>.<target mimetype>.priority=<number>  (Priority for this transformation) 
>content.<transformer bean id>.extensions.<source mimetype>.<target mimetype>.supported=[true|false]

And that's it.

### Testing

Restart ACS, open up Share, upload a document, and away you go.

![Google PDF](/assets/files/2019-11-28-custom-trans-html2pdf/test-share-new-transformer.png)

There is an interesting side effect caused by adding this transformer. An error is logged for thumbnails, I'm not concerned with thumbnails and decided to ignore these errors (plus you need the below debug settings to be able to "see" the errors in the logs)

### Debugging | Troubleshooting

As with anything, there are times where things go wrong. There are some [Community Blogs](https://hub.alfresco.com/t5/alfresco-content-services-hub/content-transformation-debug/ba-p/289284) on this, and the following debug setting will provide some much needed information on what is going on:

```
log4j.logger.org.alfresco.repo.content.transform.TransformerDebug=DEBUG
```

As we are using a shell exectution in the custom transformer you would also want the following:

```
log4j.logger.org.alfresco.util.exec.RuntimeExec=DEBUG
```

or if you really want everything, add

```
log4j.logger.org.alfresco.repo.content.transform=trace
```

### Acknowledgements

The following link contains a similar how to blog that I adapted for my version of Alfresco:

http://imben.me/html-to-pdf-transformation-with-wkhtmltopdf/