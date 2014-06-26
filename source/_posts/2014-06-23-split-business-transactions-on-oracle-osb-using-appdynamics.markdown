---
layout: post
title: "Split Business Transactions on Oracle Service Bus using AppDynamics"
date: 2014-06-23 00:00:00 +1000
comments: true
categories: [Oracle OSB, AppDynamics, APM, Java, Byte Code Injection] 
---

### Overview

[We][ecetera] recently helped a customer configure AppDynamics to monitor their business transactions on [Oracle OSB][oracleosb]. AppDynamics does not have built-in support for Oracle OSB, although it does support Weblogic Application Server. It detected various business transactions out of the box, but one type of transaction in particular proved to be a little tricky. 

The OSB was accepting SOAP messages from a proprietary upstream system all on one endpoint. It then inspected the message and called one or more services on the OSB, essentially routing the incoming messages. AppDynamics grouped all these messages as one business transaction because they all arrived at the same endpoint. This was not acceptable as a significant number of distinct business transactions were processed this way. We had to find a way to separate the business transaction using the input data.

Changing the application was not an option so, we solved this by augmenting the _application server_ code to give AppDynamics an efficient way to determine the business transaction name. The rest of this article describes how AppDynamics was used to find a solution, and how we improved the solution using custom byte code injection. 


### Example Application
An example OSB application which reproduces the design of the actual application is used to illustrate the problem and solution.  

{% img center /images/split_bt_on_OSB_AppD/example_osb_proxy_service.png Example Application %}

### Finding the business transactions
It is a bit tricky to find the business transactions for this application because the services on Oracle OSB are all implemented by configuring the same Oracle classes. Each Proxy Service is just another instance of the same class and the call graph in the transaction snapshot is full of generic sounding names like _'AssignRuntimeStep'_

{% img center /images/split_bt_on_OSB_AppD/callgraph.png Proxy Service call graph %}

The first step to figuring out how to separate the business transactions is using the AppDynamics [Method Invocation Data Collector][midc] feature. This gives you a way to inspect the method parameters in the call and printing their values. Method invocation data collectors allows you to configure AppDynamics to capture the value of a parameter for a particular method invocation. Not only the value of the parameter but it is possible to apply a chain of get- methods to the parameter.

The following figure shows data collector configuration to get information out of the parameters passed to the _processMessage_ method on the _AssignRuntimeStep_ class we noticed in the call graph. This data collector tells AppDynamics to capture the first parameter to the _processMessage_ method on the class _AssignRuntimeStep_ and then to collect the result of calling _toString()_ and _getClass().getName()_ on that parameter. 

{% img center /images/split_bt_on_OSB_AppD/method_invocation_data_collector.png Method Invocation Data Collector %}

The results of this can be seen in the following images. The first shows the result of the _toString()_ applied to the first parameter

{% img center /images/split_bt_on_OSB_AppD/user_data1.png %}

and the second shows the class of the parameter. Notice that the class name is repeated three times. It is a list of values, one value saved for every invocation of the _processMessage_ method.

{% img center /images/split_bt_on_OSB_AppD/user_data2.png %}

From the first image it is obvious that the input message is contained in the first parameter. You can also see that the messages is stored in a map like structure and the key is called _body_. Note that the business transaction name is visible in the first image **TranactionName="Business Transaction1"**. The second image shows the type of the first parameter so the message is contained in an object of class _MessageContextImpl_.    

The next step is to tell AppDynamics what to use for splitting the business transactions and this can be done by using a [Business Transaction Match Rule][matchconditions]. The number of characters from the start of the message to the field we are interested in are roughly __126__ and assuming the transaction names will be around 20 characters we can set up a match rule as follows:  

{% img center /images/split_bt_on_OSB_AppD/new_bt_match_rule_1.png New Business Transaction Match Rule %}
{% img center /images/split_bt_on_OSB_AppD/new_bt_match_rule_split_1.png New Business Transaction Match Rule - Split %}

Note the number of arguments (2) set in the above image. That value is important and the transactions will not show up at all if the wrong value is used. We determined the value by decompiling the Weblogic class but you can always do it by first trying 1 and then 2.

With the above configuration in place the AppDynamics agent is able to pick up the different transactions. The transaction names aren't perfect but it works!

{% img center /images/split_bt_on_OSB_AppD/discovered_business_transactions.png Discovered business transactions %}

### Optimise and get the correct name

This solution is OK, but it has a few issues. Firstly the transaction names are either cut off or include characters that are not part of the transaction name. It is also not very efficient, because it requires the entire _MessageContextImpl_ instance to be serialised as a String just to extract a small part of it. To improve this we need to add custom code to the _MessageContextImpl_ class so that we can access the data in a more efficient way.

Consider the following Java code to search a string for the transaction name:

{% codeblock lang:java %}
	private static final SEARCH_TOKEN = "TransactionName=\"";

	private static String getTransactionType(String input) {
		int startIndex = input.indexOf(SEARCH_TOKEN); 				 
		int endIndex = startIndex;
		
		if (startIndex == -1) return null; 					
		
		startIndex += SEARCH_TOKEN.length(); 			//Jump to the open quote
		if (startIndex < input.length() -1) {
			endIndex = input.indexOf("\"", startIndex);		//Find the end quote
		}
		
		if (endIndex > startIndex && endIndex < input.length()) {		
			return input.substring(startIndex, endIndex);
		}
		
		return null;
	}
{% endcodeblock %}

This is a statically accessible piece of code that will extract the transaction name from an arbitrary string. It first tries to find the token in the input string. Once the token is found it determines the open and end quote positions and returns the transaction name. If nothing is found then return _null_. 

The next step is to write some Java code that can use the above code without loading the entire string into memory.
{% codeblock lang:java %}
	public static short WITHIN = 512;      
	public static short BUFFER_SIZE = 256;

	public static String getTransactionType(InputStream inputStream) {
		String result = null;
		BufferedReader reader = null; 
		try {
			reader = new BufferedReader(new InputStreamReader(inputStream), BUFFER_SIZE);
			int read, total = 0;

			//Read up to BUFFER_SIZE and then stop
			StringBuilder sb = new StringBuilder();
			boolean stop = false;
			do {
				char[] cbuf = new char[BUFFER_SIZE];

				read = reader.read(cbuf, 0, BUFFER_SIZE);
				if(read > -1) {
					sb.append(cbuf);
                                    
					//Search for the transaction type in the buffer
					result = getTransactionType(sb.toString());    

					total = total + read;
					stop = stop || result != null || total >= WITHIN;
				}
			} while (!stop && (read != -1));
			reader.close();

			return result;
		}
		catch (Throwable e) {
			Logger.INSTANCE.trace("Failed: {0}",e, e.getMessage() );
		}
		finally {
		//Omitted clean up code
		}
		return null; //Something went wrong
	}
{% endcodeblock %}

This method accepts an _InputStream_ and progressively reads it, 256 characters at a time, to find the transaction type. It is limited to search only the first 512 characters as an optimisation based on the known message structure. It will likely always find the transaction type within the first 256 characters, but 512 makes it a certainty. Also note the variables _WITHIN_ and *BUFFER_SIZE*, which are there to make the code configurable and future proof.

The code listed above can be included in a custom Java agent that will instrument the Weblogic code using a [ClassFileTransformer][class_transformer]. Creating Java agents and class transformers are out of the scope of this article. It focuses on the bits actually injected. For more on creating a custom Java agent see the [java.lang.instrument][java_instrument] documentation.

The next step is making the above _getTransactionType_ accessible by the AppDynamics agent. 

### Using custom byte code injection to expose internals to AppDynamics

Byte code injection can be achieved in different ways, one way is using the [ASM library][asm]. The basic idea is to inject a method into the _MessageContextImpl_ class that can be accessed by AppDynamics as a getter on the first parameter of the _processMessage_ method of _AssignRuntimeStep_. 

So for the agent to inject the following piece of code into the _MessageContextImpl_ class,

{% codeblock lang:java %}

	public String ec_getTransType() {
		try {
			Logger.INSTANCE.debug("getTransactionType called");
			return TransactionTypeExtractor.getTransactionType(getBody().getInputStream(null));
		} catch (Exception e) {
			Logger.INSTANCE.error("Failed to get transactionType", e);
		}
		return null; // Something went wrong
	}

{% endcodeblock %}

you can use ASM as listed below. It effectively writes the above method into the _MessageContextImpl_ class before it is loaded by the class loader. For more information on how to use ASM see the [ASM User Guide][asm_guide].

{% codeblock lang:java %}

   	MethodVisitor mv = super.visitMethod(ACC_PUBLIC, "ec_getTransType", "()Ljava/lang/String;", null, null);
   	mv.visitCode();
    	Label l0 = new Label();
    	Label l1 = new Label();
    	Label l2 = new Label();
    	mv.visitTryCatchBlock(l0, l1, l2, "java/lang/Exception");
    	mv.visitLabel(l0);
    	mv.visitFieldInsn(GETSTATIC, "au/com/ecetera/javaagent/logging/Logger", "INSTANCE", "Lau/com/ecetera/javaagent/logging/Logger;");
    	mv.visitLdcInsn("getTransactionType called");
    	mv.visitInsn(ICONST_0);
    	mv.visitTypeInsn(ANEWARRAY, "java/lang/Object");
    	mv.visitMethodInsn(INVOKEVIRTUAL, "au/com/ecetera/javaagent/logging/Logger", "debug", "(Ljava/lang/String;[Ljava/lang/Object;)V", false);
    	Label l3 = new Label();
    	mv.visitLabel(l3);
    	mv.visitVarInsn(ALOAD, 0);
    	mv.visitMethodInsn(INVOKEVIRTUAL, "com/bea/wli/sb/context/MessageContextImpl", "getBody", "()Lcom/bea/wli/sb/sources/Source;", false);
    	mv.visitInsn(ACONST_NULL);
    	mv.visitMethodInsn(INVOKEINTERFACE, "com/bea/wli/sb/sources/Source", "getInputStream", "(Lcom/bea/wli/sb/sources/TransformOptions;)Ljava/io/InputStream;", true);
    	mv.visitMethodInsn(INVOKESTATIC, "au/com/ecetera/javaagent/vha/TransactionTypeExtractor", "getTransactionType", "(Ljava/io/InputStream;)Ljava/lang/String;", false);
    	mv.visitLabel(l1);
    	mv.visitInsn(ARETURN);
    	mv.visitLabel(l2);
    	mv.visitFrame(Opcodes.F_SAME1, 0, null, 1, new Object[] {"java/lang/Exception"});
    	mv.visitVarInsn(ASTORE, 1);
    	Label l4 = new Label();
    	mv.visitLabel(l4);
    	mv.visitFieldInsn(GETSTATIC, "au/com/ecetera/javaagent/logging/Logger", "INSTANCE", "Lau/com/ecetera/javaagent/logging/Logger;");
    	mv.visitLdcInsn("Failed to get transactionType");
    	mv.visitVarInsn(ALOAD, 1);
    	mv.visitInsn(ICONST_0);
    	mv.visitTypeInsn(ANEWARRAY, "java/lang/Object");
    	mv.visitMethodInsn(INVOKEVIRTUAL, "au/com/ecetera/javaagent/logging/Logger", "error", "(Ljava/lang/String;Ljava/lang/Throwable;[Ljava/lang/Object;)V", false);
    	Label l5 = new Label();
    	mv.visitLabel(l5);
    	mv.visitInsn(ACONST_NULL);
    	mv.visitInsn(ARETURN);
    	Label l6 = new Label();
    	mv.visitLabel(l6);
    	mv.visitLocalVariable("this", "Lcom/bea/wli/sb/context/MessageContextImpl;", null, l0, l6, 0);
    	mv.visitLocalVariable("e", "Ljava/lang/Exception;", null, l4, l5, 1);
    	mv.visitMaxs(4, 2);
    	mv.visitEnd();

{% endcodeblock %}

### Change AppDynamics configuration

Now the AppDynamics agent configuration can be updated to use the new *ec_getTransType* method.

{% img center /images/split_bt_on_OSB_AppD/new_bt_match_rule_split_2.png "Using new method to split" %}

The resulting business transaction names now looks much better.

{% img center /images/split_bt_on_OSB_AppD/discovered_business_transactions_2.png "Properly named transactions" %}

### Conclusion

With AppDynamics it is possible to get really useful information out of a running application. It has very flexible configuration, which allows you to really dive deep into the application internals to find issues and separate transactions. However, sometimes it is better to give AppDynamics a hook into the internal information so that it can work more efficiently. When you have access to the application code then this can easily be achieved by adding some code. When it is not practical to rebuild the entire application then you can always use byte code injection. 


<a href="#references"></a>
[oracleosb]: http://www.oracle.com/technetwork/middleware/service-bus/overview/index.html 
[midc]: http://docs.appdynamics.com/display/PRO14S/Configure+Data+Collectors
[matchconditions]: http://docs.appdynamics.com/display/PRO14S/Match+Rule+Conditions
[asm]:http://asm.ow2.org/
[asm_guide]:http://download.forge.objectweb.org/asm/asm4-guide.pdf
[java_instrument]: http://docs.oracle.com/javase/7/docs/api/java/lang/instrument/package-summary.html
[class_transformer]:http://docs.oracle.com/javase/7/docs/api/java/lang/instrument/package-summary.html
[ecetera]:http://www.ecetera.com.au
