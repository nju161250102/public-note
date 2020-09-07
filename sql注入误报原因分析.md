## SQL注入的常见误报原因

检测工具：FindBugs 3.0.1

总体原因：

- 检测工具忽视了代码中各种对输入隐含的约束，或者将安全的情况识别为不安全的情况
- 未使用的变量、无法到达的语句……这类警告
- 其他原因

### 例1——105

漏洞分类：`SQL_NONCONSTANT_STRING_PASSED_TO_EXECUTE`

漏洞描述：`Value non-constant SQL string involving HTTP taint`

行号：75

解释：没有经过约束检验的字符串直接被传入执行SQL

```java
        String bar;
		
		// Simple ? condition that assigns constant to bar on true condition
		int num = 106;
		bar = (7*18) + num > 200 ? "This_should_always_happen" : param;

		String sql = "SELECT * from USERS where USERNAME='foo' and PASSWORD='"+ bar +"'";
				
		try {
			java.sql.Statement statement = org.owasp.benchmark.helpers.DatabaseHelper.getSqlStatement();
>			statement.addBatch( sql );
			int[] counts = statement.executeBatch();
            org.owasp.benchmark.helpers.DatabaseHelper.printResults(sql, counts, response);
		} catch (java.sql.SQLException e) {
			if (org.owasp.benchmark.helpers.DatabaseHelper.hideSQLErrors) {
        		response.getWriter().println("Error processing request.");
        		return;
        	}
			else throw new ServletException(e);
		}
```

误报产生原因：没有识别出前面的约束，这里使用的是三元运算符 ? 做出判断。进一步地，这类误报可能是由于忽略了业务逻辑上的约束，例如在存入数据库的时候已经做了校验，因此查询的时候就不需要校验。

### 例2——107

漏洞分类：`DLS_DEAD_LOCAL_STORE`

漏洞描述：`Local variable named f18521`

行号：73

解释：事实上是一个未使用的变量

```java
String f18521 = e18521.split(" ")[0]; // split it on a space
// 之后没有被使用
```

误报产生原因：存在着未使用的变量，引发了扫描系统的警告。但是这一警告的优先度并不高（priority为2，相比之下上例中为1）。

---

漏洞分类：`DM_DEFAULT_ENCODING`

漏洞描述：`Reliance on default encoding`

行号：71

解释：编码问题

```java
String e18521 = new String( org.apache.commons.codec.binary.Base64.decodeBase64(
		    org.apache.commons.codec.binary.Base64.encodeBase64( d18521.getBytes() ) )); // B64 encode and decode it
```

误报产生原因：没搞懂……大概是默认编码之类的？

### 例3——113

漏洞分类：`SQL_NONCONSTANT_STRING_PASSED_TO_EXECUTE`

漏洞描述：`Value non-constant SQL string involving HTTP taint`

行号：75

```java
		String bar = "safe!";
		java.util.HashMap<String,Object> map21657 = new java.util.HashMap<String,Object>();
		map21657.put("keyA-21657", "a_Value"); // put some stuff in the collection
		map21657.put("keyB-21657", param); // put it in a collection
		map21657.put("keyC", "another_Value"); // put some stuff in the collection
		bar = (String)map21657.get("keyB-21657"); // get it back out
		bar = (String)map21657.get("keyA-21657"); // get safe value back out
		
		
		String sql = "INSERT INTO users (username, password) VALUES ('foo','"+ bar + "')";
				
		try {
			java.sql.Statement statement = org.owasp.benchmark.helpers.DatabaseHelper.getSqlStatement();
>			int count = statement.executeUpdate( sql );
            org.owasp.benchmark.helpers.DatabaseHelper.outputUpdateComplete(sql, response);
		} catch (java.sql.SQLException e) {
			if (org.owasp.benchmark.helpers.DatabaseHelper.hideSQLErrors) {
        		response.getWriter().println("Error processing request.");
        		return;
        	}
			else throw new ServletException(e);
		}
```

误报产生原因：从注释中可以看出，最终变量重新被设置成了安全的值（非用户输入的值）