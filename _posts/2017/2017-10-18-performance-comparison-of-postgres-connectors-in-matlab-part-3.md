---
layout: post
title:  "Performance comparison of Postgres connectors in Matlab, Part 3: retrieving arrays"
date:   2017-10-18 19:14:27 +0300
authors: Peter Gagarinov & Ilya Rublev
---

In this paper we continue the investigation of **PostgreSQL** connectors in Matlab started
in [Part I](https://alliedtesting.github.io/pgmex-blog/2017/06/29/performance-comparison-of-postgresql-connectors-in-matlab-part-I/)
and [Part II](https://alliedtesting.github.io/pgmex-blog/2017/09/29/performance-comparison-of-postgresql-connectors-in-matlab-part-II/).
In the latter paper we compared the performance of data retrieval from **PostgreSQL** for the case all the fields to be retrieved
having scalar types. Compared were two connectors. The first one was **Matlab Database Toolbox** (working with **PosgteSQL** via
a direct JDBC connection). The second connector is [**PgMex library**](http://pgmex.alliedtesting.com) 
(providing a connection to **PostgreSQL** via libpq library). Here we consider retrieving of data containing values of
[**array types**](https://www.postgresql.org/docs/9.6/static/arrays.html).
<!--end_of_excerpt-->

Let us recall the example already mentioned in [Part I](https://alliedtesting.github.io/pgmex-blog/2017/06/29/performance-comparison-of-postgresql-connectors-in-matlab-part-I/)
illustrating the importance of retrieving arrays from **PostgreSQL**. In one of our projects we had to work with surfaces of implied volatilities derived
from prices of options. But these surfaces should be interpolated because raw data does not give their values on some regular mesh of strikes (or deltas) and
times to maturity of respective options. There are "holes" in data because traded option parameters such as option strike and maturity do not form a regular grid in option delta/time to
maturity space. Filling such holes requires an interpolation which is not easy to do quickly given large volumes option market data.
In our project we managed to perform this interpolation directly in **PostgreSQL** database in a quite efficient manner.
But it was necessary then to retrieve the results of interpolation. And these results in form of implied volatility surfaces were quite naturally represented in the database
as three-dimensional arrays due to the parametrization mentioned above and dependency of these surfaces on time.

Given the nature of such a financial data it is quite easy to image a few real-world scenarios where a possibility to retrieve this
data in large amounts very quickly is very important. Below we reveal some latent restrictions (concerning both performance and
volumes of data to be processed) that do not allow our development team to use **Matlab Database Toolbox** in such scenarios. An
alternative solution, [**PgMex library**](http://pgmex.alliedtesting.com), was developed by our team to overcome these restrictions
and to allow us to efficiently solve financial data processing problems.

In the performance benchmarks below we use the data that has also to do with financial modelling, although not of such
complex nature as in the example mentioned above (the one with the implied volatility surfaces). But this connection namely in the case under consideration may seem rather artificial.
This is because the test data is simply transformed from the one taken from the previous articles. This is done just for simplicity
and uniformity with the results obtained earlier, and thus - allowing us to concentrate on purely technical problems.
The data is based on daily prices of real stocks on some exchanges with a few modifications: all the numerical fields having initially
scalar values are transormed into arrays just be repeating each value several times. The same data was used in
[Part I](https://alliedtesting.github.io/pgmex-blog/2017/06/29/performance-comparison-of-postgresql-connectors-in-matlab-part-I/#inserting-arrays)
when we tested performance of *inserting* data that contained arrays. We omit here details concerning the structure of the
test data we use. But the latter as well as conditions of the experiments are the same as in
[Part I](https://alliedtesting.github.io/pgmex-blog/2017/06/29/performance-comparison-of-postgresql-connectors-in-matlab-part-I/#experiment-conditions-for-comparison-between-matlab-database-toolbox-and-pgmex).

## Retrieving of arrays in **Matlab Database Toolbox**

We use here for **Matlab Database Toolbox** the same method to retrieve data as in
[Part II](https://alliedtesting.github.io/pgmex-blog/2017/09/29/performance-comparison-of-postgresql-connectors-in-matlab-part-II/#methods-of-data-retrieval-in-matlab-database-toolbox).
Namely, this is a consecutive execution of **exec** and **fetch** methods. And we concentrate on details 
namely for retrieval of arrays.

The format in which data may be returned is configured by **DataReturnFormat** parameter via **setdbprefs** function. 
Setting this parameter to 'numeric' is not reasonable because in such case all the values of array types are lost as they are substituted by the value of 
**NullNumberRead** parameter (also set via **setdbprefs** function). Thus, it makes sense to consider only the cases
when **DataReturnFormat** parameter is set to 'cellarray' or 'structure'. In the examples below we assume that `dbConn` is an object 
of **database.jdbc.connection** class obtained by the command

```matlab
dbConn=database(dbName,userName,passwordStr,...
    'Vendor','PostgreSQL','Server',hostPortStr);
```

Here `dbName`, `userName`, `passwordStr` and `hostPortStr` are assumed to be char variables properly set,
`hostPortStr` has the format `<serverName>:<portNumber>`.

At first let us consider the case when **DataReturnFormat** parameter is set to 'cellarray'. Then all the values of array types
are returned as objects of **org.postgresql.jdbc.PgArray** class, each placed in a separate cell of the cell array
being the single output of **fetch**:

```matlab
>> cursObj=fetch(exec(dbConn,[...
       'select ''{1,2}''::int4[] as f union '...
       'select ''{3}''::int4[] as f']));

>> cursObj.Data
ans = 
    [1x1 org.postgresql.jdbc.PgArray]
    [1x1 org.postgresql.jdbc.PgArray]
```

Thus, we need to convert each such an object into Matlab array. This is quite simple: **org.postgresql.jdbc.PgArray** 
has a method **getArray** returning Java array. That array then can be easily transformed into a Matlab cell array 
via **cell** function (see
[here](https://www.mathworks.com/help/matlab/matlab_external/handling-data-returned-from-java-methods.html#bvi1br7-5)
for example):

```matlab
>> jdbcFieldVal=cursObj.Data{1}
jdbcFieldVal =
{1,2}

>> class(jdbcFieldVal)
ans =
org.postgresql.jdbc.PgArray

>> matlabFieldVal=cell(jdbcFieldVal.getArray())
matlabFieldVal = 
    [1]
    [2]
```

As a result each element of the respective array is in its own cell. It is necessary to note here that for NULL
values the corresponding cells are empty. Another remarks is that if the array as whole (not some of its elements) is
initially NULL, it is returned not as **org.postgresql.jdbc.PgArray** object, but as a string equal to the value
of **NullStringRead** parameter (set via **setdbprefs**):

```matlab
>> cursObj=fetch(exec(dbConn,[...
       'select ''{1,NULL,2}''::int4[] as f union '...
       'select NULL::int4[] as f']));

>> cursObj.Data
ans = 
    [1x1 org.postgresql.jdbc.PgArray]
    'null'

>> class(cursObj.Data{2})
ans =
char

>> jdbcFieldVal=cursObj.Data{1};

>> matlabFieldVal=cell(jdbcFieldVal.getArray())
matlabFieldVal = 
    [1]
    []
    [2]
```

Thus, we have the full information on NULLs. And, assuming there are no NULLs (or placing into all empty cells some value,
say, `0` or `NaN`), we are able to transform each of **org.postgresql.jdbc.PgArray** objects into ordinary Matlab array.

Now let us consider the case when **DataReturnFormat** is equal to 'structure'. Then values of each table field are placed
into the corresponding field of the structure returned by **fetch**. Moreover each array field now has **java.lang.Object\[\]\[\]** type:

```matlab
>> cursObj=fetch(exec(dbConn,[...
       'select ''{1,NULL}''::int4[] as f union '...
       'select NULL::int4[] as f']));

>> cursObj.Data
ans = 
    f: [2x1 java.lang.Object[][]]
```

Let us put all values of one of the fields into a separate variable named `jdbcFieldValVec`:

```matlab
>> jdbcFieldValVec=cursObj.Data.f
jdbcFieldValVec =
java.lang.Object[][]:
    [org.postgresql.jdbc.PgArray]
    'null'

>> jdbcFieldVal=jdbcFieldValVec(1)
jdbcFieldVal =
java.lang.Object[]:
    [org.postgresql.jdbc.PgArray]

>> jdbcFieldVal(1)
ans =
{1,NULL,2}

>> class(jdbcFieldVal(1))
ans =
org.postgresql.jdbc.PgArray

>> jdbcFieldVal=jdbcFieldValVec(2);

>> jdbcFieldVal(1)
ans =
null

>> class(jdbcFieldVal(1))
ans =
char
```

We see that `jdbcFieldVal=jdbcFieldValVec(iTuple)` being value of `iTuple`-th tuple
is of **java.lang.Object\[\]** array type and the number of elements in this array is 1. Taking `jdbcFieldVal(1)` we either obtain
an object of **org.postgresql.jdbc.PgArray** class (if the respective array is not NULL as whole) or a string equal to the
value of **NullStringRead** (otherwise). That is we can proceed further with all this exactly as in the case above 
of **DataReturnFormat** equal to 'cellarray'.

The above algorithms are clear and straightforward. But the problem occures when one needs to apply them to several millions of
tuples. As shown below in experiments, the conversion of initially returned data to native Matlab formats sums up to a huge overhead.

And last but not least. Regardless of what type each array of numerical type has in **PostgeSQL**, applying
**cell** to some Java array (as in the approach above) transforms the latter array into **double** Matlab array
(see [here](https://www.mathworks.com/help/matlab/matlab_external/handling-data-returned-from-java-methods.html)
for details). This leads to a few rather serious drawbacks:

- conversion inaccuracies are possible;
- returned data has a larger size than really necessary.

Let us have a look at a few examples for each of these problems and explain the corresponding side-effects in more details.
As for conversions imprecisions, the problem is as follows. Each number of **double** Matlab type occupies 8 bytes,
and only 52 bits from these 8 bytes are used for the fractional part (1 bit is for the sign, 11 bits are for the exponent).
If we convert some **int64** Matlab integer into **double**, these 52 bits may be insufficient to ensure an accurate value conversion. For instance, if we try to cast the maximal possible value of **int64** equal to 9223372036854775807
to **double**, we obtain 9223372036854775800 instead of the original value. Hence we cannot be sure that such a conversion does not
lead to the described data loss. And this problem is exactly there for one of the fields in our test data, namely,
**volume**. This is because the latter has **int8\[\]** type in **PostgreSQL**, that corresponds to **int64** array in Matlab.

Now let us assume that all the values to be retrieved can be cast into **double** arrays without any data loss. But even in that
situation we are left with the second drawback of very inefficient usage of operative memory. The most of the fields in the test
data are of **float4\[\]** type, i.e. each element of their values occupies 4 bytes. And for our data set a result representation
generated by **fetch** consumes almost twice more space than it should. That is because all the fields having **float4\[\]** type
with **float4** correspond to **single** Matlab type, not **double** type. Obviously the situation could potentially be even worse
if one has to retrieve logical values. In this case **fetch** representation would consume 8 times more RAM comparing to the
original data size. This inefficient data representation in **fetch** results leads to a memory shortage when the total number of
tuples to be returned is significant.

## Retrieving of arrays in **PgMex**

For [**PgMex**](http://pgmex.alliedtesting.com) we again use the same way for data retrieval as in 
[Part II](https://alliedtesting.github.io/pgmex-blog/2017/09/29/performance-comparison-of-postgresql-connectors-in-matlab-part-II/#methods-of-data-retrieval-in-pgmex).
That is we apply consecutively two commands: [**exec**](http://pgmex.alliedtesting.com/#exec) and
[**getf**](http://pgmex.alliedtesting.com/#getf). The latter command transforms results into a Matlab friendly format.
Namely, its outputs are structures, one structure per each field of retrieved table.
For the case of table fields being of array types the fields of the corresponding structures are of the following types:

- **valueVec** - a column cell array of values (multidimensional arrays) extracted from the database;
- **isNullVec** - a column cell array of logical multidimensional arrays with indicators of value elements being NULL;
- **isValueNullVec** - a logical column vector with indicators of entire table cell (a field in a specific tuple) being NULL.

The respective cells of **valueVec** and of **isNullVec** contains arrays of the same size. At that for **valueVec**
this array contains values of elements of the retrieved **PostgreSQL** array (with NULL values substituted by
some predefined values). And for **isNullVec** the respective array determines which elements of this **PostgreSQL**
array are equal to NULL. But all this is of importance only in the case the retrieved array is not NULL *as whole*
within **PostgreSQL**, i.e. when the corresponding element **isValueNullVec** is `false`. Otherwise, when
element of **isValueNullVec** equals `true` the respective cells within **valueVec** and **isNullVec** are empty.

Let us illustrate this through the following example (we assume below that `<hostName>`, `<dbName>`, `<portNum>`,
`<userName>` and `<passwordStr>` are properly filled):

```matlab
>> import com.allied.pgmex.pgmexec;

>> dbConn=pgmexec('connect',[...
       'host=<hostName> dbname=<dbName> port=<portNum> '...
       'user=<userName> password=<passwordStr>']);

>> pgResult=pgmexec('exec',dbConn,[...
       'select ''{1,NULL,2}''::int4[] as f union '...
       'select NULL::int4[] as f']);

>> SRes=pgmexec('getf',pgResult,'%int4[]',0)
SRes = 
          valueVec: {2x1 cell}
         isNullVec: {2x1 cell}
    isValueNullVec: [2x1 logical]

>> SRes.valueVec
ans = 
    [3x1 int32]
    []

>> SRes.valueVec{1}
ans =
           1
           0
           2

>> SRes.isNullVec
ans = 
    [3x1 logical]
    []

>> SRes.isNullVec{1}
ans =
     0
     1
     0

>> SRes.isValueNullVec
ans =
     0
     1
```

When compared with **Matlab Database Toolbox**, [**PgMex**](http://pgmex.alliedtesting.com) allows to obtain
results immediately in Matlab native formats without any additional conversion overhead and usage of more memory than
necessary. Besides, NULLs are represented in rather safe and consistent manner. 

## Experiment results

When **DataReturnFormat** is 'cellarray' the results are as follows:

<!---
![Retrieving of arrays, 'cellarray' mode](pictures/compareRetrieveForArraysAsCellArray_traj.jpeg)
-->
![Retrieving of arrays, 'cellarray' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForArraysAsCellArray_traj.jpeg)
<!---
![Retrieving of arrays, 'cellarray' mode](pictures/compareRetrieveForArraysAsCellArray_traj_cropped.jpeg)
-->
![Retrieving of arrays, 'cellarray' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForArraysAsCellArray_traj_cropped.jpeg)
<!---
![Retrieving of arrays, 'cellarray' mode](pictures/compareRetrieveForArraysAsCellArray_bar.jpeg)
-->
![Retrieving of arrays, 'cellarray' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForArraysAsCellArray_bar.jpeg)

One can see the red graphs on the pictures above corresponding to **fetch** "break" when a number of tuples reaches 80000.
This is because of Java heap memory problem occuring while conversion of results for a number of tuples equal to 90000,
just 104Mb in size. The exception is as follows:

```matlab
Exception for function fetch, number of tuples 90000

Caused by:
    Error using com.allied.pgmex.perftest.ATestCompareWithJDBC>performRetrieve (line 1537)
    Java exception occurred: 
    java.lang.OutOfMemoryError: GC overhead limit exceeded
    	at java.lang.AbstractStringBuilder.<init>(Unknown Source)
    	at java.lang.StringBuffer.<init>(Unknown Source)
    	at com.mathworks.jmi.OpaqueJavaInterface.getSignatureOfClass(OpaqueJavaInterface.java:515)
    	at com.mathworks.jmi.OpaqueJavaInterface.getSignatureOfClass(OpaqueJavaInterface.java:521)
    	at com.mathworks.jmi.OpaqueJavaInterface.getSignatureOfObjectClass(OpaqueJavaInterface.java:602)
```

Starting with 90000 tuples [**exec**](http://pgmex.alliedtesting.com/#exec) together with [**getf**](http://pgmex.alliedtesting.com/#getf) methods
from [**PgMex**](http://pgmex.alliedtesting.com) are the only ones that successfully solve the problem of data retrieval. 
On our configuration [**PgMex**](http://pgmex.alliedtesting.com) retrieves the test data of 2000000 tuples (2034Mb)
approximately in 28 seconds.

It turns out that if we only take a range [0, 80000] for a number of tuples (as number of tuples = 80000 is where the red graphs end)
on average [**exec**](http://pgmex.alliedtesting.com/#exec) and [**getf**](http://pgmex.alliedtesting.com/#getf) are approximately 42 (!)
times faster than **exec** and **fetch** from **Matlab Database Toolbox** along with conversion of results.

But if we consider only "pure" time, without conversion of Java objects into native Matlab formats, even in this case
[**PgMex**](http://pgmex.alliedtesting.com) is approximately 4.5 times faster than **Matlab Database Toolbox**:

<!---
![Retrieving of arrays, 'cellarray' mode](pictures/compareRetrieveForArraysAsCellArray_traj_self.jpeg)
-->
![Retrieving of arrays, 'cellarray' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForArraysAsCellArray_traj_self.jpeg)
<!---
![Retrieving of arrays, 'cellarray' mode](pictures/compareRetrieveForArraysAsCellArray_bar_self.jpeg)
-->
![Retrieving of arrays, 'cellarray' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForArraysAsCellArray_bar_self.jpeg)

Consider now the results for **DataReturnFormat** set to 'structure':

<!---
![Retrieving of arrays, 'structure' mode](pictures/compareRetrieveForArraysAsStruct_traj.jpeg)
-->
![Retrieving of arrays, 'structure' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForArraysAsStruct_traj.jpeg)
<!---
![Retrieving of arrays, 'structure' mode](pictures/compareRetrieveForArraysAsStruct_traj_cropped.jpeg)
-->
![Retrieving of arrays, 'structure' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForArraysAsStruct_traj.jpeg)
<!---
![Retrieving of arrays, 'structure' mode](pictures/compareRetrieveForArraysAsCellArray_bar.jpeg)
-->
![Retrieving of arrays, 'structure' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForArraysAsStruct_bar.jpeg)

The situation is the same as above with the single exception the total time for obtaining results
for 'structure' format is almost twice as long as the time spent for 'cellarray' format. This is because 'structure' format makes it more difficult 
to "pick out" Java arrays out of the values returned by **fetch**.
One can see that **Matlab Database Toolbox** digests 80000 tuples (75Mb in size), but fails to retrieve 90000 tuples (84Mb).

On average the execution time for [**exec**](http://pgmex.alliedtesting.com/#exec) along with
[**getf**](http://pgmex.alliedtesting.com/#getf) is approximately 81 (!) times longer of that time
for **exec** and **fetch** from **Matlab Database Toolbox** along with conversion of results, at least for those volumes
both function succeeded to retrieve. If again we consider only "pure" time, i.e. without transformation
of results into native Matlab formats, [**PgMex**](http://pgmex.alliedtesting.com) turns out to be about 5.5 times faster than
**Matlab Database Toolbox**:

<!---
![Retrieving of arrays, 'structure' mode](pictures/compareRetrieveForArraysAsStruct_traj_self.jpeg)
-->
![Retrieving of arrays, 'structure' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForArraysAsStruct_traj_self.jpeg)
<!---
![Retrieving of arrays, 'structure' mode](pictures/compareRetrieveForArraysAsStruct_bar_self.jpeg)
-->
![Retrieving of arrays, 'structure' mode]({{ site.baseurl }}/img/posts{{ page.path | remove: '_posts' | remove: '.md' }}/compareRetrieveForArraysAsStruct_bar_self.jpeg)

## Summary

So as we have seen, the functionality provided by **Matlab Database Toolbox** for JDBC connection to **PostgreSQL** has serious restrictions
both for importing and exporting data, especially for large datasets. The limitations of [**PgMex**](http://pgmex.alliedtesting.com)
are sufficiently higher and this library is free from those shortcomings related to
data types conversions (because it works with native Matlab formats).