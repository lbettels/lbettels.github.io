# Writing a Proxy Driver For JDBC


## Motivation: Why would you want to do that?
In my case, I wanted to have a driver that can seamlessly intercept user queries
and alters them in a specific way, to be more specific I wanted to query additional fields transparent to the user.
You could also use the driver to add a "last changed" attribute to additionally store in your database, for an audit
tool for example.

## Some More Information
Before I started this project, I never worked with JDBC or databases in general. I knew that p6spy could
be used to intercept queries, so their project was my starting point. (For anyone interested: Their project is
open source and can be found here on [Github](https://github.com/p6spy/p6spy)). My driver is largely based on
their classes, although I got rid of most class files and some code blocks I did not need. In this short guide I will only highlight the functions I think are interesting. You can find a small example in my GitHub repository right [here](https://github.com/lbettels/simple_proxy).

## Disclaimer
Note that this project is by no means state of the art or even close to it, to be honest I'm pretty sure that
it has bugs somewhere. Some of my project files are just stripped down versions of the p6spy driver and
I am not sure how they are used. I am just writing this Markdown because it took me way to long to figure out something
presumedly simple such as writing a proxy driver. To the best of my knowledge there is no approachable guide out there.

## Getting Started
First of all I want to clarify what the driver does. The driver uses a specific URL to connect to the
database through my driver, and my driver uses MySQL backend.

Application --> my driver (changes the queries) --> MySQL --> Database

The application can't tell the difference between talking to the MySQL driver directly or talking to my driver. This is because my driver seamlessly intercepts the user queries and then hands the altered queries to the MySQL driver.

## The Driver Class

To implement the main driver class we simply have to implement the [Oracle
driver interface](https://docs.oracle.com/javase/7/docs/api/java/sql/Driver.html).
The most interesting thing is going on with the selection of the backend driver. The ``acceptsURL()`` function looks for a specific pattern in the URL you use to connect to your database.
```java
@Override
public boolean acceptsURL(final String url) {
    return url != null && url.startsWith("jdbc:mydriver:");
}
```
For example, if your database URL is "jdbc:mysql://localhost:3306/sample_db",  the driver would accept the url "jdbc:mydriver:mysql://localhost:3306/sample_db".
The driver extracts the real URL that you would normally use to connect to your database with the `extractRealUrl()` function.

```java
private String extractRealUrl(String url) {
    return acceptsURL(url) ? url.replace("mydriver:", "") : url;
}
```
 The other important functions are `findPassthru()` and `registeredDrivers()`.
The `findPassthru()` function will find the driver that's able to connect to the database.

```java
protected Driver findPassthru(String url) throws SQLException {

    String realUrl = extractRealUrl(url);
    Driver passthru = null;
    for (Driver driver: registeredDrivers() ) {
        try {
            if (driver.acceptsURL(extractRealUrl(url))) {
                passthru = driver;
                break;
            }
        } catch (SQLException e) {
        }
    }
    if( passthru == null ) {
        throw new SQLException("Unable to find a driver that accepts " + realUrl);
    }
    return passthru;
}
```
The `registeredDrivers()` function is a really simple function that returns all registered drivers.

```java
static List<Driver> registeredDrivers() {
    List<Driver> result = new ArrayList<Driver>();
    for (Enumeration<Driver> driverEnumeration = DriverManager.getDrivers(); driverEnumeration.hasMoreElements(); ) {
        result.add(driverEnumeration.nextElement());
    }
    return result;
}
```
The `connect()` function takes the correct driver and returns a connection to the database.
```java
@Override
public Connection connect(String url, Properties properties) throws SQLException {
    // if there is no url, we have problems
    if (url == null) {
        throw new SQLException("url is required");
    }

    if( !acceptsURL(url) ) {
        return null;
    }

    // find the real driver for the URL
    Driver passThru = findPassthru(url);

    final Connection conn;

    try {
        conn =  passThru.connect(extractRealUrl(url), properties);
    } catch (SQLException e) {
        throw e;
    }

    return ConnectionWrapper.wrap(conn);
}
```
Therefore if you want to use MySQL as your backend, you must include
the respective driver dependency in your project (e.g. in your build.gradle). But a driver that just returns a connection through another driver is
kind of useless. To actually intercept the queries, we take the connection and wrap it with the ``ConnectionWrapper`` class.

## ConnectionWrapper

The `ConnectionWrapper` class has to implement the Java [Connection](https://docs.oracle.com/javase/7/docs/api/java/sql/Connection.html)
interface and has to extend the `AbstractWrapper` class. What I am doing (and the guys from p6spy are doing) is putting the query I alter
with my `parseSql` function into another wrapper class, depending on the type of the statement.


We take the original sql String, manipulate it and then let our `delegate` connection
talk to the database. The `delegate` connection is the underlying connection from our `MyProxyDriver` class, therefore
it is the backend driver that handles the connection to our database.
The delegate is the connection to the actual MySQL database.

Here is a basic `wrap` function and a simple constructor.
```java
public static ConnectionWrapper wrap(Connection delegate) {
    if (delegate == null) {
        return null;
    }
    final ConnectionWrapper connectionWrapper = new ConnectionWrapper(delegate);
    return connectionWrapper;
}

protected ConnectionWrapper(Connection delegate) {
    super(delegate);
    if (delegate == null) {
        throw new NullPointerException("Delegate must not be null");
    }
    this.delegate = delegate;
}
```
To change the queries, we will intercept them before we hand them to the delegate driver. For example, the connection interface has a `prepareStatement` function. We will take the sql-String, parse it and hand the changed query to our MySQL driver.

```java
@Override
public PreparedStatement prepareStatement(String sql) throws SQLException {
    String transformedSql = parseSql(sql); //this is where i change the query
    return delegate.prepareStatement(transformedSql);
}
```
You can implement every other function in the `connection` interface in a similar fashion. In my project, I use lots of different Wrapper classes, one for each "statement type". I use a `StatementWrapper`,`PreparedStatementWrapper`,`CallableStatementWrapper` and a `ResultSetWrapper`.

Note that you do not have to use even more wrapper classes. Even though they were very helpful for my project, I think the
`MyProxyDriver` and the `ConnectionWrapper` class should be sufficient to implement a simple proxy driver.
The wrapper classes also have to extend the `AbstractWrapper` class.
In the [repository](https://github.com/lbettels/simple_proxy) I published the version consisting only of the `MyProxyDriver` class, the `StatementWrapper` class and the `ConnectionWrapper` class.
If you want to have additional functions, you can call the other wrapper classes similar to this code:

```java
@Override
public PreparedStatement prepareStatement(String sql) throws SQLException {
    String transformedSql = parseSql(sql); //this is where i change the query
    return PreparedStatementWrapper.wrap(delegate.prepareStatement(transformedSql),sql,transformedSql);
}
```

## AbstractWrapper

The `AbstractWrapper` implements the [Wrapper interface](https://docs.oracle.com/javase/7/docs/api/java/sql/Wrapper.html).

The documentation states that "The wrapper pattern is employed by many JDBC driver implementations to provide extensions
beyond the traditional JDBC API that are specific to a data source. Developers may wish to gain access to these resources
that are wrapped (the delegates) as proxy class instances representing the actual resources. This interface describes a
standard mechanism to access these wrapped resources represented by their proxy, to permit direct access to the resource delegates."

To be honest I am not quite sure how this class is used, I took it from the p6spy project right [here](https://github.com/p6spy/p6spy/blob/master/src/main/java/com/p6spy/engine/wrapper/AbstractWrapper.java).

## Example for a StatementWrapper
For the people that want additional functionality: This is how a simple StatementWrapper-class looks like:

```java
public class StatementWrapper extends AbstractWrapper implements Statement {
    private static final String LINE_SEPARATOR = System.getProperty("line.separator");
    private final Statement delegate;

    public static Statement wrap(Statement delegate) {
        //put your code here//
        if (delegate == null) {
            return null;
        }
        return new StatementWrapper(delegate);
    }

    protected StatementWrapper(Statement delegate) {
        //put your code here//
        super(delegate);
        this.delegate = delegate;
    }

    @Override
    public ResultSet getResultSet() throws SQLException {
        //put your code here//
        return ResultSetWrapper.wrap(delegate.getResultSet());
    }

    ...

  }
```

As you can see the StatementWrapper class extends the `AbstractWrapper` class and implements
the [Statement interface](https://docs.oracle.com/javase/7/docs/api/java/sql/Statement.html).
You simply have to implement the functions of the interface and you are good to go. It is not necessary to wrap the results in a `ResultSetWrapper`.
If you want to have another wrapper e.g. a `ResultSetWrapper`, the `ResultSetWrapper` class
has to extend the `AbstractWrapper` class and implement the [ResultSet interface](https://docs.oracle.com/javase/7/docs/api/java/sql/ResultSet.html).

If you are not sure why you should use a ResultSet-Wrapper: In my case I used it to return only queries with specific indexes transparent to the enduser. The number of wrapper-classes really is dependent on the task you want to achieve.

## META-INF
In order to use your driver in another project, the [driver manager](https://docs.oracle.com/javase/8/docs/api/java/sql/DriverManager.html) needs to know about your driver.
If you want your driver to be registered automatically upon embedding, you have to create a file called
`java.sql.Driver`. This file just contains one line, which is the path to the `MyProxyDriver` in your project.
So if your `MyProxyDriver` class is in the package `foo.bar`, the content of the `java.sql.Driver` file would be
`foo.bar.MyProxyDriver`. Put your `java.sql.Driver` file into resources -> META-INF -> services.

# Additional Information for Publishing to Maven Local

## build.gradle
```groovy
plugins {
    id 'java'
    id 'maven-publish'
}

group 'foo.bar'
version '1.0-SNAPSHOT'

repositories {
    mavenLocal()
    mavenCentral()
}

dependencies {
    compile group: 'mysql', name: 'mysql-connector-java', version: '8.0.19'
    //put the dependencies you need here
}

publishing {
    publications {
        maven(MavenPublication) {
            groupId = 'foo.bar.MyProxyDriver'
            artifactId = 'MyProxyDriver'
            version = '0.1'

            from components.java
        }
    }
}
```
You need the java plugin to publish your java code to your local maven repository.
The interesting block is the one at the bottom. You just have to set a `groupId`, an `artifactId` and
a `version` and you are good to go and publish your project to your local maven repository.

## Using it in Another Project

```groovy
plugins {
  id 'java'
}

group 'foo.bar'
version '1.0-SNAPSHOT'

repositories {
  mavenLocal()
  mavenCentral()
}

dependencies {
  compile group:'foo.bar.MyProxyDriver', name:'MyProxyDriver', version: '0.1'
}
```
If you finally want to use your driver, just specify your dependency in your `build.gradle`, just
as you would with every other dependency. Keep in mind that you have to use your local maven repository.
