# DbHelper
一个比Apache-commons-dbutils更加好用，更加简洁轻便的微型框架。最新版本为dbhelper-0.1.0。
![镇楼图](http://img.blog.csdn.net/20160719164640373)
---


## 核心理念

本工具以查询为主要核心理念，即query操作为主，覆盖了多种查询方式的自动化运行流程。
其他诸如insert, update, delete以及事务操作，均由QueryRunner的update方法完成。

配备了自定义大小的数据库连接池，提升程序运行的效率，并由底层自动维护连接池的释放，申请问题。省心。



## 如何使用

- Step 1：
	首先在项目的src目录下创建一个db.cfg.xml的文件，格式如下：
```
<?xml version="1.0" encoding="UTF-8" ?>
<project>
	<database name="mysql">
		<driver>com.mysql.jdbc.Driver</driver>
		<url>jdbc:mysql://localhost:3306/fams</url>
		<user>root</user>
		<password>mysql</password>
		<poolsize>20</poolsize>
	</database>
	···
</project>
```

- Step 2:
	通过注册的方式实现配置文件中的数据源的配置
	`Dbhelper.register()`即可


- Step 3：
	主要的业务逻辑操作类`QueryRunner`，封装了对数据库JDBC操作的增删改查等一系列的操作。
	对于update方法的JDBC操作，我们无需手动的关闭数据库连接，仅仅简单的通过DbHelper的重载的release方法来实现对数据库连接对象，查询语句以及数据集的关闭操作,自动的维护数据库连接池。是不是感觉很省心啊。


---

## 测试实例

这里先根据数据库中的表结构来创建一个对应的Bean对象吧。
```
/**
 * @Date 2016年11月19日
 *
 * @author Administrator
 */

/**
 * @author 郭璞
 *
 */
public class DateTest {

	private int id;
	private String name;
	private Date date;

	@Override
	public String toString() {
		return "DateTest [id=" + id + ", name=" + name + ", date=" + date + "]";
	}

	public int getId() {
		return id;
	}

	public void setId(int id) {
		this.id = id;
	}

	public String getName() {
		return name;
	}

	public void setName(String name) {
		this.name = name;
	}

	public Date getDate() {
		return date;
	}

	public void setDate(Date date) {
		this.date = date;
	}

}


```



## 数据库内详细数据信息

```
mysql> use test;
Database changed
mysql> select * from datetest;
+----+----------+------------+
| id | name     | date       |
+----+----------+------------+
|  1 | dlut     | 2016-07-06 |
|  2 | 清华大学 | 2016-07-03 |
|  3 | 北京大学 | 2016-07-28 |
|  4 | Mark     | 2016-08-20 |
|  5 | Tom      | 2016-08-19 |
|  6 | Jane     | 2016-08-21 |
+----+----------+------------+
6 rows in set (0.00 sec)

mysql>

```

## 测试代码

### 数据库连接池测试

```
public static void main(String[] args) throws Exception {
		DbHelper helper = new DbHelper();
		helper.register();
		System.out.println(helper.connpool.size());
		Connection conn = helper.getConnection();
		System.out.println(helper.connpool.size());
		Connection conn1 = helper.getConnection();
		System.out.println(helper.connpool.size());
		Connection conn2 = helper.getConnection();
		System.out.println(helper.connpool.size());

		System.out.println(conn.toString());
		System.out.println(conn1.toString());
		System.out.println(conn2.toString());

		helper.release(conn);
		System.out.println(helper.connpool.size());

		PreparedStatement stmt = conn1.prepareStatement("select * from user");
		helper.release(conn2, stmt);
		System.out.println(helper.connpool.size());

		PreparedStatement stmt1 = conn1.prepareStatement("select * from user");
		ResultSet rs = stmt1.executeQuery();
		helper.release(conn2, stmt1, rs);
		System.out.println(helper.connpool.size());

		System.out.println(conn.toString());
		System.out.println(conn1.toString());
		System.out.println(conn2.toString());
	}

```

测试结果：

```
20
19
18
17
dbhelper.DbHelper$MyConnection@28c97a5
dbhelper.DbHelper$MyConnection@6659c656
dbhelper.DbHelper$MyConnection@6d5380c2
18
19
20
dbhelper.DbHelper$MyConnection@28c97a5
dbhelper.DbHelper$MyConnection@6659c656
dbhelper.DbHelper$MyConnection@6d5380c2


```


### 测试集

```
/**
 * @Date 2016年11月19日
 *
 * @author 郭  璞
 *
 */
package mp;

import java.sql.Connection;
import java.util.List;

import org.junit.Test;

import dbhelper.DbHelper;
import dbhelper.QueryRunner;
import handlers.BeanHandler;
import handlers.BeanListHandler;

/**
 * @author 郭  璞
 *
 */
public class TestCase {
	
	@Test
	public void testSingleBean() throws Exception {
		System.out.println("-----------------testSingleBean-------------------------------------");
		String sql = "select * from datetest where id=?";
		QueryRunner runner = new QueryRunner();
		DbHelper.register();

		DbHelper helper = new DbHelper();
		Connection conn = helper.getConnection();
		DateTest datetest = runner.query(conn, sql, new BeanHandler<DateTest>(DateTest.class), 1);
		
		System.out.println(datetest.toString());
		
		DbHelper.release(conn);
	}
	
	
	@Test
	public void testBeanList() throws Exception {
		System.out.println("-------------- testBeanList----------------------------------------");
		String sql = "select * from datetest";
		QueryRunner runner = new QueryRunner();
		DbHelper.register();

		DbHelper helper = new DbHelper();
		Connection conn = helper.getConnection();
		List<DateTest> dates = runner.query(conn, sql, new BeanListHandler<DateTest>(DateTest.class));
		for (DateTest date : dates) {
			System.out.println(date.toString());
		}
		
		DbHelper.release(conn);
	}
	
	@Test
	public void testBeanListWithParams1() throws Exception {
		System.out.println("--------------testBeanListWithParams1----------------------------------------");
		String sql = "select * from datetest where id in (?, ?, ?)";
		QueryRunner runner = new QueryRunner();
		DbHelper.register();

		DbHelper helper = new DbHelper();
		Connection conn = helper.getConnection();
		Object[] params = new Object[]{1, 3, 6};
		List<DateTest> dates = runner.query(conn, sql, new BeanListHandler<DateTest>(DateTest.class), params);
		for (DateTest date : dates) {
			System.out.println(date.toString());
		}
		
		DbHelper.release(conn);
	}

	@Test
	public void testBeanListWithParams2() throws Exception {
		System.out.println("--------------testBeanListWithParams2----------------------------------------");
		String sql = "select * from datetest where id in (?, ?, ?) or name = ?";
		QueryRunner runner = new QueryRunner();
		DbHelper.register();

		DbHelper helper = new DbHelper();
		Connection conn = helper.getConnection();
		Object[] params = new Object[]{1, 3, 6, "Mark"};
		List<DateTest> dates = runner.query(conn, sql, new BeanListHandler<DateTest>(DateTest.class), params);
		for (DateTest date : dates) {
			System.out.println(date.toString());
		}
		
		DbHelper.release(conn);
	}
	
	
	@Test
	public void testupdate() throws Exception {
		System.out.println("--------------testupdate----------------------------------------");
		String sql = "update datetest set name=? where name=?";
		QueryRunner runner = new QueryRunner();
		DbHelper.register();

		DbHelper helper = new DbHelper();
		Connection conn = helper.getConnection();
		Object[] params = new Object[]{"郭璞", "Mark"};
		
		runner.update(conn, sql, params);
		
		DbHelper.release(conn);
	}
}


```

测试结果：

```
-----------------testSingleBean-------------------------------------
DateTest [id=1, name=dlut, date=Wed Jul 06 00:00:00 CST 2016]


-------------- testBeanList----------------------------------------
DateTest [id=1, name=dlut, date=Wed Jul 06 00:00:00 CST 2016]
DateTest [id=2, name=清华大学, date=Sun Jul 03 00:00:00 CST 2016]
DateTest [id=3, name=北京大学, date=Thu Jul 28 00:00:00 CST 2016]
DateTest [id=4, name=郭璞, date=Sat Aug 20 00:00:00 CST 2016]
DateTest [id=5, name=Tom, date=Fri Aug 19 00:00:00 CST 2016]
DateTest [id=6, name=Jane, date=Sun Aug 21 00:00:00 CST 2016]



--------------testBeanListWithParams1----------------------------------------
DateTest [id=1, name=dlut, date=Wed Jul 06 00:00:00 CST 2016]
DateTest [id=3, name=北京大学, date=Thu Jul 28 00:00:00 CST 2016]
DateTest [id=6, name=Jane, date=Sun Aug 21 00:00:00 CST 2016]



--------------testBeanListWithParams2----------------------------------------
DateTest [id=1, name=dlut, date=Wed Jul 06 00:00:00 CST 2016]
DateTest [id=3, name=北京大学, date=Thu Jul 28 00:00:00 CST 2016]
DateTest [id=6, name=Jane, date=Sun Aug 21 00:00:00 CST 2016]




--------------testupdate----------------------------------------




```

数据库中对应更新结果：

```
mysql> use test;
Database changed
mysql> select * from datetest;
+----+----------+------------+
| id | name     | date       |
+----+----------+------------+
|  1 | dlut     | 2016-07-06 |
|  2 | 清华大学 | 2016-07-03 |
|  3 | 北京大学 | 2016-07-28 |
|  4 | Mark     | 2016-08-20 |
|  5 | Tom      | 2016-08-19 |
|  6 | Jane     | 2016-08-21 |
+----+----------+------------+
6 rows in set (0.00 sec)

mysql> select * from datetest;
+----+----------+------------+
| id | name     | date       |
+----+----------+------------+
|  1 | dlut     | 2016-07-06 |
|  2 | 清华大学 | 2016-07-03 |
|  3 | 北京大学 | 2016-07-28 |
|  4 | 郭璞     | 2016-08-20 |
|  5 | Tom      | 2016-08-19 |
|  6 | Jane     | 2016-08-21 |
+----+----------+------------+
6 rows in set (0.00 sec)

```




## 延伸

由于源代码可以随意更改，所以你可以根据自己的需求来实现私人定制。仅仅需要实现自己的handler接口就可以了。
如果你看到了这篇文章，欢迎给我提issue。当然也可以修改成你自己的工具。


## 快速应用

不妨参照下面的这篇文章，相信会给您些许灵感。 



------
